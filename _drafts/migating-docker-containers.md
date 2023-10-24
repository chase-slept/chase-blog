---
title: Migrating Docker Containers
#date: 2023-10-20 14:15 -0500
category: 
tags: []
---
## Migrating Docker Containers Between Hosts
Inevitably, it seems, we encounter a situation where a Docker container isn't quite in the right place. In my case, I had a handful of containers running on an Ubuntu server and needed to move then do a different server on my network. Here's a rough outline of what I did to migrate them from one host to another.

### Collect Info
- Note the container's host volume location
- Stop the container
- Create a duplicate of the container on the new host
- Note the new container's host volume location
- Stop the new container

### Sync Betweeen Hosts
- Design and run an rsync command to sync data
- Restart new container on new host

Let's walk through this process with a little more detail, starting with the info collection phase.

We'll need to first figure out where our container stores its persistent data--things like config files, user data, working files, etc. Most containers will have some persistant data. If we spin up a new container instance on a different host by copying our Docker Compose or Docker run command exactly, we'll have a precise duplicate of the Docker instance **except** for any of those persistent volumes that live on the old host's filesystem. Ultimately, we want that existing volume to move too.

We can check for where a container's volume is located in a number of ways. If you use Portainer, this info can be found by navigating to your container list by selecting **Containers** from the menu, selecting your container from the list to view its details, and scrolling down to the **Volumes** section where the paths are listed.

![Screenshot example of Volumes section](/assets/img/)

We can also find this path information from the CLI:
    docker inspect container container_example
where 'container_example' is the name the container. This should spit out something like this:

    "HostConfig": {
                "Binds": [
                    "/data/compose/23/<container_example>:/config:rw",
                    "/mnt/data/:/data:rw"
                ],