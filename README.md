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

## Create self-signed certificate
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/selfsigned.key -out /etc/ssl/certs/selfsigned.crt
```

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
# Prepare for intermediate CA
```
mkdir /root/ca/intermediate
cd /root/ca/intermediate
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
# Create CRL number to track CRL
echo 1000 > /root/ca/intermediate/crlnumber
```
# Create and update openssl.conf for intermediate CA with above directory
```
[ CA_default ]
dir             = /root/ca/intermediate
private_key     = $dir/private/intermediate.key.pem
certificate     = $dir/certs/intermediate.cert.pem
crl             = $dir/crl/intermediate.crl.pem
policy          = policy_loose
```
# Create intermediate CA private key
```
cd /root/ca
openssl genrsa -aes256 \
      -out intermediate/private/intermediate.key.pem 4096

# Enter pass phrase for intermediate.key.pem: secretpassword
# Verifying - Enter pass phrase for intermediate.key.pem: secretpassword

chmod 400 intermediate/private/intermediate.key.pem
```
# Create CSR for intermediate
```
# cd /root/ca
# openssl req -config intermediate/openssl.cnf -new -sha256 \
      -key intermediate/private/intermediate.key.pem \
      -out intermediate/csr/intermediate.csr.pem
```

# Create intermediate CA certificate and sign with root CA
```
cd /root/ca
openssl ca -config openssl.cnf -extensions v3_intermediate_ca \
      -days 3650 -notext -md sha256 \
      -in intermediate/csr/intermediate.csr.pem \
      -out intermediate/certs/intermediate.cert.pem

chmod 444 intermediate/certs/intermediate.cert.pem
```

# Verify intermediate CA cert with root CA
```
openssl verify -CAfile certs/ca.cert.pem \
      intermediate/certs/intermediate.cert.pem
```

# Create CA cert chain
```
cat intermediate/certs/intermediate.cert.pem \
      certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```

# Create client private key, CSR and certificate and sign with CA

```
# following can run multiple time based on different domain and requirements
#Create Client Cert private key
openssl genrsa -des3 -out example.key 2048

#Remove passphrase
openssl rsa -in example.key -out example.key.insecure


#Create CSR
# openssl req -new -key example.key.insecure -out example.csr -config openssl.cnf
openssl req -config intermediate/openssl.cnf \
      -key example.key \
      -new -sha256 -out example.csr


#Create Certificate using CA and CSR
# openssl ca -in example.csr -config openssl.cnf
openssl ca -config intermediate/openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in example.csr \
      -out example.cert

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
