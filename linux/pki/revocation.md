# OpenSSL revocation, AIA

## Pre configurations

```shell
[ CA_default ]
# ...
dir = /storage/ca/
crl_dir = $dir/crl/
database = $crl_dir/index.txt
crl_number = $crl_dir/crl_number
crl = $crl_dir/ca.crl
basicConstraints = critical, CA:true
keyUsage = critical, keyCertSign, cRLSign

# ...
[ v3_sub_ca ]
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer

authorityInfoAccess = caIssuers;URI:http://pki.company.com/ca.crt
crlDistributionPoints = URI:http://pki.company.com/ca.crl
# ...
```

To generate the subca using the following extensions that you put in the `openssl.cnf` file, 

```shell
openssl x509 -req -in subca.csr -CA root.crt -CAkey root.key -CAcreateserial -out subca.crt -days 180 -sha256 -extfile /etc/ssl/openssl.cnf -extensions v3_sub_ca
```

## Server certificate extension:

```shell
authorityInfoAccess = caIssuers;URI:http://pki.company.com/subca.crt
crlDistributionPoints = URI:http://pki.company.com/subca.crl
```

## Generate revocation file
```shell
openssl ca gencrl -cert /ca/subca.crt -keyfile /ca/subca.key -out /ca/subca.crl
```


## Revoke certificates

```shell
openssl ca -revoke /ca/user1.crt -keyfile /ca/subca.key -cert /ca/subca.crt
openssl ca -gencrl -keyfile /ca/subca.key -cert /ca/subca.crt -out /ca/subca.crl
```

## Test revocation on a certificate

```shell
cp /ca/chain.pem /ca/subca.crl > /tmp/test.pem
openssl verify -extend_crl -CAfile /tmp/test.pem -crl_check /ca/user.crt
rm /tmp/test.pem
```