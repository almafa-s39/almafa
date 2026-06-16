# PKI - OpenSSL

## Pre-settings

```bash
apt install openssl # Install by default on Debian based distros
mkdir /ca && cd /ca
```

## Self signed CA

### Generate files

```bash
# Generate ca key
openssl genrsa -out ./ca.key -aes256 4096 # Enter password on prompt

# Generate ca certificate
openssl req -x509 -new -nodes -key ./ca.key -out ./ca.crt \
    -sha256 -days 365 -subj "/C=HU/O=Ceg/CN=Kozonseges nev"
```

### Accept on CA on computers

```bash
# Copy CA certificate to its path
cp /ca/ca.crt /usr/local/share/ca-certificates

# Issue command to trust the certificate chain
/usr/sbin/update-ca-certificates
```

## Subordinate CA

### Create extensions file

`/ca/subca.v3.ext`

```bash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
```

### Generate files

```bash
# Generate key and cert signing request
openssl req -new -nodes -newkey rsa:4096 -keyout ./subca.key -out ./subca.csr -subj "/C=HU/O=Ceg/CN=Kozonseges alnev"

# Sign the signing request
openssl x509 -req -in ./subca.crt -extfile ./subca.v3.ext -out ./subca.crt -sha256 -days 365 -CACreateSerial -CA ./ca.crt -CAkey ./ca.key
```

> [!NOTE]
> You're ready now, to sign your certificates using the /ca/subca.crt file!
> For some services, create a chain from ca files (`cp /ca/subca.crt /ca/ca.crt > /ca/ca-chain.pem`), so you can create pem files for services, and give the full certificate chain to the services as well (like OpenVPN).

> [!WARNING]
> Don't forget to add this certificate to the Debians ca store, to trust the whole chain!

## End user certificates

### Extensions file

> [!NOTE]
> Only apply the extensions you configured, and want to use!

`certificate.v3.ext`

```bash
basicConstraints = CA:FALSE
authorityKeyIdentifier = keyid,issuer:always
keyUsage = digitalSignature,keyEncipherment,dataEncipherment,nonRepudiation,critical
extendedKeyUsage = clientAuth, serverAuth
authorityInfoAccess = caIssuers;URI:http://aia.domain.name/ca.crt;URI:http://aia.domain.name/subca.crt
CrlDistributionPoints = URI:http://crl.domain.name/ca.crl;URI:http://crl.domain.name/subca.crl
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = *.domain.name
IP.1 = 1.1.1.1
IP.2 = 2001:db8::1
```

### Generate the certificate

```bash
# Generate key and cert signing request
openssl req -new -nodes -newkey rsa:4096 -keyout ./certificate.key -out ./certificate.csr -subj "/C=HU/O=Ceg/CN=fqdn"

# Sign the signing request - you have to use CACreateSerial just for the first certificate
openssl x509 -req -in ./certificate.crt -extfile ./certificate.v3.ext -out ./certificate.crt -sha256 -days 365 -CACreateSerial -CA ./subca.crt -CAkey /subca.key
```

> [!NOTE]
> And you're done! You're ready to use the certificate!