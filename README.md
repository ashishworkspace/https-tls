## HTTPS-TLS
#### Initial folder structure:  https://github.com/ashishworkspace/https-tls/tree/initial-fold

### Folder strucaure</br> 
```
pki/
├── rootca
│   ├── certs
│   ├── index.txt
│   ├── index.txt.attr
│   ├── index.txt.old
│   ├── newcerts
│   │   └── 01.pem
│   ├── openssl.cnf
│   ├── private
│   │   └── root-ca.pem
│   ├── root-ca.crt
│   ├── serial
│   └── serial.old
├── server
│   ├── server.crt
│   ├── server.csr
│   └── server.pem
└── subca
    ├── certs
    ├── index.txt
    ├── index.txt.attr
    ├── index.txt.old
    ├── newcerts
    │   └── 01.pem
    ├── openssl.cnf
    ├── private
    │   └── subca.pem
    ├── serial
    ├── serial.old
    ├── subca.crt
    └── subca.csr
```

Run inside the `/pki/rootca` dir</br>

* command used to generate the private key and at rest it is being encrypted via AES256 algo</br>
> openssl genrsa -aes256 -out private/root-ca.pem 4096</br>

* command used to get the root certificate from private key | Since it is self signed certificate here the Issuer and Subject are same</br>
> openssl req -extensions v3_ca  -config openssl.cnf -new -sha256  -days 3650 -x509 -key private/root-ca.pem  -out root-ca.crt</br>

MUST BE ADDED TO `/pki/rootca/openssl.cnf` file</br>
```
[ v3_intermediate_ca ]

subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = critical,CA:true,pathlen:0


[ CA_default ]

dir		= /pki/rootca		# Where everything is kept
certs		= $dir/certs		# Where the issued certs are kept
crl_dir		= $dir/crl		# Where the issued crl are kept
database	= $dir/index.txt	# database index file.
#unique_subject	= no			# Set to 'no' to allow creation of
					# several certs with same subject.
new_certs_dir	= $dir/newcerts		# default place for new certs.

certificate	= $dir/root-ca.crt	# The CA certificate
serial		= $dir/serial 		# The current serial number
crlnumber	= $dir/crlnumber	# the current crl number
					# must be commented out to leave a V1 CRL
crl		= $dir/crl.pem 		# The current CRL
private_key	= $dir/private/root-ca.pem # The private key

```


* command to view the certificate bundled with public key</br>
> openssl x509 -in root-ca.crt -text</br>


Run inside the `/pki/subca` dir</br>

* command used to generate the private key and at rest it is being encrypted via AES256 algo</br>

> openssl genrsa -aes256 -out private/subca.pem 4096</br>

* here we are creating Certificate Signing request (.csr) to be signed by RootCA</br>
> openssl req -new -sha256 -key private/subca.pem -out subca.csr</br>


Run inside the `/pki/rootca` dir</br>

* command used to sign the subca certificate by rootca</br>
> openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 3650 -in /pki/subca/subca.csr -out /pki/subca/subca.crt</br>


MUST BE ADDED TO `/pki/subca/openssl.cnf` file</br>
```
[ CA_default ]

dir		= /pki/subca		# Where everything is kept
certs		= $dir/certs		# Where the issued certs are kept
crl_dir		= $dir/crl		# Where the issued crl are kept
database	= $dir/index.txt	# database index file.
#unique_subject	= no			# Set to 'no' to allow creation of
					# several certs with same subject.
new_certs_dir	= $dir/newcerts		# default place for new certs.

certificate	= $dir/subca.crt	# The CA certificate
serial		= $dir/serial 		# The current serial number
crlnumber	= $dir/crlnumber	# the current crl number
					# must be commented out to leave a V1 CRL
crl		= $dir/crl.pem 		# The current CRL
private_key	= $dir/private/subca.pem # The private key
```

* view the newly created certificate in subca dir | here the issuer will be rootca and subject will be subca</br>
> openssl x509 -in subca.crt -text</br>

* again we need to create the certificate for server in this case for webserver httpd</br>

Run inside the `/pki/server` dir</br>

* command used to generate the private key </br>
> openssl genrsa -out server.pem 4096</br>


* now we need to create the csr for the server to be signed by subca</br>
* Note: when we create the csr we need to put the host name as the domain name | In this case i have set "www.theboyz.in"</br>
> openssl req -new -days 365 -sha256 -key server.pem -out server.csr</br>

```
[root@f78c6c147420 server]# openssl req -new -days 365 -sha256 -key server.pem -out server.csr 
Ignoring -days without -x509; not generating a certificate
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:IN
State or Province Name (full name) []:BIHAR
Locality Name (eg, city) [Default City]:BHAGALPUR
Organization Name (eg, company) [Default Company Ltd]:THEBOYZ
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:www.theboyz.in
Email Address []:init.ashish@gmail.com
```


Run inside the `/pki/subca` dir</br>
* once that is done we can sign the csr of server with subca </br>
> openssl ca -config openssl.cnf -extensions v3_req -days 365 -in /pki/server/server.csr -out /pki/server/server.crt</br>

* we need to add the record to /etc/hosts same as we the name in server hostname</br>
> echo "127.0.0.1 www.theboyz.in" >> /etc/hosts</br>

* further the client need to have the certificate of the server inorder to do further communction </br>
* since in first case the client do not have the certificate of server it will fail to connect </br>
* so we need to add the certificate of server to client </br>
* in our case since the server and client is in same machine we can put the root-ca.crt in  /etc/pki/tls/certs/ca-bundle.crt  location</br>

> cat /pki/rootca/root-ca.crt  >> /etc/pki/tls/certs/ca-bundle.crt </br>

* also start the httpd webserver
> http &

* now we need to start the server to listen on port 443 i.e. https</br>

> openssl s_server -accept 443 -www -key server.pem -cert server.crt -CAfile /pki/subca/subca.crt </br>

* open a new terminal and do curl</br>

> curl https://www.theboyz.in:443</br>
