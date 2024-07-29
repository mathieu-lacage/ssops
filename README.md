# What is ssops about ?

[sops](https://github.com/getsops/sops) & [age](https://github.com/FiloSottile/age) 
are two tools used often to manage encryption and decryption of shared secrets.
They are extremly useful if you have a KMS like HC Vault but if you do not,
you are going to be typing your password a crazy number of times, especially if you like
to store encrypted secrets in ansible vaults.

[sshcrypt(https://github.com/leighmcculloch/sshcrypt) is another tool that
allows you to encrypt and decrypt data with the ssh keys managed by your local 
ssh-agent. 

ssops puts together all these tools to allow you to easily encrypt and decrypt files securely,
optionally by encrypting and decrypting your encryption keys with your local ssh-agent
so you only have to type your passphrase once.

# sops/age operation

We can reproduce a typical sops/age workflow by creating multiple key pairs, and then encrypting
content for each of these key pairs so only the owners of the relevant private keys can decrypt
the data:

```
# alex creates a key
alex@alex$ ssops key gen alex
Password:<alex>
Retype password:<alex>
# she creates an encryption method named 'dev' to share data with the dev team
alex@alex$ ssops method dev create
# she adds her key to this encryption method
alex@alex$ ssops method dev add-key alex
alex@alex$ git add dev
alex@alex$ git commit -m "Encryption method for developers"
alex@alex$ git push
# Then, she asks mathieu to create a key and add his key to the 'dev' team encryption method
mathieu@mathieu$ git pull
mathieu@mathieu$ ssops key gen mathieu
Password:<mathieu>
Retype password:<mathieu>
mathieu@mathieu$ ssops method dev add-key mathieu
mathieu@mathieu$ git add dev
mathieu@mathieu$ git commit -m "add my key to the team encryption method"
mathieu@mathieu$ git push
# we now have an encryption method that can encrypt data with two public keys
mathieu@mathieu$ ssops method dev show
name       | type       | embedded ?
------------------------------------
alex       | rsa        | no      
mathieu    | rsa        | no        
# Then, alex encrypts data for the 'dev' team
alex@alex$ git pull
alex@alex$ echo "hello" | ./ssops encrypt dev > encrypted
# and shares that data with the team
alex@alex$ git add encrypted
alex@alex$ git commit -m "new data to share with the dev team"
alex@alex$ git push
# And then, mathieu can decrypt this
mathieu@mathieu$ ./ssops decrypt < ./encrypted
Password for "alex":<ENTER>
Password for "mathieu": <mathieu>
hello
```

# SSH-based operation

Now, what sets ssops apart from sops and age, is that you can encrypt and decrypt
your private/public key pair with an ssh-agent that is able to decrypt data on your
behalf.

The only difference this time is that when you create the key, you need to specify
which SSH public/private keypair should be used to encrypt and decrypt your key pair:

```
alex@alex$ ssh-add ~/.ssh/id_rsa
Enter passphrase for /home/alex/.ssh/id_rsa: 
Identity added: /home/alex/.ssh/id_rsa (alex@fedora)
alex@alex$ ./ssops key gen alex --ssh ~/.ssh/id_rsa.pub
```

Alternatively, it is possible to change how a keypair is protected after it has
been created:

```
alex@alex$ ssh-add ~/.ssh/id_rsa
Enter passphrase for /home/alex/.ssh/id_rsa: 
Identity added: /home/alex/.ssh/id_rsa (alex@fedora)
alex@alex$ ./ssops key protect alex --ssh ~/.ssh/id_rsa.pub
```

# Embedding keys in encrypted output

By default, all private keys are stored only locally in ~/.ssops/ as encrypted
files and the expectation is that you are never going to copy them anywhere else.

This means that, if you want to decrypt content on multiple hosts, you should
generate multiple keys for yourselve, one per host and encrypt your
content with all these keys. This is the right thing to do in general.

In some cases, it might be oh so much more convenient to use a single key
per human identity and instead embed the encrypted private keys in the encrypted
content itself to allow decryption of the private keys and their content
on a remote host. 

Typically, this is very convenient to do with ssh-protected keys because
you just need to forward your ssh-agent from one host to another to be able
to password-lessly decrypt content  on remote hosts.

```
alex@local$ ssops key protect --ssh ~/.ssh/id_rsa.pub alex 
alex@local$ ssops method dev add-key --embed alex 
alex@local$ echo "hello" | ssops encrypt ./dev > encrypted
alex@local$ git commit -m "encrypted content with embedded encrypted keys" ./encrypted
alex@local$ git push
alex@local$ ssh alex@remote
alex@remote$ git pull
# passwordless !
alex@remote$ ssops decrypt ./dev < encrypted
```
