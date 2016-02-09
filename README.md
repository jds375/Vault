# Vault

###Introduction###
Security is a critical issue in modern software that is becoming ever more important. This stems from a combination of increasingly valuable secrets being stored in software, a lack of focus on security in Computer Science curriculum, and a lack of well-documented open source, free-to-use security solutions. 

Luckily, Hashicorp has released an open source solution for managing 'secrets' (passwords, api keys, certificates, credentials, etc.) called Vault. While it is pretty well documented, it is not documented well enough that a security 'noob' like myself could pick it up and after a bit of work have production-ready security. Below is my guide to setting it up and using it to manage account credentials for a collection of users on a Mac/Linux computer. As you're reading if you notice any errors or security flaws, please let me know so I can improve this guide and my understanding!

Before following this guide, make sure you download and install vault according to their guide and you have a new folder ready to go with the 'vault' executable in it.

###Configuring the Server###
The first step is to configure the Vault server to run securely. The first step is settuing up Transport Layer Security (TLS). TLS basically ensures that when a client and a server are communicating all of the information shared is private and cannot be tampered with. This requires us to generate a TLS certificate. In reality this will have to be done using a trusted Certificate Authority (CA), but for now we'll generate and use our own. There are several reasons for doing this. The two most important are probably that these certificates can cost a fair bit of money from a trusted site and that for local sandboxed development you will likely want to use a self-generated certificate anyway (to save money, time, and allow for scability for multiple developers). 

We begin by generating a root certificate. This can be done using the command: 
```bash
$ openssl req -newkey rsa:2048 -days 3650 -x509 -nodes -out root.cer
```
This will prompt you to fill out a series of questions for your certificate. Fill them out properly. Next, we will generate a certificate and key. Fill out all of the information, but leave the challenge password and optional company name (last two prompts) blank since few CA's support this. Simply press enter to skip these fields. This can be done with:
```bash
$ openssl req -newkey rsa:1024 -nodes -out vault.csr -keyout vault.key
```
Next we must create a serialfile and a certindex. We use the following two commands:
```bash
$ echo 000a > serialfile
$ touch certindex
```
In order to legitimize this certificate we will create our own Certificate Authority as mentioned above. We first will need to create the configuration file for it. We can do this using the root certificate and its corresponding key generated in the first step. Create a file as shown below with the proper paths filled in where so indicated called vault-ca.conf:
```
[ ca ]
default_ca = myca

[ myca ]
new_certs_dir = /tmp
unique_subject = no
certificate = <full path to root.cer generated in step 1>
database = <full path to certindex generated in step 3>
private_key = <full path to privkey.pem generated in step 1>
serial = <full path to serialfile generated in step 3>
default_days = 365
default_md = sha1
policy = myca_policy
x509_extensions = myca_extensions
copy_extensions = copy

[ myca_policy ]
commonName = supplied
stateOrProvinceName = supplied
countryName = supplied
emailAddress = optional
organizationName = supplied
organizationalUnitName = optional

[ myca_extensions ]
basicConstraints = CA:false
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always
subjectAltName = IP:127.0.0.1
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = serverAuth
```
Now we can use this as our CA for creating a signed certificate. We can do this using this command:
```bash
$ openssl ca -batch -config vault-ca.conf -notext -in vault.csr -out vault.crt
```
We will now copy our key and cert to a more standard location. In their residing directory execute the following commands:
```bash
$ sudo cp vault.crt /var/lib/vault/ssl/vault.crt
$ sudo cp vault.key /var/lib/vault/ssl/vault.key
```
We now must create the configuration for the Vault server itself. For now we will use a file-backed Vault. We then will provide the certificate and key we generated. The file should be called vault.conf. We provide an example here:
```
backend "file" {
    path = "<desired path for the backend file>"
}

listener "tcp" {
    address = "127.0.0.1:8200"
    tls_cert_file = "/var/lib/vault/ssl/vault.crt"
    tls_key_file = "/var/lib/vault/ssl/vault.key"
}
```
We are now ready to start our server. We can do this with the following command:
```bash
$ ./vault server -config=vault.conf
```
Finally, we can initialize the server using the following command:
```bash
$ ./vault init -ca-cert=root.cer
```
Make sure to save and/or write down the 5 keys and the root token. We will need them for the next steps.

###Unsealing the Vault###
By default the vault is sealed. While the vault is sealed, there really isn't much you can do on the administrative end. So, our first order of business will be to unseal the vault. Let us begin by checking the status with the following command:
```bash
$ ./vault status -ca-cert=root.cer
```
You should see a response similar to:
```
Sealed: true
Key Shares: 5
Key Threshold: 3
Unseal Progress: 0

High-Availability Enabled: false
```
What this means is that there are 5 distributed keys available (which you should have received during initialization) and it will take 3 of those 5 keys to unseal the vault. Using any of the three keys, unseal the vault:
```bash
$ ./vault unseal -ca-cert=root.cer <key 1>
$ ./vault unseal -ca-cert=root.cer <key 2>
$ ./vault unseal -ca-cert=root.cer <key 3>
```
As you enter each command you will be informed of your progress. Now if we check the vault status as we did earlier it should say that the vault is no longer sealed.

###Mounting the Backend###
Now that we have our sever securely up and running and unsealed we need to actually set it up to be used. Here we will be using the 'userpass' backend, which is perfect for managing account credentials. Before making the following configuration commands we need to verify that we have access to the root token (which we received following our initialization method). We can do this with the following command:
```bash
$ export VAULT_TOKEN=<root token>
```
Now that we have the proper permissions, we will begin by enabling the 'userpass' backend:
```bash
$ ./vault auth-enable -ca-cert=root.cer userpass
```
Now that we have enabled the backend we can add new accounts and verify username/password combinations. A user can be added via:
```bash
$ ./vault write -ca-cert=root.cer auth/userpass/users/<username> password=<password>
```
We can check if a user/password combination is valid with the following:
```bash
$ ./vault auth -ca-cert=root.cer -method=userpass username=<username> password=<password>
```
