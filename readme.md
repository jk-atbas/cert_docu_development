# Zertifikatsaustellung mit OpenSSL

## Root-Cert

Konfiguration in `configs/openssl_root.cnf`:

```cnf
[ req ]
default_bits       = 4096
distinguished_name = dn
x509_extensions    = v3_ca
prompt             = no

[ dn ]
CN = atbas-Root-CA

[ v3_ca ]
basicConstraints = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
```

Mit Cli das Root-Cert erstellen

```bash
openssl req -x509 -newkey rsa:4096 -days 3650 -nodes \
    -keyout private/rootCA.key \
    -out private/rootCA.crt \
    -config ./configs/openssl_root.cnf
```

Erstellt ein neues Root-Cert mit 4096-bit RSA Private-Key welches 10 Jahre gültig ist

Ggf. als pfx-Container exportieren

```bash
openssl pkcs12 -export -out private/rootCA.pfx \
    -inkey private/rootCA.key \
    -in private/rootCA.crt
```

Das bundelt Cert + Private Key in einen passwortgeschützten PKCS12 Container, welcher als ein Paket importiert werden kann (Nur empfohlen für Ersteller/CA; **NICHT** für jeden Entwickler)

## Serverauth und RDP-Cert für Web-VM

Konfiguration in `configs/openssl_server.cnf`:

```cnf
[ req ]
default_bits       = 2048
distinguished_name = dn
req_extensions     = v3_req
prompt             = no

[ dn ]
CN = web.atbas.de

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, 1.3.6.1.4.1.311.54.1.2
subjectAltName = @alt_names
crlDistributionPoints = URI:http://SRV060.dd.atbas.de:1337/atbas-root-ca.crl

[ alt_names ]
DNS.1 = web.atbas.de
```

> CRL/OCSP URL kann sich später auch ändern

> Im Moment (März 2026) läuft auf srv060 ein Mini-Http-Server der die CRL über http bereitstellt

### CSR (Certificate Signing Request) für Server-Cert erstellen

```bash
openssl req -new -nodes -out server.csr -keyout server.key \
    -config ./openssl_server.cnf
```

### Root-Cert signiert CSR und stellt Server-Cert aus

```bash
openssl x509 -req -in server.csr \
    -CA private/rootCA.crt \
    -CAkey private/rootCA.key \
    -CAcreateserial \
    -out server.crt \
    -days 825 -extensions v3_req \
    -extfile ./configs/openssl_server.cnf
```

Stellt ein ~2 Jahre gültiges Server-Cert aus

Ggf wieder als pfx Container exportieren

```bash
 openssl pkcs12 -export -out server.pfx \
    -inkey server.key \
    -in server.crt \
    -certfile private/rootCA.crt
```

Das exportiert einen passwortgeschützten PKCS12 Container mit Root-Cert.

## CRL Erstellung

Für unseren einfachen Fall reicht zu Beginn eine leere CRL. Die Dateien `index.txt`, `crlnumber` und `serial` werden für das Erstellen von CRLs benötigt.

Konfiguration in `configs/openssl_crl.cnf`:

```cnf
[ ca ]
default_ca = CA_default

[ CA_default ]
dir             = .
database        = $dir/index.txt
serial          = $dir/serial
crlnumber       = $dir/crlnumber

certificate     = $dir/private/rootCA.crt
private_key     = $dir/private/rootCA.key

default_md      = sha256
default_crl_days = 36500
crl             = $dir/atbas-root-crl.crl
```

Gültigkeit der CRL ist 100 Jahre

### Erzeugen der CRL

```bash
openssl ca -gencrl -config configs/openssl_crl.cnf -out atbas-root-ca.crl
```

### Deployment auf dem CRL Bereitstellungsort

Die [Url](#serverauth-und-rdp-cert-für-web-vm) muss logischerweise existieren. Von dort ruft sich der prüfende Client sich die CRL ab und prüft, ob das Server-Cert revoked wurde.

> Das wird bei uns in der Entwicklung im Normalfall so gut wie nie passieren

Im Moment wird auf `SRV060.dd.atbas.de` auf Port `1337` ein einfacher interner Python-Http-Webserver bereitgestellt, der das Verzeichnis, in dem sich die CRL befindet, abbildet.

> Bei Änderung dieser URL muss ein neues Server-Cert ausgestellt werden!

#### Definition Web-Server auf SRV060

Mit `nssm` erstellter Windows-Dienst `python_crl_deployment`

Installieren des Dienstes (in Admin Powershell)

```pwsh
nssm install python_crl_deployment "python" "-m http.server 1337 --bind 0.0.0.0"
```

Setzen des AppDirectory

```pwsh
nssm set python_crl_deployment AppDirectory "D:\crl"
```

> Dort liegt unter `D:\crl` die CRL

Als automatischer Dienst konfigurieren

```pwsh
nssm set python_crl_deployment Start SERVICE_AUTO_START
```

Dienst starten

```pwsh
Start-Service python_crl_deployment
```

## Einrichtung Server-Cert auf Web-VM

### Import des Server-Cert

Graphisch:

Starten der Zertifikat-Konsole über `Win-Taste + R` und `certlm.msc`

Importieren des `server.pfx` in `Eigene Zertifikate/My` mit gewähltem Passwort. Kopieren des neu importieren Server-Certs (mit Rechtsklick kopieren) nach `RemoteDesktop` (Rechtsklick einfügen)

> Beim Importieren nicht die Option `Privaten Schlüssel mit virtualisierungsbasierter Sicherheit schützen (nicht exportierbar)` auswählen!

### Für RDP verwenden

> Fingerabdruck des Server-Cert merken!

Fingerabdruck des Cert für RDP-TCP hinterlegen

```pwsh
$path = (Get-WmiObject -Class "Win32_TSGeneralSetting" `
    -Namespace root\cimv2\terminalservices `
    -Filter "TerminalName='RDP-tcp'").__path 

Set-WmiInstance -Path $path -Argument @{SSLCertificateSHA1Hash="Fingerabdruck hier eintragen"}
```

TermService neustarten (RDP-Service) 

```pwsh
# Als admin
Restart-Service TermService -Force
```

> Damit werden alle aktiven RDP-Sitzungen getrennt

> **Stand jetzt muss man beim Verbinden auf die Web-VM mit der Firmen-VPN verbunden sein!**

Nach RDP-Verbindungsaufbau kann die VPN wieder getrennt werden
