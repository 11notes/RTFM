![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# DOCKER VOLUMES

*What are Docker volumes and why should I always prefer named volumes instead of bind mounts?*

# SYNOPSIS 📖

Docker volumes are the best way to persist data from your containers in an ephemeral way, yet many people use them completely wrong. Lets clear up this misconception and misuse of Docker volumes!

## TYPES OF VOLUMES

Docker knows exactly three types of volumes/mounts:

**Named:** Docker will create the volume for you in the path specified in your [daemon.json](https://github.com/11notes/RTFM/blob/main/linux/container/docker/daemon.json) you set via the ```data-root``` property. Docker will create folders that do not exist and use the UID/GID that is used for the executing process of the image.

**Bind:** Docker will mount the volume from the host into the container. Docker **will not create any missing folders** and **will not set any permissions** like the UID/GID executing the process in the image.

**Tmpfs:** Docker will mount a tmpfs directory into the container. This type of mount has multiple additional options compared to the other two types and is not really persistent since the data is kept in RAM (tmpfs) and swap and not on disk. Use this type of volume for temporary data, like transcoding, logs and other files you do not need and can be re-created.

# BIND MOUNTS AND THE HORROS BEYOND

So why is basically everyone using bind mounts and not named volumes? Well, a lot of compose examples on the internet feature this type of volume by default. It’s also the easiest to understand since it is identical to mounting a folder in Linux. Be it from an NFS server or from an external HDD. It works exactly the same, but since we use Docker as IaC[^1] we actually complicate things needlessly when using them.

The first problem arises is that bind mounts are not managed by Docker but by you. This simply means you are responsible for creating the entire folder structure and you are responsible for settings all the correct permissions and so on. Pretty tedious if you ask me when we are using Docker compose and all its IaC[^1] advantages.

Another limiting factor is that we can’t limit the size of a bind mount via compose, but we must set this quota directly on the host to ensure the bind mount can’t grow bigger than 10GB for instance. So instead of defining this property, 10GB limit, in my compose, I must configure this directly on the host, which is anti-IaC, since I now have to manage two sets of configurations: The host and the compose file.

# NAMED VOLUMES AS THE SOLUTION

Named volumes solve all of this, not only does Docker take care that all folders used are automatically created if missing, it also sets the correct permissions on everything, and if set, applies the quota setting limiting the container to a specific size. This is exactly what we want when using IaC[^1].

So why are people not using them? It’s for very silly reasons. Most people do not know you can configure the ```data-root``` property, and actually tell Docker where to store all the images and volumes. Another reason is that people think named volumes work only locally, while in fact you can use named volumes with any storage driver, be it NFS, CIFS or more exotic ones like S3. This gives us the ability to define the NFS mount directly in the compose file and not mount the folder first on the host, which would again break the whole IaC[^1] idea.

The last reason people are against using named volumes, is they think **they can’t directly edit the files** in those volumes, **which again is totally wrong**. You can mount a named volume to infinite containers and expose them in infinite ways. You can also edit the files directly on the host, whatever you prefer. If you need to pre-populate a volume with data before the container exists, simply create the volume first, mount the source directory and the volume and then copy everything you need over and then start the image. This whole process can even be added as a service in a compose file itself, to automate the whole ordeal and to adhere to IaC[^1] once more.

# NAMED VOLUMES EXAMPLES

```yaml
# normal type
name: "reverse-proxy"
services:
  traefik:
    image: "11notes/traefik:3.5.0"
    # ...
    volumes:
      - "var:/traefik/var"
      - "plugins:/traefik/plugins"
    # ...

volumes:
  var:
  plugins:


# NFS
name: "reverse-proxy"
services:
  traefik:
    image: "11notes/traefik:3.5.0"
    # ...
    volumes:
      - "var:/traefik/var"
      - "plugins:/traefik/plugins"
    # ...

volumes:
  var:
    driver_opts:
      type: "nfs"
      o: "addr=192.168.1.30,nolock,soft,nfsvers=4"
      device: ":/volume1/containers/traefik/var"
  plugins:


# CIFS
name: "reverse-proxy"
services:
  traefik:
    image: "11notes/traefik:3.5.0"
    # ...
    volumes:
      - "var:/traefik/var"
      - "plugins:/traefik/plugins"
    # ...

volumes:
  var:
    driver_opts:
      type: cifs
      o: username=service.traefik,password=${PASSWORD},domain=CONTOSO,uid=1000,gid=1000,dir_mode=0700,file_mode=0700
      device: //192.168.1.40/contoso/containers/traefik/var
  plugins:


# TMPFS
name: "reverse-proxy"
services:
  traefik:
    image: "11notes/traefik:3.5.0"
    # ...
    volumes:
      - "var:/traefik/var"
    tmpfs:
      - "/traefik/plugins:uid=1000,gid=1000"
    # ...

volumes:
  var:
```

# CONCLUSION

Now we know why named volumes are the better option and why we should always use them instead of bind mounts. The benefits are just too many to ignore them and the downsides are basically inexistent. Migrating from bind mounts to named volumes involves a simply copy process from the source directory, the bind mount, to the named volume, that’s it. I highly recommend switching over to named volumes for all your data, be it local, be it NFS, CIFS or even S3.

**If you want IaC you need named volumes and not bind mounts. Keep your compose files portable and not dependent on folders on the host.**

[^1]: Infrastructure as code, the concept that configurations of systems are done via a set of reproducible instructions and not manual execution of tasks or scripts