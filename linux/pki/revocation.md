<!-- 
---
title: "OpenSSL revocation, AIA"
author: "Gergő Téringer"
---
 -->
# OpenSSL revocation, AIA

This document provides administrative procedures for configuring Authority Information Access (AIA) and Certificate Revocation Lists (CRL) using OpenSSL on Debian 13 (Trixie). It details modifying the `openssl.cnf` structure, generating revocation lists, revoking end-user certificates, and validating certificate statuses locally.

> [!NOTE]
> Certificate revocation ensures compromised or deprecated certificates are actively rejected by clients. The AIA extension assists clients in dynamically locating the issuing CA certificate if the full chain is not initially provided.

## 1. Core OpenSSL Configuration and AIA Extensions

To support revocation tracking and dynamic AIA lookup, the Root and Subordinate CAs must define their specific CRL directory structures and HTTP dissemination endpoints within the `/etc/ssl/openssl.cnf` file.

```Ini, TOML
# Define the CRL tracking database and pathing in the [ CA_default ] section
[ CA_default ]
# ...
dir = /storage/ca/
crl_dir = $dir/crl/
database = $crl_dir/index.txt
crl_number = $crl_dir/crl_number
crl = $crl_dir/ca.crl
basicConstraints = critical, CA:true
keyUsage = critical, keyCertSign, cRLSign

# Define Subordinate CA extensions including AIA and CRL endpoints in the [ v3_sub_ca ] section
[ v3_sub_ca ]
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer

authorityInfoAccess = caIssuers;URI:http://pki.company.com/ca.crt
crlDistributionPoints = URI:http://pki.company.com/ca.crl
# ...
```

**Command Breakdown & Explanation:**

- `database = $crl_dir/index.txt`: Specifies the local flat-file database OpenSSL utilizes to track all issued and revoked certificates.
- `authorityInfoAccess`: Embeds the HTTP URL into the certificate, directing clients where to download the issuing parent CA certificate.
- `crlDistributionPoints`: Embeds the HTTP URL into the certificate, indicating where clients can download the active Certificate Revocation List.

## 2. Issuing the Subordinate CA and Server Certificates

When generating the Subordinate CA, instruct OpenSSL to parse the newly defined `v3_sub_ca` block directly from the global configuration file.

```Bash
# Sign the Subordinate CSR applying the v3_sub_ca extensions from openssl.cnf
openssl x509 -req -in subca.csr -CA root.crt -CAkey root.key -CAcreateserial -out subca.crt -days 180 -sha256 -extfile /etc/ssl/openssl.cnf -extensions v3_sub_ca
```

Server and end-user certificates signed by this Subordinate CA require their own distinct AIA and CRL extensions pointing to the Subordinate CA's infrastructure. Add the following parameters to your server certificate extensions file before signing:

```Ini, TOML
# Append to the individual server/user certificate extensions file
authorityInfoAccess = caIssuers;URI:http://pki.company.com/subca.crt
crlDistributionPoints = URI:http://pki.company.com/subca.crl
```

**Command Breakdown & Explanation:**

- `-extfile /etc/ssl/openssl.cnf`: Directs OpenSSL to ingest the global configuration file to populate extension variables.
- `-extensions v3_sub_ca`: Explicitly targets the `[ v3_sub_ca ]` stanza to inject the designated `crlDistributionPoints` and `authorityInfoAccess` attributes into the resulting `.crt` file.

## 3. Certificate Revocation and CRL Management

The Subordinate CA must generate a Certificate Revocation List (CRL) for clients to verify trust states. Whenever a certificate is manually revoked, the CRL must be immediately regenerated and distributed to the web server endpoint.

```Bash
# Generate the initial Certificate Revocation List (CRL)
openssl ca -gencrl -cert /ca/subca.crt -keyfile /ca/subca.key -out /ca/subca.crl

# Revoke a specific end-user or server certificate
openssl ca -revoke /ca/user1.crt -keyfile /ca/subca.key -cert /ca/subca.crt

# Regenerate the CRL to publish the newly revoked serial number
openssl ca -gencrl -keyfile /ca/subca.key -cert /ca/subca.crt -out /ca/subca.crl
```

**Command Breakdown & Explanation:**

- `openssl ca -gencrl`: Parses the OpenSSL index database (`index.txt`) and compiles a cryptographically signed `.crl` file listing the serial numbers of all compromised or revoked certificates.
- `openssl ca -revoke`: Updates the `index.txt` database, altering the target certificate's internal status from 'Valid' (V) to 'Revoked' (R) and logging the revocation timestamp.

## 4. Verification and Troubleshooting

> [!NOTE]
> Validate the CRL contents and test local certificate revocation verification using the OpenSSL parsing engine on Debian 13.

### 4.1 Verify certificate revocation status locally

You can simulate a client validating a certificate against the active CRL by merging the trust chain and the active CRL into a temporary test bundle.

```Bash
# Combine the CA chain and the active CRL into a temporary validation bundle
cat /ca/chain.pem /ca/subca.crl > /tmp/test.pem

# Test the validity of a specific certificate against the bundled CRL
openssl verify -extend_crl -CAfile /tmp/test.pem -crl_check /ca/user.crt

# Clean up the temporary validation bundle
rm /tmp/test.pem
```

**Command Breakdown & Explanation:**

- `cat ... > /tmp/test.pem`: Merges the CA certificate chain and the active CRL into a single file readable by the verification engine.
- `-crl_check`: Enforces a strict check against the provided revocation list data. If the certificate's serial number is present, validation will immediately fail.
- `-extend_crl -CAfile /tmp/test.pem`: Instructs the verifier to load both the valid CA chain and the CRL appended within the `test.pem` block.

**What it checks and variables to look for:**

- **Valid Certificate Output**: Must output `/ca/user.crt: OK`
- **Revoked Certificate Output**: Must output `error 23 at 0 depth lookup:

<!-- Created by: Gergő Téringer, 2026 -->