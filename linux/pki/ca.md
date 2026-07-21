<!-- 
---
title: "PKI - OpenSSL"
author: "Gergő Téringer"
---
 -->
# PKI - OpenSSL

This document provides administrative procedures for establishing a Public Key Infrastructure (PKI) using OpenSSL on Debian 13 (Trixie). It covers the creation of a self-signed Root Certificate Authority (CA), generating a Subordinate CA, configuring v3 certificate extensions, issuing end-user certificates, and trusting the CA chain system-wide.

> [!NOTE]
> OpenSSL is typically installed by default on Debian-based distributions. Ensure you operate within a dedicated, secure directory (such as `/ca`) with restricted permissions when generating cryptographic private keys.

## 1. Root Certificate Authority (CA) Generation

The Root CA is the ultimate cryptographic trust anchor. You must generate a highly secure private key and self-sign the initial Root CA certificate.

```Bash
# Ensure OpenSSL is installed and create a dedicated CA directory
apt install openssl
mkdir -p /ca
cd /ca

# Generate an AES-256 encrypted 4096-bit RSA private key for the Root CA
openssl genrsa -out ./ca.key -aes256 4096

# Generate the self-signed Root CA certificate (valid for 365 days)
openssl req -x509 -new -nodes -key ./ca.key -out ./ca.crt \
    -sha256 -days 365 -subj "/C=HU/O=Ceg/CN=Kozonseges nev"
```

**Command Breakdown & Explanation:**

- `apt install openssl`: Installs the core OpenSSL cryptographic toolkit.
- `openssl genrsa ... -aes256`: Creates an RSA private key protected by an AES-256 passphrase (which will be prompted interactively during execution).
- `openssl req -x509`: Instructs OpenSSL to output a self-signed certificate rather than a standard Certificate Signing Request (CSR).

## 2. Subordinate CA Configuration

Best practices dictate keeping the Root CA securely offline and utilizing a Subordinate CA to issue daily end-user certificates. The Subordinate CA requires a specific v3 extensions file to securely assert its intermediate authority.

```Bash
# Create the v3 extensions file for the Subordinate CA
cat << 'EOF' > /ca/subca.v3.ext
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
EOF

# Generate the private key and CSR for the Subordinate CA
openssl req -new -nodes -newkey rsa:4096 -keyout ./subca.key -out ./subca.csr -subj "/C=HU/O=Ceg/CN=Kozonseges alnev"

# Sign the Subordinate CSR using the Root CA
openssl x509 -req -in ./subca.csr -extfile /ca/subca.v3.ext -out ./subca.crt -sha256 -days 365 -CACreateSerial -CA ./ca.crt -CAkey ./ca.key
```

**Command Breakdown & Explanation:**

- `basicConstraints = critical, CA:true, pathlen:0`: Strictly defines this certificate as a Certificate Authority, but restricts it from issuing further Subordinate CAs downstream (`pathlen:0`).
- `openssl x509 -req ... -CACreateSerial`: Signs the CSR and generates a tracking serial file (`.srl`) so the issuing CA can track spawned certificates.

> [!NOTE]
> You are now ready to sign certificates using `/ca/subca.crt`. For services requiring the full chain of trust (such as OpenVPN or web servers), bundle the Subordinate and Root CA certificates into a single PEM file: `cat /ca/subca.crt /ca/ca.crt > /ca/ca-chain.pem`.

## 3. Trusting the CA Chain on Debian

To prevent local services and applications (like `curl` or local web browsers) from throwing SSL warnings, you must import your custom Root and Subordinate CA certificates into the Debian system trust store.

> [!WARNING]
> Do not forget to add both the Root CA and the Subordinate CA to the system store to ensure the entire trust chain resolves properly without gaps!

```Bash
# Copy the CA certificates to the local trust store directory
cp /ca/ca.crt /usr/local/share/ca-certificates/
cp /ca/subca.crt /usr/local/share/ca-certificates/

# Update the system-wide CA bundle
/usr/sbin/update-ca-certificates
```

**Command Breakdown & Explanation:**

- `cp ... /usr/local/share/ca-certificates/`: Places custom `.crt` certificates in the directory specifically designated by Debian for local administrator additions.
- `update-ca-certificates`: Scans the directory and compiles the keys into the system's global `/etc/ssl/certs/ca-certificates.crt` binary store.

## 4. End-User Certificate Generation

End-user certificates require their own v3 extensions file. This configuration file dictates usage constraints, Subject Alternative Names (SANs), and revocation endpoints.

> [!TIP]
> Only apply the extensions you actively intend to use. Modify the `[ alt_names ]` block to match the exact DNS records and IP addresses that clients will use to access the host.

```Bash
# Create the v3 extensions file for the end-user certificate
cat << 'EOF' > /ca/certificate.v3.ext
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
EOF

# Generate the private key and CSR for the end-user
openssl req -new -nodes -newkey rsa:4096 -keyout ./certificate.key -out ./certificate.csr -subj "/C=HU/O=Ceg/CN=fqdn"

# Sign the end-user CSR using the Subordinate CA
openssl x509 -req -in ./certificate.csr -extfile /ca/certificate.v3.ext -out ./certificate.crt -sha256 -days 365 -CACreateSerial -CA ./subca.crt -CAkey ./subca.key
```

**Command Breakdown & Explanation:**

- `basicConstraints = CA:FALSE`: Explicitly prevents this certificate from being used to sign other downstream certificates.
- `subjectAltName = @alt_names`: Binds the certificate to multiple DNS names and IP addresses (strictly required by modern browsers, as Common Name matching is deprecated).
- `openssl x509 -req`: Signs the endpoint CSR utilizing the intermediate Subordinate CA (`-CA ./subca.crt` and `-CAkey ./subca.key`).

## 5. Verification and Troubleshooting

> [!NOTE]
> Validate the certificate chain logic, parsed extensions, and system trust store imports on Debian 13 using standard OpenSSL diagnostic commands.

### 5.1 Verify certificate text and parsed extensions

**Command:** `openssl x509 -in /ca/certificate.crt -text -noout`

**What it checks and variables to look for:**

- **Issuer**: Must match the `Subject` name of your Subordinate CA
- **Subject**: Must match the end-user `/CN=fqdn`
- **X509v3 Subject Alternative Name**: Must accurately list the DNS entries and IPs defined in your `.ext` file block

### 5.2 Verify the full CA chain of trust

**Command:** `openssl verify -CAfile /ca/ca-chain.pem /ca/certificate.crt`

**What it checks and variables to look for:**

- **Output validation**: Must output exactly `/ca/certificate.crt: OK` without reporting missing local issuer errors

<!-- Created by: Gergő Téringer, 2026 -->