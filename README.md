# Gluster Server

This repository will help to create a GlusterFS server cluster using docker containers.

This example setup needs 2 servers and several dns entries. 
Serversnames are core-1 and core-2, with the following dns entries:
- core-1 -> gluster.core-1.mydomain
- core-2 -> gluster.core-2.mydomain

Use round-robin dns to core-1 and core-2 on the client.

Both servers are identical in this setup and use the same volume mounts.

Take care to use a private network since all ports are opened to the host interface.
The hosts /data directory filesystem needs to be glusterfs compatible, btrfs, xfs etc.

## Build

Build the docker image
```bash
docker build -t blang/gluster-server .
```

It's also available on dockerhub as `blang/gluster-server`.


## Bootstrap
Servers need a setup with some manual steps.

### Create a swarm cluster

Create a swarn cluster with 2 nodes (see https://docs.docker.com/get-started/part4/)

```bash
docker-machine create --driver virtualbox myvm1
docker-machine create --driver virtualbox myvm2
```
```bash
docker-machine ssh myvm1 "docker swarm init --advertise-addr $(docker-machine ip myvm1)"
export token=$(docker-machine ssh myvm1 "docker swarm join-token manager|grep token")
docker-machine ssh myvm2 "eval $token"
```

On the manager node, create a share network :
```bash
docker-machine ssh myvm1 "docker network create --attachable --driver overlay net-glusterfs"
```

### On both servers

Initilialize two nodes with ths script initialize.sh:

```bash
# password is tcuser for user docker
ssh-copy-id docker@$(docker-machine ip myvm1)
ssh-copy-id docker@$(docker-machine ip myvm2)
```


```bash
#!/bin/bash
function init() {
  index=$1
  echo "mkdir -p /data/glusterserver/data;" > init.sh
  echo "mkdir -p /data/glusterserver/metadata;" >> init.sh
  echo "mkdir -p /data/glusterserver/etc;" >> init.sh
  echo "cp /etc/hosts /data/glusterserver/etc/hosts;" >> init.sh
  echo "echo gluster.core-1.mydomain > /etc/hostname;" >> init.sh
  if [ $index = 1 ]; then
    echo "echo \"127.0.0.1 gluster.core-$index.mydomain\" >> /data/glusterserver/etc/hosts;" >> init.sh
    echo "echo \"$(docker-machine ip myvm2)  gluster.core-2.mydomain\" >> /data/glusterserver/etc/hosts;" >> init.sh
  else
    echo "echo \"127.0.0.1 gluster.core-$index.mydomain\" >> /data/glusterserver/etc/hosts;" >> init.sh
    echo "echo \"$(docker-machine ip myvm1)  gluster.core-1.mydomain\" >> /data/glusterserver/etc/hosts;" >> init.sh
  fi
  init=$(cat init.sh)
  if [ -n "$init" ]; then
    cmd="scp init.sh docker@$(docker-machine ip myvm${index}):~/"
    echo "INFO: execute: $cmd" && eval $cmd
    cmd="docker-machine ssh myvm${index} \"chmod 755 ~/init.sh\""
    echo "INFO: execute: $cmd" && eval $cmd
    echo "INFO: execute sudo -s and then ./init.sh"
    docker-machine ssh myvm${index}
  else
    echo "ERROR; init is empty"
  fi
}
init 1
init 2
```

### On core-2
Start gluster server non-interactive since setup is done on core-1.

```bash
docker-machine ssh myvm2
docker run -d --name gluster.core-2.mydomain --net=net-glusterfs --rm --privileged -v /data/glusterserver/data:/data -v /data/glusterserver/metadata:/var/lib/glusterd -v /data/glusterserver/etc/hosts:/etc/hosts -p 24007:24007 -p 24009:24009 -p 49152:49152 blang/gluster-server
docker exec -it gluster.core-2.mydomain glusterd && gluster peer probe gluster.core-1.mydomain
```

### On core-1
Start shell on core-1:
```bash
docker-machine ssh myvm1
docker run --name gluster.core-2.mydomain --net=net-glusterfs --rm --privileged -v /data/glusterserver/data:/data -v /data/glusterserver/metadata:/var/lib/glusterd -v /data/glusterserver/etc/hosts:/etc/hosts -p 24007:24007 -p 24009:24009 -p 49152:49152 -i -t blang/gluster-server /bin/bash
docker exec -it gluster.core-1.mydomain glusterd && gluster peer probe gluster.core-2.mydomain
docker exec -it gluster.core-1.mydomain gluster --mode=script volume create datastore replica 2 gluster.core-1.mydomain:/data/datastore gluster.core-2.mydomain:/data/datastore force
docker exec -it gluster.core-1.mydomain gluster volume start datastore
```

Inside the core-1 container:
```bash
# should return 127.0.0.1
ping gluster.core-1.mydomain

# core-2 should be reachable via private network
ping gluster.core-2.mydomain

# start glusterd
glusterd

# connect to core-2
gluster peer probe gluster.core-2.mydomain

# create volume 'datastore'
gluster --mode=script volume create datastore replica 2 gluster.core-1.mydomain:/data/datastore gluster.core-2.mydomain:/data/datastore force

# start volume
gluster volume start datastore

# info should return replicated set with 2 bricks
gluster volume info datastore

# status should print both nodes to be online 
gluster volume status datastore
```

Stop both containers now, if everything was successful, your containers are ready to go.

## Production
Run on each server:

on myvm1:
```bash
docker run -d --privileged --rm --name gluster.core-1.mydomain --net=net-glusterfs -v /data/glusterserver/data:/data -v /data/glusterserver/metadata:/var/lib/glusterd -v /data/glusterserver/etc/hosts:/etc/hosts -p 24007:24007 -p 24009:24009 -p 49152:49152 blang/gluster-server
docker exec -it gluster.core-1.mydomain bash
> glusterd && gluster peer probe gluster.core-2.mydomain
> gluster --mode=script volume create datastore replica 2 gluster.core-1.mydomain:/data/datastore gluster.core-2.mydomain:/data/datastore force
> gluster volume start datastore
> gluster volume status
> gluster volume info
> tail -f /var/log/glusterfs/glusterd.log
> mount -t glusterfs gluster.core-1.mydomain:datastore /mnt
> watch "ls -la /mnt| wc -l"
```

on myvm2:
```bash
docker run -d --privileged --rm --name gluster.core-2.mydomain --net=net-glusterfs -v /data/glusterserver/data:/data -v /data/glusterserver/metadata:/var/lib/glusterd -v /data/glusterserver/etc/hosts:/etc/hosts -p 24007:24007 -p 24009:24009 -p 49152:49152 blang/gluster-server
docker exec -it gluster.core-1.mydomain bash
> glusterd && gluster peer probe gluster.core-1.mydomain
> gluster --mode=script volume create datastore replica 2 gluster.core-1.mydomain:/data/datastore gluster.core-2.mydomain:/data/datastore force
> gluster volume start datastore
> gluster volume status
> gluster volume info
> tail -f /var/log/glusterfs/glusterd.log
> mount -t glusterfs gluster.core-2.mydomain:datastore /mnt
> for i in `seq -w 1 100`; do cp -rp /var/log/glusterfs/cli.log /mnt/copy-test-$i; done
```

Both server will connect to each other and heal automatically.
If you need to interact with the gluster interface later, start one of those servers in shell mode, start `glusterd` and use `gluster help`.

## Documentation


https://gluster.readthedocs.io/en/latest/Quick-Start-Guide/Quickstart/#installing-glusterfs-a-quick-start-guide
