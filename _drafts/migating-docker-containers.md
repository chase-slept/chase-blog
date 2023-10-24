---
title: Migrating Docker Containers
#date: 2023-10-20 14:15 -0500
category: 
tags: []
---
## Migrating Docker Containers Between Hosts

Inevitably, it seems, we'll encounter a situation where a Docker container isn't quite in the right place. In my case, I had a handful of containers running on an Ubuntu server and needed to move them to a different server on my network. 

### Overview

Here's a rough outline of what I did to migrate them from one host to another.

#### Collect Info

- Note the container's bind mount location
- Stop the container
- Create a duplicate of the container on the new host
- Note the new container's bind mount location
- Stop the new container

#### Sync Data Betweeen Hosts

- Design and run an rsync command to sync data
- Restart new container on new host

Let's walk through this process with a little more detail, starting with the info collection phase. In most cases, you probably already know the paths your container uses as a bind mount because you set up the container. There may be scenarios where you don't---such as if you're working on someone else's existing containers, or if you used a tool like Portainer that semi-obscures this info to the user, or if you set up your container decades and decades ago and just forgot. In any case, we'll pretend you don't know the bind paths.

### Locating Container's Bind Paths

We'll need to figure out where our container stores its persistent data---things like config files, user data, working files, etc. Most containers will have some persistant data. If we spin up a new container instance on a different host by copying our Docker Compose or Docker run command exactly, we'll have a precise duplicate of the Docker instance **except** for any of those persistent volumes which live on the old host's filesystem. Ultimately, we want that volume's data to move to the new host too.

We can check for where a container's mount point is located in a number of ways. If you use Portainer, this info can be found by navigating to your container list by selecting **Containers** from the menu, selecting your container from the list to view its details, and scrolling down to the **Volumes** section where the bind mount paths are listed.

![Screenshot example of Volumes section](/assets/img/)

We can also find this path information from the CLI with:
{% highlight shell %}
    docker inspect container container_example
{% endhighlight %}
where *container_example* is the name the container. This should spit out a quite a bit of info about the container---we're looking for something like this:
{% highlight console %}
    "HostConfig": {
                "Binds": [
                    "/data/compose/23/<container_example>:/config:rw",
                    "/mnt/data/:/data:rw"
                ],
                ... }
{% endhighlight %}

Alternatively, if you're working with Docker Compose files, you should be able to pretty easily spot the paths we're looking for in the volumes section:

{% highlight yaml %}
    services:
    container_example:
        image: example_img/example_img:latest
        container_name: container_example
        volumes:
        - /path/to/data/container_example:/config
        - /mnt/data/:/data
        restart: unless-stopped
{% endhighlight %}

When I set my containers up originally, I set the path in my compose file to ./container_example, which translated to /data/compose/23/container_example once it was imported into Portainer. Portainer is a little weird with relative paths and sort of opaque about how things work behind-the-scenes anyway, so be careful!

### Creating a Duplicate Container

Once you've figured out where your existing container's data is located, you can move on to replicating it on the new host. To begin, we'll stop our existing container. This step is important---if your container has exposed ports, Docker can't create another container on your network using those same ports. Stop the container how you normally would, either in the Portainer/Docker Desktop UI or in the CLI with `docker stop container_example`. Once stopped, move over to the new host and create the container there using the same Docker Compose file, or with Portainer's duplication feature, or by running a docker command with the same parameters as the original container's (if you even have a record of this still!--one more reason to use Compose files). Once the container is successfully created, make note of the bind points used on the new host as we'll need those in a minute. After making sure the service is accessible/working (it will be missing your data and will act like a brand-new, unconfigured service), stop the new container and move back to the old host.

### Syncing the Two Volumes

