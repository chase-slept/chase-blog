---
title: Migrating Docker Containers
date: 2023-10-25 12:57 -0500
category: homelab
tags: [docker, homelab, guide]
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
- Restart the new container on the new host

Let's walk through this process in a little more detail, starting with the info collection phase. In most cases, you probably already know the paths your container uses because you configured the container's mount points to begin with. There may be scenarios where you don't---such as if you're working on someone else's existing containers, or if you used a tool like Portainer that semi-obscures this info to the user, or if you set up your container decades and decades ago and just forgot. In any case, we'll pretend you don't know the bind paths.

### Locating Container's Bind Paths

We'll need to figure out where our container stores its persistent data---things like config files, user data, working files, etc. Most containers will have some persistant data. If we copy our Docker Compose or Docker run command exactly and use that to spin up a new container instance on a different host, we'll have a precise duplicate of that container **except** for any of the persistent data volumes, since those live on the old host's filesystem. Ultimately, we want that data to move to the new host too, so we need to locate those paths on the old host to facilitate a data transfer later.

We can check for where a container's mount point is located in a number of ways. If you use Portainer, this info can be found in the UI by selecting **Containers** from the side menu, selecting your container from the list to view its details, and scrolling down to the **Volumes** section where the bind mount paths are listed.

![Screenshot example of Volumes section](/assets/img/volume-path-example.jpg)

We can also find this path information from the CLI with `docker inspect container container_example`, where *container_example* is the name of our target container. This should spit out a quite a bit of info---we're looking for something like this:
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

When I set my containers up originally, I set the path in my compose file to ./container_example, which translated to /data/compose/23/container_example once it was imported into Portainer. Portainer is a little weird with relative paths and sort of opaque about how things work behind-the-scenes anyway, so be careful! Despite its quirks, I've found it to a very handy tool for managing and adminstering Docker Compose files and their associated containers.

### Creating a Duplicate Container

Once you've figured out where your existing container's data is located, you can move on to replicating it on the new host. To begin, we'll stop our existing container. This step is important---if your container has exposed ports, Docker can't create another container on your network using those same ports. Stop the container how you normally would, either in the Portainer/Docker Desktop UI or in the CLI with `docker stop container_example`. Once stopped, move over to the new host and create the container there using the same Docker Compose file, or with Portainer's duplication feature, or by running a docker command with the same parameters as the original container (if you even have a record of this still!--one more reason to use Compose files). Once the container is successfully created, make note of the bind points used on the new host as we'll need those in a minute. After making sure the service is accessible/working (it will be missing your data and will act like a brand-new, unconfigured service), stop the new container and move back to the old host.

### Syncing the Two Volumes

Lastly, we'll need to figure out a way to move our existing data from the old host to the new host. Our two containers should be identical in how they're installed, but the configuration and other important persistent data is still stuck on the old host. This is where `rsync` comes in. Maybe you're already familiar with it, but this command is used to sync data between sources, which is exactly what we're trying to accomplish here. You should check the rsync man page if you're unsure of the arguments and parameters, but here are some of the crucial bits I ended up using:

`rsync [options] /source /destination`

- `--archive` (shorthand as `-a`, essentially the same as `-rlptgoD`) Preserves links, permissions, times, group, owner, special files---basically, this recursively syncs folder/file(s) in almost their exact state, while preserving special properties and links, etc.
- `--delete` This tells rsync to delete any files that already exist on the receiving side/destination. Essentially, this will allow our existing 'old' files to overwrite the default config files that were created when the container was spun up on the new host.
- `--dry-run` (`-n`) Adding this option will do a trial run and doesn't make any actual changes. Highly recommend toggling this on to test and confirm the files are transferring to the correct path and folder heirarchy on the new host.
- `--compress` (`-z`) Compresses files before sending over the network. Probably not necessary but may be useful for slower connections or larger datasets.
- `--progress --partial` (`-P`) This is useful for seeing the progress (like verbose, sorta) and keeps any partial files (such as when interrupted) to speed up their transfer later. Also not necessary, I just like the verbose output.

The full command I ended up with looks like this:

`sudo rsync -anzP --delete /data/compose/23/container_example/ root@10.0.0.10:/path/to/new/host/container_example`

Looking at the full command, there are a few things worth noting. Use sudo if you're not already the root user---same goes for the destination user. It's a good idea to test for the correct behavior first by setting the `--dry-run` (`-n`) flag; remove this flag when you've confirmed things are working and you're ready to sync for real. The origin path should have a trailing slash, which tells rsync that the folder's **contents** should be transferred. Lastly, our remote destination is specified much like an SSH command, using a username@hostname:path format.

After you've successfully run this command and transferred your data from the old host, restart the Docker container on the new host and confirm your service works as expected. It should have all of your existing data and be working as it did on the old host. Congratulations! There may be other steps needed to migrate things like databases, but for simply moving configuration files and other persistent data, this method should do the trick.

Hopefully this write-up has been useful for someone and, as always, thanks for reading!
