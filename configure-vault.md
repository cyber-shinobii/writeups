# How to Install and Configure HashiCorp Vault: A Beginner-Friendly Guide

*By Tobi Owolabi*

*6 min read*

---

HashiCorp Vault is a powerful tool for securely managing secrets like API keys, passwords, and certificates. If you’re new to DevOps or IT, this guide will walk you through installing and configuring Vault on a server in a clear and easy-to-follow way.

## Step 1: Install Vault

Vault needs to be installed on your server. For this guide, we’re using an AWS EC2 instance as our Vault server—a T2.micro instance with 8GB of storage, keeping costs low and ideal for testing purposes.

To install Vault, we need to install the required packages:

``sudo yum install -y yum-utils shadow-utils``

Next, let’s add the HashiCorp Repository:

``sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo``

Now we can install Vault:

``sudo yum -y install vault``

``vault -version``

## Step 2: Set Up TLS

Vault is now installed! In a production environment, it’s important to secure Vault with TLS (Transport Layer Security). While you could skip this step for testing, why not learn how to do it the right way from the start? To begin, let’s create a home directory for Vault.

``sudo mkdir /home/vault/``

``sudo chown -R vault:vault /home/``

Next, we’ll create a private key for the Certificate Authority (CA). A CA is a trusted entity that issues digital certificates to verify identities and enable secure communication.

``openssl genpkey -algorithm RSA -out ca-key.pem -aes256``

Now that we have the private key for our CA, let’s create a CA certificate.

``openssl req -key ca-key.pem -new -x509 -out ca-cert.pem -days 365``

Now that we have the CA’s key and certificate, we can use the CA’s key to sign Vault’s certificate, enabling clients to trust Vault during connections. First, we need to generate the Vault server’s private key.

``openssl genpkey -algorithm RSA -out vault-server-key.pem``

Next, we’ll create a Certificate Signing Request (CSR). The CSR is a file requesting a CA to sign a certificate.

``openssl req -new -key vault-server-key.pem -out vault-server.csr``

Before the CA approves the CSR to create Vault’s certificate, we need to create a SAN configuration file. This ensures the certificate includes Subject Alternative Names (SANs), which associate specific IP addresses, domain names, or hostnames with the certificate, allowing them to be securely accessed.

``sudo vi /opt/vault/tls/vault-san.cnf``

Add the following and make sure to update IP.2 to your server’s public IP address:

```[req_ext]
subjectAltName = @alt_names

[alt_names]
IP.1 = 127.0.0.1
IP.2 = YOUR_SERVER_PUBLIC_IP # update this to your server’s public IP address
DNS.1 = localhost
```

Now we can approve the CSR and use the CA’s key and certificate to create a signed certificate for Vault.

``sudo openssl x509 -req -in vault-server.csr -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial -out vault-server-cert.pem -days 3650 -extfile /opt/vault/tls/vault-san.cnf -extensions req_ext``

For AWS EC2 instances, there is a directory for trusted CA certificates. It’s best to copy our CA’s certificate there to ensure the system recognizes and trusts the CA.

``sudo cp ca-cert.pem /etc/pki/ca-trust/source/anchors/``

``sudo update-ca-trust``

## Step 3: Configure Vault

The certificates we created for Vault need to be stored in the /opt/vault/tls directory. Let’s clean up and move the certificates to the appropriate location.

``sudo mv * /opt/vault/tls/``

Now we’ll update Vault’s configuration file to point to the new certificates and ensure the api_addr is set to the correct IP address, which should be the public IP address of our server.

``sudo vi /etc/vault.d/vault.hcl``

Update the tls_cert_file to this: ``/opt/vault/tls/vault-server-cert.pem``
Update the tls_key_file to this: ``/opt/vault/tls/vault-server-key``
Add ``api_addr = "https://52.23.177.32:8200"`` after the HTTP listener stanza # this should be your server's public IP address. Try not to copy and paste these entries because of editor errors.

``` listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/vault-server-cert.pem"
  tls_key_file  = "/opt/vault/tls/vault-server-key.pem"
}

api_addr = "https://YOUR_SERVER_PUBLIC_IP:8200" # this should be your server’s public IP address
```

We’re almost done! Let’s ensure the Vault user has the necessary permissions for the /opt/vault directory. After that, we'll restart Vault to apply all the changes we've made.

``sudo chown -R vault:vault /opt/vault``

``sudo systemctl restart vault``

``sudo systemctl status vault``

## Step 4: Initialize Vault

Alright, let’s switch to the Vault user to start interacting with Vault. We’ll initialize it, log in, enable the secrets engine (used to store passwords and other sensitive data), and create a user account for logging in. While you could use the root token to log in, it’s not best practice—so we’ll set up a regular user account.

``sudo -i -u vault`` 

``vault operator init``

Vault is now initialized! Be sure to securely save the keys and token provided by the initialization command — store them in a safe location outside of Vault. Remember, Vault cannot be accessed until it is unsealed. This is a security feature that locks Vault from use until it’s explicitly unsealed with the keys provided. The next few commands will unseal Vault, allowing us to log in and continue configuring it for secure use.

Next, we need to unseal Vault. This requires a quorum of unseal keys (by default, 3 out of 5).

``vault operator unseal <key>``

Repeat this command two more times, each time entering a different unseal key.

Now we can log into Vault! To do this, we’ll use the root token provided during initialization since it’s the only access credential we have at this point.

``vault login <root-token>`` 

Perfect! Now we can set up our secrets engine to securely store sensitive information like passwords. Vault offers many different secrets engines, but for now, we’ll use the Key/Value (KV) secrets engine. Let’s enable it to get started!

``vault secrets enable -path=secret kv``

The Secrets Engine is now enabled! Let’s store a test secret that we can later validate using the Vault GUI. We’ll create a secret named ‘password’ with a value of ‘changemenow’. This will be stored under a path named splunk (secret/splunk).

``vault kv put secret/splunk password=changemenow``

Before we log into the GUI, let’s set up a user account by enabling the userpass authentication. This way, we avoid logging in with the root account and root token, which is not a best practice.

``vault auth enable userpass``

We’ll create an admin policy to grant the user access to the secrets engine and other system-related Vault functions. This policy will define the permissions the user needs to manage Vault effectively.

``mkdir -p /home/vault/policies``

``vi /home/vault/policies/admin-policy.hcl``

Add this: 

```path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

Now let’s apply the admin policy to make it recognizable and usable by Vault. This ensures the policy is available for assigning to users.

``vault policy write user-policy user-policy.hcl``

Now let’s add the user. In production, it’s best to add a user using a hashed password for security. However, we’ll demonstrate both the production way (with a hashed password) and a non-production way (with a cleartext password) in case you can’t use a hashing tool like bcrypt.

Install bcrypt and hash password:

```
python3 -m ensurepip — upgrade
python3 -m pip install bcrypt
python3 -c ‘import bcrypt; print(bcrypt.hashpw(b”changemenow”, bcrypt.gensalt()).decode())’
```

Add users:

`` vault write auth/userpass/users/itachi policies=”admin-policy” password_hash=”\$2b\$12\$ZMJOKjSemC7JLXP5yn6PF.8A.TXMVV248m2cgP5Iv5emW8D9/Sruu”``

``vault write auth/userpass/users/naruto policies=”admin-policy” password=”changemenow”``

Perfect! Users are now created, and we can log into Vault from the front end. Open your browser and go to ``https://<your-public-ip>:8200``. Make sure to switch the sign-in method to username.

## Congratulations! 
You’ve successfully installed and configured Vault! This setup provides a secure environment to manage secrets, passwords, certificates, keys, policies, and other user authentication. Keep experimenting to build your skills in managing secure systems.
