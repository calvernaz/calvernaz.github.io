+++
title = 'Data-Only Containers in Docker: Flexibility in Data Management'
date = 2024-02-05T08:26:47Z
draft = false
slug = "data-only-containers-data-management"
tags = ["Docker", "Data Management", "Data-Only Containers", "Containerization", "DevOps", "Software Engineering", "Data Portability", "Distributed Systems"]
description = "Exploring the enduring relevance and advantages of Data-Only Containers in Docker for efficient and flexible data management in containerized environments."
+++

We will explore how DOCs (data-only containers), as a deliberate and thought-out approach, for managing data in Docker.
DOCs are particularly interesting in scenarios requiring the distribution of static, unmodifiable data – such as packaged **website content**, 
**tenanted configuration files**, or **standardized test datasets**.

DOCs offers a reliable and efficient method to manage _data separation_ and _portability_ in Docker to deploy immutable 
data resources across different stages of development since they can be versioned, shared, and reused across different environments.

DOCs are not a new concept, but I believe they are still relevant and useful in modern containerized environments.

## What are Data-Only Containers?

Unlike standard containers, they do not run applications or services. Instead, they act as static volumes where 
data is stored and shared among other containers, but offering immutability and versioning.

Creating and using a data-only container is _kinda_ straightforward.
Here's how they can be created and used. 

**Creating a data-only container**:

Start by creating the data-only container using the `busybox` image, which is a minimalistic Linux distribution:

```bash
docker create -v /data --name static-data busybox
```

The `-v` flag specifies the volume to be created, and the `--name` flag assigns a name to the container.
Check the container's status using the `docker ps -a`, that has been created (not running):

```plaintext
$ docker ps -a

CONTAINER ID   IMAGE     COMMAND   CREATED              STATUS    PORTS     NAMES
626cd6581ecb   busybox   "sh"      About a minute ago   Created             static-data
```

There is a volume created using the `docker volume ls` command and the `docker inspect` command to view the volume's details:

```plaintext
$ docker volume inspect fbc19e487531933c774ef5b73f8846d1657377c0f38fd37a27fcbb5a3c942610
[
    {
        "CreatedAt": "...",
        "Driver": "local",
        "Labels": {
            "com.docker.volume.anonymous": ""
        },
        "Mountpoint": "/var/lib/docker/volumes/fbc19e487531933c774ef5b73f8846d1657377c0f38fd37a27fcbb5a3c942610/_data",
        "Name": "fbc19e487531933c774ef5b73f8846d1657377c0f38fd37a27fcbb5a3c942610",
        "Options": null,
        "Scope": "local"
    }
]
```
**Using data from the data-only container in another container**:

To use the data from the data-only container in another container, we can use the `--volumes-from` flag to mount the volume 
from the data-only container to the new container.

```bash
docker run -d --volumes-from static-data --name webapp nginx
```

Executing the `docker inspect` command on the `webapp` container, we can see the volume mounted from the `static-data` container:

```plaintext
$ docker inspect webapp
[
    {
        "Mounts": [
            {
                "Type": "volume",
                "Name": "fbc19e487531933c774ef5b73f8846d1657377c0f38fd37a27fcbb5a3c942610",
                "Source": "/var/lib/docker/volumes/fbc19e487531933c774ef5b73f8846d1657377c0f38fd37a27fcbb5a3c942610/_data",
                "Destination": "/data",
                "Driver": "local",
                "Mode": "",
                "RW": true,
                "Propagation": ""
            }
        ]
    }
]
```

In practice inside the running container,
there is a directory `/data` that contains the data from the `static-data` container.
Anyway, so far so good, this is pretty much how data-only containers work, and it's present in documentation and tutorials all over the place.

## The read-only use case

The _read-only_ use case is still the reason I find data-only containers relevant and the reason for this post.

### Background
You are managing a multi-tenant application where each tenant has its unique API keys, database credentials, and configuration settings. 
These secrets are critical for the operation of each tenant's instance of the application and must be securely managed to prevent unauthorized access.

Here is a simple illustration of how a Kubernetes pod can be configured with a secrets mount volume, under `etc/secrets/`
there are tenant-specific secrets that are accessible to the application container.

![Kubernetes Pod with Secrets Volume](/k8s-secrets.png "Kubernetes Pod with Secrets Volume")

### The Problem
The problem arises when we need to have a consistent application approach across the different runtimes,
docker in development and kubernetes, for example, in production.

### The Solution

#### The Docker Way

The solution is to use a data-only container to manage the secrets and configuration files for each tenant.

* We can build a data-only container from a `Dockerfile` that copies the secrets and configuration 
files into the container and declares a volume at the location of the secrets.

```Dockerfile
FROM busybox

# Copy the secrets file into the container
COPY tenant1-secret /etc/secrets/tenant1
COPY tenant2-secret /etc/secrets/tenant2
COPY tenant3-secret /etc/secrets/tenant3

# Declare a volume at the location of the secrets
VOLUME /etc/secrets
```

Tag the image and push it to a registry to share it across different environments and the team. 

* Create a data-only container from the image:

```bash
docker create -v /etc/secrets --name secrets secrets:latest
```

* Use the data-only container in another container:

```bash
docker run -d --volumes-from secrets --name api-gateway api-gateway:latest
```

Inside the `api-gateway` container, the secrets are accessible at `/etc/secrets`.
```plaintext
/etc# ls -lart secrets
total 12
-rw-r--r-- 1 root root    0 Feb  5 14:03 tenant1
-rw-r--r-- 1 root root    0 Feb  5 14:03 tenant2
-rw-r--r-- 1 root root    0 Feb  5 14:03 tenant3
```

#### The Docker-Compose Way

In a `docker-compose.yml` file, we can define the data-only container and the application container using the following configuration:

```yaml
version: '3.8'
services:
  secrets:
    build: .
    volumes:
      - secrets_volume:/etc/secrets
  api-gateway:
    image: api-gateway:latest
    volumes_from:
      - secrets
    ports:
      - "80:80"
    container_name: api-gateway

volumes:
  secrets_volume:
```

Again, the secrets are accessible at `/etc/secrets` inside the `api-gateway` container.


## Conclusion

Hopefully, this post has been able to shed some light on the relevance and advantages of data-only containers and
how they can be strategically used.
The example used, offers a consistent approach
to making secrets available to the application container across different runtimes,
however, for production uses,
we should use external secrets backends like Hashicorp Vault or AWS Secrets Manager instead of files.  
