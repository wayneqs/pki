# Creating Client Certificates using openssl
## How to create a client certificate for use in Azure APIM without creating a CA

If you don't need to validate your certificates other than checking basic certificate meta-data then you can use the simple approach.

1. Create private key for the client certificate

```
openssl genrsa -out client.key 4096
```

2. Create Certificate Signing Request for client certificate

```
openssl req -new -key client.key -out client.csr -config openssl.cnf
```

3. Self-sign the CSR

```
openssl x509 -req -days 365 -in client.csr -signkey client.key -out client.crt -extfile ext.cnf
```
This creates a client certificate called _client.crt_.

4. Create the pfx file that can be uploaded to KeyVault

_client.crt_ needs to be converted to PKCS12 format in order to be uploaded to KeyVault by running the following command.

```
openssl pkcs12 -export -in client.crt -inkey client.key -out client.pfx
```
This generates a file called _client.pfx_ which can be uploaded.

## How to create a client certificate for use in Azure APIM when you must create a CA.

__NOTE: the CA that is stored in this repository is for informational and instructional purposes only. It should be considered as _compromised_ because the private keys of both the Root CA and Intermediate CA are stored in the repository. It must not be used for any non-development activities.__

If you need to validate certificates then you will need to upload the root or CA certificate for your self-signed certificates to APIM and then reference your client certificates from KeyVault.

If you need to do this then this is the set of instructions for you.

You will need to create a Root CA and an Intermediate CA. The Intermediate CA is not critical unless you are planning to use self-signed certs with your production systems. 

If you are planning to use self-signed certificates in your production systems then it is essential that the Root CA's private key is protected by being stored in an air-gapped system. 

If a Root CA is compromised then all certificates signed by it are compromised and can no longer be trusted. In general the best way to protect the Root CA is to use it only for creating Intermediate CAs and to never use it for routine certificate signing.

In the example that follows and for all the artifacts in this repository all passwords are set to 'secret', other than in the password file that is encrypted in the second step.

1. Create Root CA directory structure

```
[~]# mkdir -p root/tls/{certs,private}
[~]# cd root/tls
[~/root/tls]# echo 01 > serial
[~/root/tls]# touch index.txt
```

2.  Create an encrypted password file

```
[~/root/tls]# echo pa55w0rd > mypass
[~/root/tls]# openssl enc -aes256 -pbkdf2 -salt -in mypass -out mypass.enc
[~/root/tls]# rm mypass
```
Check this has worked by decrypting the file...
```
[~/root/tls]# openssl enc -aes256 -pbkdf2 -salt -d -in mypass.enc
```

3. Generate Root CA Private Key

```
[~/root/tls]# openssl genrsa -des3 -passout file:mypass.enc -out private/cakey.pem 4096
```

4. Create Root CA Certificate

Ensure that the Common Name of the root CA certificate is not the same as the server or client certificates or a naming collision issue will occur which will cause problems later.

```
[~/root/tls]# openssl req -new -x509 -days 3650 -passin file:mypass.enc -config openssl.cnf -extensions v3_ca -key private/cakey.pem -out certs/cacert.pem
[~/root/tls]# openssl x509 -in certs/cacert.pem -out certs/cacert.pem -outform PEM
```

Verify the certificate by running the following command.

```
openssl x509 -noout -text -in certs/cacert.pem
```

You can inspect various certificate meta-data such as dates, issuer, subject, etc.

5. Create Intermediate CA directory structure. Optionally you can create an encrypted password file as described already. We are reusing the same one here but you shouldn't if doing this for real.

```
[~/root/tls]# mkdir -p intermediate/{certs,csr,private}
[~/root/tls]# cd intermediate
[~/root/tls/intermediate]# echo 01 > serial
[~/root/tls/intermediate]# touch index.txt
[~/root/tls/intermediate]# echo 01 > crlnumber
```
Note: the crlnumber file is used to keep track of certificate revocation lists.

6. Generate Intermediate CA Key

```
[~/root/tls]# openssl genrsa -des3 -passout file:mypass.enc -out intermediate/private/intermediate.cakey.pem 4096
```

7. Create Intermediate CA Certificate Signing Request (CSR)

```
[~/root/tls]#  openssl req -new -sha256 -config intermediate/openssl.cnf -passin file:mypass.enc -key intermediate/private/intermediate.cakey.pem -out intermediate/csr/intermediate.csr.pem
```
8. Sign and Generate Intermediate CA Certificate

The expiry date must be less than the Root CA certificate.

```
[~/root/tls]# openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 2650 -notext -batch -passin file:mypass.enc -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cacert.pem
```

The index.txt file will now contain a line referring to the intermediate certificate.

Verify the trust chain of the intermediate certificate is intact with the Root CA certificate.

```
[~/root/tls]# openssl verify -CAfile certs/cacert.pem intermediate/certs/intermediate.cacert.pem
```

1. Create the Certificate Chain (aka Certificate Bundle)

The bundle is just all the certificates in the chain in PEM format concatenated into the same file.

```
[~/root/tls]# cat intermediate/certs/intermediate.cacert.pem certs/cacert.pem > intermediate/certs/ca-chain-bundle.cert.pem
```

10. Convert Root CA certificate to a format that can be uploaded to Azure APIM as a CA Certificate.

```
[~/root/tls]# openssl x509 -inform PEM -in certs/cacert.pem -outform DER -out certs/cacert.cer
```

11. Follow Steps 1, 2 and 4 of the first set of instructions to generate a client certificate. 

For step 3 (Signing the CSR) do the following instead.

```
openssl ca -days 365 -batch -create_serial -cert ./certs/ca-chain-bundle.cert.pem -keyfile ./private/intermediate.cakey.pem -in client.csr -out client.pem -extfile ext.cnf -config openssl.cnf -passin file:mypass.enc
```