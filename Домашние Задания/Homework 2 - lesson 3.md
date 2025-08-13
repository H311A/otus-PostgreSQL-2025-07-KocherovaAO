# Установка PostgreSQL 
```
Развернута ВМ Oracle Linux 8.10 в VMware.
```
Установка Docker Engine:
```
[root@postgresql ~]# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
[root@postgresql ~]# sed -i 's/centos/rhel/g' /etc/yum.repos.d/docker-ce.repo
[root@postgresql ~]# yum install -y --allowerasing docker-ce docker-ce-cli containerd.io
[root@postgresql ~]# systemctl start docker & systemctl enable docker
[root@postgresql ~]# usermod -aG docker $USER
[root@postgresql ~]# newgrp docker

[root@postgresql ~]# docker --version
Docker version 28.3.3, build 980b856
[root@postgresql ~]# docker info
Client: Docker Engine - Community
 Version:    28.3.3
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.26.1
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.39.1
    Path:     /usr/libexec/docker/cli-plugins/docker-compose
```
