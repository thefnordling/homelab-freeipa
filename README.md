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