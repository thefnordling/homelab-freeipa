# homelab-freeipa
this is the freeipa server configuration i have in my home lab environment

## Setup ##

can't run this on the swarm cluster due to the requirement for userns-remap, so i am running freeipa on a separate/dedicated host.

the first thing to do is to update `/etc/docker/daemon.json` and add this line (may need to create the file)

```
{ "userns-remap": "default" }
```

then restart the docker service.


after reviewing the [documentation](https://hub.docker.com/r/freeipa/freeipa-server), i went with the rokcy-9-4.*.* reccomendation as that was the most 'stable.

i'll eventually run this via docker compose, but we need an interactive tty/shell to complete initialization first. 

create a volume for the freeipa data

```
mkdir /opt/freeipa/data
mkdir /opt/freeipa/logs
```

give ownership to that folder to the uid:gid the service will run as

```
chown 165536:165536 /opt/freeipa/data
chown 165536:165536 /opt/freeipa/logs
```

run through intial setup by running the below command:
```
docker run --name freeipa-server-container -ti \
    -h ipa.home.arpa --read-only \
    --sysctl net.ipv6.conf.all.disable_ipv6=0 \
    -v /opt/freeipa/data:/data:Z freeipa/freeipa-server:rocky-9-4.10.1
```

after the installation has been completed, stop the container and then run the compose file


```
docker comopse up -d
```

the no-login-popup.conf file (mounted as a volume in the compose file) will prevent nginx tryingt to grab a kerberos token from the browser (and showing an annoying basic-auth popup when you first go to the ipa ui)

I was hoping to be able to have freeipa generate a CSR, which vault could sign generating an intermediate CA to be used for LDAP known by my PKI setup; however - that isn't possible at the moment since freeipa is only able to generate a CSR or support a CA using UTF8STRING encoded properties (organization and commonName) - but vault is only able to support PRINTABLESTRING encoded properties.   

Because of this i won't use a vault-signed/issued intermediate cert for LDAP; instead i will load my vault root CA into freeipa and then use a cert from vault (issued by an intermediate ca within vault) for the web hosting certificate.

attach a shell to the ipa container, and make a directory to work in, and copy over the vault root (ca.pem) and the intermediate (intermediate.pem - a chain including ca.pem), then tell ipa to load the updated certs
```
mkdir /cert-workflow
cd /cert-workflow
ipa-cacert-manage install ca.pem
ipa-cacert-manage install intermediate.pem
ipa-certupdate
```

it will take a minute (quite a few actually) for `ipa-certupdate` to finish

prepare the new httpd certificate for import.   freeipa needs a pkcs#12 formatted certificate (ugh) so use openssl to combine everything into a pkcs#12 file that will work

`openssl pkcs12 -export -out ipa.p12 -in cert.pem -inkey key.pem -chain -CAfile chain.pem`

`cert.pem` is the hosting cert - just the hosting cert no chained/intermediate certs
`key.pem` this is the private key for the hosting cert
`chain.pem` this contains the intermediate and root ca (in that order) in pem format

then copy ipa.p12 to the freeipa container and run 

```
ipa-server-certinstall -w -p {dir manager pwd here} -v ipa.p12
ipa-certupdate
```

if everything worked, go ahead and clean up the certs, and proceed to enrolling the rest of your hosts to freeipas
```
rm -f *
```

# Enrolling hosts to freeipa #

I'll run through the steps i used to configure my nginx reverse proxy `proxy.home.arpa` - just repeat for other hosts as needed (will work on ansible playbooks later)

update the hostname to the FQDN

edit /etc/cloud/cloud.cfg and set the parameter "preserve_hostname" from "false" to "true"

```
sudo hostname proxy.home.arpa
```

check /etc/hostname and update it to the hostname part (proxy in above example)

then install the freeipa client

```
sudo apt-get install sssd-tools freeipa-client -y


```
this will kick off the install - we're going to redo it so it doesn't matter what you enter at the prompts

then run
```
sudo ipa-client-install --mkhomedir
```
when prompted `Provide the domain name of your IPA server (ex: example.com):` enter home.arpa
when prompted `Provide your IPA server name (ex: ipa.example.com):` enter ipa.home.arpa
when prompted `Proceed with fixed values and no DNS discovery? [no]:` enter yes
when prompted `Do you want to configure chrony with NTP server or pool address?`: hit no (default)

then configure with those values at the last prompt

when prompted `User authorized to enroll computers:` enter admin
then do the password you set up during the ipa server install

you should be good - now, navigate back to the ipa server, go into hosts and confirm proxy.home.arpa is now shown there

to force a cache reload (on the client) run `sudo sss_cache -E`

to remove a user from the client host not sync'd from freeipa, run `userdel {username}`