# pki-notes

![Diagram from Wiki](https://upload.wikimedia.org/wikipedia/commons/3/34/Public-Key-Infrastructure.svg)


## Keywords
* CA = Certificate Authority
* VA = Validation Authority
* RA = Registration Authority

* Private Key
* Public Certificate
* Certificate signing request (CSR)
* Certificate revocation list (CRL)

## Certifiate types

* Domain Validation (DV) is the lowest level of validation, and verifies that whoever requests the certificate controls the domain that it protects.
* Organization Validation (OV) verifies the identity of the organization (e.g. a business, nonprofit, or government organization) of the certificate applicant.
* Individual Validation (IV) verifies the identity of the individual person requesting the certificate.
* Extended Validation (EV), like OV, verifies the identity of an organization. However, EV represents a higher standard of trust than OV and requires more rigorous validation checks to meet the standard of the CA/Browser Forum’s Extend.


## Instruction for self-signed CA
```
mkdir /root/ca
cd /root/ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
```
# Update openssl.conf for new directories
# Create root CA private key
```
cd /root/ca
openssl genrsa -aes256 -out private/ca.key.pem 4096

# Enter pass phrase for ca.key.pem: secretpassword
# Verifying - Enter pass phrase for ca.key.pem: secretpassword

chmod 400 private/ca.key.pem
```
# Create root CA certificate
```
cd /root/ca
openssl req -config openssl.cnf \
      -key private/ca.key.pem \
      -new -x509 -days 7300 -sha256 -extensions v3_ca \
      -out certs/ca.cert.pem
      
```

# Create client private key, CSR and certificate and sign with CA

```
# following can run multiple time based on different domain and requirements
#Create Client Cert private key
openssl genrsa -des3 -out example.key 2048

openssl genrsa -des3 -out example.key 2048

#Remove passphrase
openssl rsa -in example.key -out example.key.insecure


#Create CSR
openssl req -new -key example.key.insecure -out example.csr -config openssl.cnf

#Create Certificate using CA and CSR
openssl ca -in example.csr -config openssl.cnf

# convert to p12 file
openssl pkcs12 -export -inkey cert_key_pem.txt -in cert_key_pem.txt -out cert_key.p12
```
# Create tls secret file for k8s
```
cat example.pem | base64 -w 0 > tls.crt
cat example.key | base64 -w 0 > tls.key

TLSCERT=$(cat $REGISTRY_INGRESS_HOSTNAME.crt | base64 -w 0)
TLSKEY=$(cat $REGISTRY_INGRESS_HOSTNAME.key | base64 -w 0)

SEDEXPRS=(
  "-e" "s/{{tlscert}}/$TLSCERT/g"
  "-e" "s/{{tlskey}}/$TLSKEY/g"
)

cat <<EOF | sed ${SEDEXPRS[*]} | kubectl replace -f -
apiVersion: v1
kind: Secret
metadata:
  name: registry-tls-data
type: Opaque
data:
  tls.crt: {{tlscert}}
  tls.key: {{tlskey}}
EOF  

```
