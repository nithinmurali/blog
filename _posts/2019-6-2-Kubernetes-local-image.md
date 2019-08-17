---
layout: post
title: minikube use Local Image
---

Hey guys, 

By default you need to push to images to repo and need to pull inside minkube.

This are the steps to setup a local registry.

**Setup in local machine**

Setup hostname in local machine: edit `/etc/hosts` to add this line

    docker.local 127.0.0.1

Now start a local registry (remove -d to run non-daemon mode) :

    docker run -d -p 5000:5000 --restart=always --name registry registry:2

Now tag your image properly:

    docker tag ubuntu docker.local:5000/ubuntu

Now push your image to local registry:

    docker push docker.local:5000/ubuntu

Verify that image is pushed:

    curl -X GET http://docker.local:5000/v2/ubuntu/tags/list

**Setup in minikube**

ssh into minikube with: `minukube ssh`

edit `/etc/hosts` to add this line

    docker.local <your host machine's ip>

Verify access:

    curl -X GET http://docker.local:5000/v2/ubuntu/tags/list

Now if you try to pull, yo might get an http access error.

**Enable insecure access**:

    systemctl stop docker

edit the docker serice file: get path from ```systemctl status docker```

it might be : 

> /etc/systemd/system/docker.service.d/10-machine.conf  or 
> /usr/lib/systemd/system/docker.service

append this text (replace 192.168.1.4 with your ip)

> --insecure-registry docker.local:5000 --insecure-registry 192.168.1.4:5000

to this line 

> ExecStart=/usr/bin/docker daemon -H tcp://0.0.0.0:2376 -H
> unix:///var/run/docker.sock --tlsverify --tlscacert /etc/docker/ca.pem
> --tlscert /etc/docker/server.pem --tlskey /etc/docker/server-key.pem --label provider=virtualbox --insecure-registry 10.0.0.0/24

    systemctl daemon-reload
    systemctl start docker

try pulling:

    docker pull docker.local:5000/ubuntu

Now change your yaml file to use local registry.

>       containers:
>         - name: ampl-django
>           image: dockerhub/ubuntu

to 

>       containers:
>         - name: ampl-django
>           image: docker.local:5000/nymbleup


**Don't use http in production, make the effort for securing things up.**


  [1]: https://stackoverflow.com/a/46068235/3701807
