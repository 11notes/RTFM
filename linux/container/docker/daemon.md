![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# /etc/docker/daemon.json

*What is the Docker daemon.json configuration and why would I ever need to change it?*

# SYNOPSIS 📖

The Docker daemon.json, located at ```/etc/docker``` holds configuration data to configure the Docker Daemon (its runtime settings). This is a very useful configuration file to set some default for Docker itself, but also for all the images that are run by Docker. Depending on your distro, there might already be a file present or you can create one yourself. The settings in this file will (or better should) overwrite any setting that was set by whatever installation method you took to install Docker.

Here is the daemon.json used in this RTFM repository:

```json
{
  "hosts": ["unix:///run/docker.sock"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "1",
    "labels": "production_status",
    "env": "os,customer"
  },
  "data-root": "/opt/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.size=4G"
  ],
  "bip": "169.254.253.254/23",
  "fixed-cidr": "169.254.252.0/23",
  "default-address-pools":[
    {"base":"169.254.2.0/23","size":28},
    {"base":"169.254.4.0/22","size":28},
    {"base":"169.254.8.0/21","size":28},
    {"base":"169.254.16.0/20","size":28},
    {"base":"169.254.32.0/19","size":28},
    {"base":"169.254.64.0/18","size":28},
    {"base":"169.254.128.0/18","size":28},
    {"base":"169.254.192.0/19","size":28},
    {"base":"169.254.224.0/20","size":28},
    {"base":"169.254.240.0/21","size":28},
    {"base":"169.254.248.0/22","size":28}
  ],
  "mtu": 9000,
  "dns": ["DNS1","DNS2"],
  "registry-mirrors": ["https://docker.domain.com"]
}
```

That doesn’t tell us much, so let’s break it down a little.

# HOSTS AND LOGS

```json
"hosts": ["unix:///run/docker.sock"],
"log-driver": "json-file",
"log-opts": {
  "max-size": "10m",
  "max-file": "1",
  "labels": "production_status",
  "env": "os,customer"
},
```

```hosts``` tells the Docker Daemon to listen on this interface. This can be a network interface with an IP and a port or in this default, the UNIX socket exposed via the socket file located at ```/run/docker.sock```. Anyone with access to this file, can directly interact with the Docker socket without any form of authentication. This means access to this file has to be very well guarded. If you plan to expose this file to a container which needs to interact with the socket, please do so with a secure socket-proxy like my own [11notes/socket-proxy](https://github.com/11notes/docker-socket-proxy) image.

```log``` and any of the adjacent options, simply tell Docker how to create the log files for each container. You can and maybe should, replace the standard logging to a file, with logging to a collector, like Loki, this is all possible. The standard is to log to *.json* files. You can set how many log files should be kept (```max-file```) and how big a file can grow before it gets rotated (```max-size```). Setting ```labels``` means that the log will log all container labels that are called *production_status*, same goes for ```env``` with what variables you want to log.

# STORAGE

```json
  "data-root": "/opt/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.size=4G"
  ],
```

```data-root``` is the most important one. It will tell Docker to store everything there, this has a huge impact on performance as well as the abilities you gain from that. It is recommended to store everything Docker related on an XFS formatted volume. Simply create a LVM volume and format it with XFS and mount it on ```/opt/docker```.

```storage-driver``` is the driver used and supported by the file system from ```data-root```, since we want to use XFS to get the best performance and most capabilities, this defaults to overlay2.

```storage-opts.overlay2.size``` lets us configure options of this storage driver. For the overlay2 driver we can set a default image size. This will limit any volume to a maximum of this size. You can still set individual max sized on each volume itself, but all volumes will be allow to be maximum this size.

It is also adviced to move the entire Docker installation to the ```data-root```, not just the volumes and images.

# NETWORKING

```json
  "bip": "169.254.253.254/23",
  "fixed-cidr": "169.254.252.0/23",
  "default-address-pools":[
    {"base":"169.254.2.0/23","size":28},
    {"base":"169.254.4.0/22","size":28},
    {"base":"169.254.8.0/21","size":28},
    {"base":"169.254.16.0/20","size":28},
    {"base":"169.254.32.0/19","size":28},
    {"base":"169.254.64.0/18","size":28},
    {"base":"169.254.128.0/18","size":28},
    {"base":"169.254.192.0/19","size":28},
    {"base":"169.254.224.0/20","size":28},
    {"base":"169.254.240.0/21","size":28},
    {"base":"169.254.248.0/22","size":28}
  ],
  "mtu": 9000,
  "dns": ["DNS1","DNS2"],
```

```bip``` and ```fixed-cidr``` is our default bridge network for containers that don't define a ```networks:``` property in the compose. ```bip``` is the default gateway used on this network.

```default-address-pools``` are the IP address pools that Docker is allowed to create itself when using default settings for networking. As you can see the entire 169.254/16 subnet is split into very weird /23, /22, /21 network chunks. From these network chunks Docker is allowed to create a /28 subnet for each defined ```networks```.  A /28 means that each defined network can only assign a maximum of 14 IP addresses, so in theory you can run a maximum of 14 container images that use networking in a single ```networks:``` definition. Since we can define multiple ```networks:``` however we like, this is not really a limitation, unless you really need more than 14 containers in a single network (hint: containers can communicate between multiple networks too!).

```mtu`` is the default MTU size used for each network defined. It must align with the physical switchport MTU. If your network is not using jumbo frames (aka MTU 9000), the default is 1500. Some applications, like file sharing, video streaming and other apps that send large amount of data, benefit from an MTU setting of 9000, but only if your entire network supports MTU 9000. Check your switches and routers if they do support jump frames.

```dns``` are the default DNS servers used for each container. You can specify individual DNS servers via compose, but these are the default ones used. If you run your own DNS servers, it makes sense to set these here.

# NETWORKING 169.254.0.0/16

You have probably never seen this subnet and might think this is not a private subnet, but a public one. This is not the case. The 169.254/16 subnet is a special subnet that can be used on private networks, with an additional *drawback*. The subnet is non-routed and can only exist on a L2 network between devices which are all on the same L2 network. This makes it  useless if you want to route networks, but very useful for containers, which are by definition, in their own flat L2 network. Using this subnet also prevents any IP collisions from occurring on actual networks, since no normal network is using this subnet.

# MIRROS

```json
  "registry-mirrors": ["https://docker.domain.com"]
}
```

```registry-mirrors``` is a very useful setting when you are concerned about the security of your docker hosts as well as being rate limited by Docker hub itself. With this setting you can tell Docker to download the images from these mirrors, instead of using the official ones. This gives you the ability to run a local mirror (for instance via my [11notes/registry-proxy](https://github.com/11notes/docker-registry-proxy). This mirror will download all the images on behalf of all your Docker hosts, and all your Docker hosts will pull all images from this mirror. This means we can take your Docker hosts offline, with no WAN access to increase security. Since the local mirror will locally store all images we have pulled, this removes the rate limiting of Docker hub when you pull the same image 150 times an hour.


# CONCLUSION

You should never use the defaults provided to you by some distro or Docker itself, always adapt them to your use case and what works best for you. Limiting the MTU to 1500 by using the default configuration can have performance impact for a container that can benefit from MTU 9000 for instance!

**Defaults can never capture the complexity of an individual installation.**