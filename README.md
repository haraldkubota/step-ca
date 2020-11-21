# Simple step CA setup

I plan to run step CA on a very small ARM PC. Very small power consumption, no noise. While I could use a containers for this, it does not add much value as that PC is on the lower end side.

## Prerequisites

* Linux installed (Debian 10)
* A non-root account to log in with a home directory and that user can become root via sudo without being asked for a password
* ansible installed: "apt install ansible" should do it

## Configuration

* ./files/key-pass.txt contains the passphrase for protecting the root and intermediary CA key. The latter is used by step-ca.
* ./files/provisioner-pass.txt contains the passphrase the step clients needs to use to request a certificate via JWK (default)
* ./vars/main.yaml contains variables used by the Ansible script. Mainly change the version and maybe the architecture

## Installation
This sets up everything.
```
$ ansible-playbook -i inventory playbook.yaml
```

You now have:
* ~/.step the CA files for step-ca
* a systemd service step-ca which runs and will re-run upon reboot
* a config file for above service in /etc/(default|sysconfig)/step-ca

In my example, my step CA is now reachable on https://ca.lan:8443

## Connecting from a step client

On a client machine, do:
```
❯ step ca bootstrap --ca-url https://ca.lan:8443 --fingerprint 812a...1f3c
The root certificate has been saved in /home/harald/.step/certs/root_ca.crt.
Your configuration has been saved in /home/harald/.step/config/defaults.json.
```
If you forgot your root CA fingerprint, do on the CA server:
```
❯ step certificate fingerprint $(step path)/certs/root_ca.crt
```

## Getting certificates

Using the default provider, on a client do:
```
❯ mkdir ~/.step/pass
❯ echo "efgh" >~/.step/pass/provisioner_pass.txt
❯ step ca certificate testserver.lan srv.crt srv.key --provisioner-password-file ~/.step/pass/provisioner_pass.txt
✔ Provisioner: myCA@home (JWK) [kid: It1q...saz0]
✔ CA: https://ca.lan:8443
✔ Certificate: srv.crt
✔ Private Key: srv.key
```

You can check the generated certificate:
```
❯ step certificate inspect srv.crt
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 252245498258449346334477191356099148818 (0xbdc4b741815fce4d892cbf018fe53c12)
    Signature Algorithm: ECDSA-SHA256
        Issuer: CN=myCA Intermediate CA
        Validity
            Not Before: Nov 21 05:41:40 2020 UTC
            Not After : Nov 28 05:42:40 2020 UTC
        Subject: CN=testserver.lan
[...]
```

## What next to do

Get the root certificate and install in your OS and browser's default certificate store:
```
❯ step ca root myCAroot.crt
The root certificate has been saved in myCAroot.crt.
❯ sudo cp myCAroot.crt /usr/local/share/ca-certificates/
❯ sudo update-ca-certificates
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
Running hooks in /etc/ca-certificates/update.d...

Adding debian:myCAroot.pem
done.
Updating Mono key store
Mono Certificate Store Sync - version 6.8.0.105
Populate Mono certificate store from a concatenated list of certificates.
Copyright 2002, 2003 Motus Technologies. Copyright 2004-2008 Novell. BSD licensed.

Importing into legacy system store:
I already trust 139, your new list has 140
Import process completed.

Importing into BTLS system store:
I already trust 139, your new list has 140
Import process completed.
Done
done.
```
You can verify this:
```
# Before

❯ curl https://ca.lan:8443/health
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

# After

❯ curl https://ca.lan:8443/health
{"status":"ok"}
```

