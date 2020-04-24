# CA + yubikey guide 

## About CA and SUBCA
 * CA key is stored in keepass
 * CA certyfiicate is stored `certs/cacert.pem` and in keepass
 * SUBCA key is stored on yubikey
 * SUBCA certyficate is stored `newcerts/1000.pem` on yubikey and in keepass
 * List of signed certs `index.txt`

-----

## Prerequisites
* DEBIAN: openssl + opensc + pkcs11 engine for openssl
* ARCH: `pacman -S openssl libp11 pkcs11-helper` + `yay -S yubikey-piv-manager`
* prepare directory for CA - in this tutorial `/dev/shm/ca/` later moved to disk location.
```bash
mkdir /dev/shm/ca && cd /dev/shm/ca
mkdir certs crl keys requests
touch index.txt
echo '1000' > serial
```
* put `openssl.conf` from this repository to `/dev/shm/ca/`
* in openssl.conf configure:
    * MODULE_PATH = variable accordding to your system
    * [ req_distinguished_name ] section

-----
## CA setup
### 1. Generate CA master key and self signed certyficate
  *This will be a master CA what is stored in keepass and used only to generate internediate keypairs for operational pursposes. Never ever use it for signing requests. Never unpack it form keepass except to generate new intermediate CA. Always use **RAM memory** or encrypted disk when you download it form keepass* 
```
openssl req -config openssl.conf -new -newkey rsa:4096 -x509 -sha512 -extensions v3_ca -keyout keys/cakey.pem -out certs/cacert.pem -days 365000 -set_serial 0
```
Now you have:
 * `keys/cakey.pem` - secret key of your master CA
 * `carts/cacert.pem` - secret cert of your master CA

Please copy then and password for cakey.pem to keepass. Do not remove them yet. We will need those later.

-----

## Intermediate CA setup
### 1. Generate key and certyficate signing request
 * Plug in your configured yubikey (pin, mgmt key - you should have them set up and configured)
 * Start pivman
 * Generate in slot 9c new RSA 2048 key
 * Generate new signing request from key stored in 9c slot and save it under `/dev/shm/ca/requests/intermediate_req.pem`
 * Hint: use / to separate different DN fields ex. `/C=PL/ST=malopolska/L=Cracow/O=pyton/CN=pyton_ca/emailAddress=ca@myserver/`
 * Hint2: Check what fields in DN are need to be the same with master CA - openssl.conf

### 2. Sign request for intermediate CA using master CA
```
openssl ca -config openssl.conf -extensions v3_intermediate_ca -days 3650 -notext -keyfile keys/cakey.pem -cert certs/cacert.pem -in requests/intermediate_req.pem

ln -s 1000.pem certs/intermediate_cert.pem
```
### 3. Upload certyficate to yubikey
After step 2 you should have your signed certyficate in `/dev/shm/certs/1000.pem`. Please open pivman and upload your signed cert to slot 9c. Additionally save it in keepass.


-----

## Cleanup
### 1. Now you are ready to made some cleanup. Please check files listed:
 * `keys/cakey.pem` - secret key of your master CA - **NEED TO BE IN KEEPASS**
 * `certs/cacert.pem` - public cert of your master CA - **NEED TO BE IN KEPASS**
 * `certs/1000.pem` - your intermediate public cert signed by master CA - **NEED TO BE IN KEEPASS and on YUBIKEY**
 * `certs/intermediate_cert.pem` - symbolic link to `certs/1000.pem` - **NEED TO BE ON DISK ONLY**
 * `requests/intermediate_req.pem` - **NEED TO BE ON DISK ONLY**
 * `serial` - should contain number 1001 (if you have only 1 yubikey)
 * `index.txt` - should contain only one entry (if you have 1 yubikey)

### 2. Now we remove private key and other unnecessary informations and move our CA directory to more persistent location:

```
rm keys/cakey.pem requests/intermediate_req.pem 
mv /dev/shm/ca ~/ca
```

-----


## Generate new key and certyficate request for openvpn server (as example)

### 1. All at once - recommended
```
cd ~/ca

openssl req -config openssl.conf -new -newkey rsa:4096 -extensions server_cert -keyout keys/openvpn_key.pem -out requests/openvpn_req.pem
```

### 2. Alternative method key and csr independent
```
openssl genrsa -out keys/openvpn_key.pem 4096

openssl req -sha256 -new -config openssl.conf -key keys/openvpn_key.pem -nodes -out requests/openvpn_req.pem
```

-----

## Signing openvpn request using SUBCA on yubikey
### 1. Signing using CA (recommended method):
```
openssl ca -config openssl.conf -in requests/openvpn_req.pem -policy clug_subca_policy -extensions server_cert -engine pkcs11 -keyfile 02 -cert certs/intermediate_cert.pem -keyform engine -updatedb
```
### 2. Alternative method with openssl CLI
#### 2.1 PKCS integration with OPENSSL -  DEBIAN:
```
user@server # > openssl

engine dynamic -pre SO_PATH:/usr/lib/x86_64-linux-gnu/engines-1.1/libpkcs11.so -pre ID:pkcs11 -pre NO_VCHECK:1 -pre LIST_ADD:1 -pre LOAD -pre MODULE_PATH:/usr/lib/x86_64-linux-gnu/pkcs11/opensc-pkcs11.so -pre VERBOSE
```

#### 2.2 PKCS integration with OPENSSL -  ARCH:
```
user@server # > openssl

engine dynamic -pre SO_PATH:/usr/lib/engines-1.1/libpkcs11.so -pre ID:pkcs11 -pre NO_VCHECK:1 -pre LIST_ADD:1 -pre LOAD -pre MODULE_PATH:/usr/lib/opensc-pkcs11.so -pre VERBOSE
```

#### 2.3 Signing using x509 module:
```
x509 -engine pkcs11 -CAserial=serial -CAkeyform engine -CAkey id_02 -sha256 -CA certs/intermediate_cert.pem -req -in requests/openvpn_req.pem -out certs/openvpn_cert.pem
```

## Revoke certyficate
 * First you need to locate particular certyficate file. List file `index.txt`
 * Third column is a HEX number of signed cert. If your certyficate to revoke has for example number 07, file with that certyficate is localted in certs/1001.pem
 * To revoke that cert please issue command:
  `openssl ca -config openssl.conf -revoke certs/1001.pem -engine pkcs11 -keyfile id_02 -keyform engine -updatedb -cert certs/intermediate_cert.pem`
