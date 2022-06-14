## Generating certificate/key pair for the private Docker registry
```
mkdir certs
```
Copy the below to the file __openssl.conf__

_Of course we need to update **Docker Server IP** with the IP address of the server where you will be running docker registry_
```
[ req ]
distinguished_name = req_distinguished_name
x509_extensions     = req_ext
default_md         = sha256
prompt             = no
encrypt_key        = no

[ req_distinguished_name ]
countryName            = "EG"
localityName           = "Cairo"
organizationName       = "Rania"
organizationalUnitName = "Docker"
commonName             = "<Docker Server IP>"
emailAddress           = "rania.fahmy22@gmail.com"

[ req_ext ]
subjectAltName = @alt_names

[alt_names]
DNS = "<Docker Server IP>"
```
Generate the certificate and private key
```
openssl req \
 -x509 -newkey rsa:4096 -days 365 -config openssl.conf \
 -keyout certs/domain.key -out certs/domain.crt
```
We can verify the certificate with the command
```
openssl x509 -text -noout -in certs/domain.crt
```
