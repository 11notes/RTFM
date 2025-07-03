![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# DISTROLESS CONTAINER IMAGES

*What are distroless container images and why are they the best option to run applications as containers?*

# SYNOPSIS 📖

A distroless image is a container image that is not using the default file system of any operating system. A container image like [alpine](https://hub.docker.com/_/alpine) is very small, but contains all needed binaries to run said operating system in a container. This has advantages but also major disadvantages. Let's talk about the advantages first:

## ADVANTAGES OF A FULL IMAGE
> [!TIP]
>* You can ```docker exec -ti``` into the shell of the image (bash/sh/ash) and use it like any normal Linux and run any commands which are present
>* It contains many binaries that can help you if you have problems with a container image (ls, netstat, grep, find)
>* Binaries that need shared libraries can find them just like on a bare metal installation

Sadly, these advantages are also disadvantages when it comes to container and **security**. Having a shell is problem number one. If an app inside a container gets exploited through a remote code execution, the attacker can now use the shell to do anything inside the container image. If the image has access to the internet, an attacker can use ```curl``` or ```wget``` to download malicious binaries to poison the image even further or even place malicious content on your volumes. This brings us to the disadvantages:

## DISADVANTAGES OF A FULL IMAGE
> [!CAUTION]
>* Since they contain a lot of binaries from the OS in question, they are a lot bigger in image size on disk
>* An exploited app inside the container gets access to a plethora of tools to further exploit the image or the network or even the host itself (shell, downloads, permission management, setuid) as well as your persistent volumes

# WHY DISTROLESS IS BETTER FOR SECURITY

How can distroless images solve these issues (but introduce new ones)? Well, since they are distroless, they do not have binaries of an operating system. No ```ls```, no ```curl``` and no ```sudo```. An attacker that exploited the app within the container, gets access to basically nothing, and that’s the whole point. To prevent access to system binaries that could be used for further exploits. This prevents lateral movement within the container or the running container host. Nothing is without flaws, so let’s see the disadvantages of distroless:

## DISADVANTAGES OF DISTROLESS
> [!CAUTION]
>* Only statically linked binaries work, dynamically linked binaries depend on OS provided libraries like OpenSSL and zlib
>* You can’t ```docker exec -ti``` into the image, you can however access the image the same way by switching to it’s namespace[^1]
>* You can't inspect and see the file system of the image via ```docker exec -ti ls -lah *```, but you can on the host[^2]

As you can see in the footnotes, we can solve some of the disadvantages, from a day-to-day operations point of view, quite easily. The static linking requirement can also be solved by compiling the binaries as such, this requires a little more effort during the compilation of the app in question and also longer compilation time, but it’s possible for almost all apps to do this. Sadly, most official images or providers do not statically link their binaries, which means you have to do it yourself or use image providers that do ship distroless, like a lot of my images do (you can check out my github repo).

# CONCULUSION

Now we know why distroless is better in terms of security, but requires more effort on the image creator’s side. **If you can, always opt for distroless** when running an image. If the image provider doesn’t have one, you can gladly reach out to me on this repo to ask if it’s possible to create a distroless image for your app in question. There are apps which can’t be run distroless, basically all of python is very difficult to run true distroless because you can’t really efficiently compile python to a static binary. If the app in question is based on C or Go, then it’s pretty simple to create a static binary that can then be run distroless, unless the app creator actually prevents this by depending on dynamic links (which can happen sometimes) and refuses to change the code.

**If you want better security for your container images, run them distroless.**

[^1]: You can execute commands from your host OS inside another namespace via ```nsenter```, like this: ```nsenter -t $(docker inspect -f '{{.State.Pid}}' adguard-app-1) -n netstat -tulpn```. This means you can execute binaries which are not present in the distroless image from your host OS as if they were in the image, pretty neat
[^2]: If you get the process ID of the container image you can view the file system under ```ls -lah /proc/$(docker inspect -f '{{.State.Pid}}' adguard-app-1)/root/*```