![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# DOCKER SOCKET

*What is the Docker socket accessed by so many images and why should it not be done?*

# TL;DR - FOR BEGINNERS
Never expose the Docker socket to anything without the use of a socket-proxy in-between! If an app needs access to your Docker socket, always ask **why does this app need access to my Docker socket in the first place?**

# SYNOPSIS 📖

The Docker socket is a simple UNIX socket allowing you to connect to the Docker API, instead via TCP, directly via a file. Many images that need to read or interact with the Docker socket need access to it, but most of the images do it with full permissions and not just the subset of instructions they actually need.

## DOCKER.SOCK:RO TO THE RESCUE

You might see this a lot in compose examples:

```yaml
services:
  image:
    ...
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ...
```

If you are familiar with how Docker threats the ```:ro``` flag at the end of the volume, you might think **read-only** in this context means you can only access the socket with read permissions. **WRONG!** The ```:ro``` flag instructs the Docker daemon to mount this file as read-only, so the image can't delete the file or rename it, that's it. It does not prevent any form of access to the socket itself. Any image with access like this can do whatever it wants, even create containers that will give the image itself root access to your host[^1]! **Never expose your Docker socket like this!**.


# USE A SOCKET-PROXY, PROBLEM SOLVED!

You are correct, use a socket-proxy between the Docket socket and any app that needs access to it except a certain kind of app. If you need to use container management engines like Portainer, Dockge, Komodo. You must expose the entire socket with full permissions to these apps or they simply won’t work. If you use such apps, you must accept the fact that you are giving full control over Docker on your server to a third party app. These apps, can in theory, do whatever they want, even create containers that get access to the host[^1] itself. That’s why these apps should never be used if you care about security and integrity of your systems.

A socket-proxy, just like a reverse proxy, can block access to the Docker socket in a certain way, sadly, most socket-proxies that exist, run not [rootless](https://github.com/11notes/RTFM/blob/main/linux/container/image/rootless.md) nor [distroless](https://github.com/11notes/RTFM/blob/main/linux/container/image/distroless.md). Which destroys the entire premise of a secure and restricted access to the Docker socket, when the proxy itself is now the problem and not secure.

How can we solve this? Actually, pretty easy. We should not rely on apps that need a distro as proxy apps, we should not run the app exposing the socket as root at all, but we must access the socket from within the app as root or at least as the user with the correct permissions for the socket. The most used proxy images **all fail to do this**, they also fail to expose the socket as **true read-only**. Most images that need to access the Docket socket in the first place, only do this to **read** information about the running containers or want to get informed by events (start, stop, create). They do not need to give the Docker API any commands to create new containers or add the ```privileged: true```flag. If you have such an app, like [Traefik](https://github.com/11notes/docker-traefik) that can access the Docker socket to automatically create reverse proxy entries based on your labels, then you want to use a true [rootless](https://github.com/11notes/RTFM/blob/main/linux/container/image/rootless.md) and [distroless](https://github.com/11notes/RTFM/blob/main/linux/container/image/distroless.md) image that is **read-only** like my own [11notes/socket-proxy](https://github.com/11notes/docker-socket-proxy).


# CONCLUSION

Now we know why distroless is better in terms of security, but requires more effort on the image creator’s side. **If you can, always opt for distroless** when running an image. If the image provider doesn’t have one, you can gladly reach out to me on this repo to ask if it’s possible to create a distroless image for your app in question. There are apps which can’t be run distroless, basically all of python is very difficult to run true distroless because you can’t really efficiently compile python to a static binary. If the app in question is based on C or Go, then it’s pretty simple to create a static binary that can then be run distroless, unless the app creator actually prevents this by depending on dynamic links (which can happen sometimes) and refuses to change the code.

**If you want better security when running apps that need access to the Docker socket, use a socket proxy that follows a rootless and distroless design.**

[^1]: An app with access to the Docker socket can create and run a new container with the privileged flag enabled and executed as root and then access the host via said container. The app can also access all volumes of all containers and therefore access all persistent and mostly private data.