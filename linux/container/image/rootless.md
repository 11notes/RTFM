![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# ROOTLESS CONTAINER IMAGES

*What are rootless container images and why are they the best option to run applications inside containers?*

# SYNOPSIS 📖

A rootless container is simply put a container that by default will not execute as root but as a pre-defined user during the creation of the image. A build file for Docker for instance, can contain the ```USER``` directive to set the user during the build process but also at the end. During the build process you most certainly want to execute everything as root, since you want to be able to install packages, modify the file system of the container and much more. The last step of a build file is the execution step, where the builder of the image instructs the container runtime what process to start and as **who**. If no user us specified it will **default to root.** You can however, specify any arbitrary UID and GID to start the container as. Let's check a simple build file with no user:

```dockerfile
FROM alpine
```

Now build this image and check the ID/GID of the user that runs inside the container image:

```sh
docker buildx build -f rtfm.dockerfile -t rtfm/rootless .
docker run --rm rtfm/rootless id
uid=0(root) gid=0(root) groups=0(root),0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

As expected, the id of the process running inside the container is root (0:0). Now this is something we want to avoid for a number of reasons[^1], but how can we do that? Well, quite simple: Change the user as the last execution step in a build file.

```dockerfile
FROM alpine
USER 1000:1000
```

Build the image again and run it:

```sh
docker buildx build -f rtfm.dockerfile -t rtfm/rootless .
docker run --rm rtfm/rootless id
uid=1000 gid=1000 groups=1000
```

Well, that’s what we want. The process inside the container is not executed as root, but as ```1000:1000```. Why does this matter though? I thought containers are by default secure? Sadly, they are not, and there is a huge misconception about this topic and some fair amount of misinformation, especially from certain container image providers that make you believe running a process as root inside a container poses almost no risk at all.

# THE LIES THEY SPREAD ABOUT ROOT

Let’s talk about capabilities and the Linux kernel real quickly. Capabilities give you access to certain Linux kernel features. Normal users do not get any caps at all, but root does, even as containers. Let's quicky build an image with a binary needed to display our Linux capabilities:

```dockerfile
FROM alpine
RUN set -ex; \
  apk --update --no-cache add libcap;
```

```sh
docker buildx build -f rtfm.dockerfile -t rtfm/caps .
docker run --rm rtfm/caps capsh --print

Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=ep
```

Now that's a lot of capabilities. Just to quickly compare it when running the container as a different user:

```sh
docker run --rm -u 1000:1000 rtfm/caps capsh --print

Current: =
```

As you can see, root has many Linux caps, while the user 1000 has none, just like it should be. So why does root in a container, and isolated namespace, have any capabilities at all? Well, Docker and other container orchestrations ship with **default capabilities** for each container. Why is it done that way? Because most of the capabilities are needed for an app running inside the container, but most does not mean all, and herein lies the issue with root inside a container.

Why give an app that runs inside the container [cap_net_raw]( https://man7.org/linux/man-pages/man7/capabilities.7.html) if it does not need that functionality? Well that’s exactly what you are doing if you execute the container as root. It get’s even worse if you copy/pasted some random compose.yml that contains for whatever reason the ```privileged: true``` flag. With this flag enabled, root within the container can assign itself **all capabilities** including the ones to access the host from within the container. If you execute said container as a user, not root, this is not possible, even though this flag was present.

# S6 AND ROOT

Now address the elephant in the room: Starting a container as root but then using [setuid](https://man7.org/linux/man-pages/man2/setuid.2.html) to drop to another UID for the actual app. A very well known image provider does exactly that. They use [s6](https://github.com/just-containers/s6-overlay) to run multiple services inside a container, which on its own is a little bit against the idea of containers in the first place (one service = one container). The issue doesn’t stem from the multiple services though, but from s6 requiring to start the container as root to then use setuid to change to another user you can define via an environment variable.

The app does not run as root, it runs as the UID specified, so far so good, but the container starts as root, which means anything in the pre-startup phase is executed as root. This means anything in that pre-startup phase (aka s6) can be abused and misused to do all kinds of things. Now is this easy to do or achieve? No. It’s a highly sophisticated attack from within the app inside the container, but it is possible and that’s the problem. An attacker can use s6 to give itself more capabilities which are present thanks to s6 being run as root, something that does not work if s6 would not be executed as root (which doesn’t work because of setuid issue).

This image provider uses root and s6 for convenience, foremost to ```chown``` the directories with the correct UID/GID when you use bind mounts and you forgot to set the correct permissions. This tradeoff between security and convenience is not a very good trade in my opinion, ignoring the fact that s6 is also used to run a single service inside a container, something s6 is not needed for. It also prevents said image provider from using [distroless](https://github.com/11notes/RTFM/blob/main/linux/container/image/distroless.md) images all together, because s6 relies on OS libraries to work.

# S6 AND PSEUDO ROOTLESS (SUID PROBLEM)

Even if you start an s6 based image with a user, like from the very well known provider, there is an executable present in the image that will **always execute as root**, that’s also why said image provider does not allow you to set the ``` no-new-privileges:true``` flag, because this prevents this executable to run as root, breaking the whole **rootless** image lie. Don’t believe me? Well, simply search any image that uses s6 or pretends to be rootless with ```find / -user root -perm -4000 -exec ls -ldb {} \;```. This will show you all executables that will execute as root, even if you are not running as root.

What do you find in all images of said image provider that uses s6 for everything:
```
-rwsr-xr-x 1 root root 42216 May  7 08:23 /package/admin/s6-overlay-helpers-0.1.2.0/command/s6-overlay-suexec
```

What happens if you start an image like this while preventing new Linux caps:
```
docker run --rm -ti --user 7000:7000 --security-opt=no-new-privileges:true ...
s6-overlay-suexec: fatal: child failed with exit code 100
```

To no ones surprise, it does not work. As you can see, this image provider is lying to you and hiding the fact that there is an executable that will execute as root, no matter what user you set when starting the image. A true rootless image does not have an executable in it that can be executed as root, it is by default rootless and can only run as a pre coded user ID.


# ROOTLESS RUNTIMES OR USERNS MAP

The solution, besides running rootless images, is to simply run a rootless container runtime, like Podman, k8s, sysbox and so on. Docker itself can be run rootless too, and requires some preparation to do so. You can also use userns-remap to remap root inside the container to an arbitrary user on the host, eliminating most of the issues which are presented by images that use root. The problem with all of this is, that by default, most people use Docker, and Docker is by default **not rootless**. This means that most people are affected by the issues described and should opt, whenever possible, always to use **rootless images** and not rely on some promises of an image provider that root can’t be exploited in their images, which it totally can and in the future will be.

# CONCLUSION

Now we know why rootless images are the better option to run apps withing a container. We also know that certain image providers lie to you about their capabilities of running images rootless. It requires a little more preparation for the image creator to make their images run rootless by default, but it’s possible for any apps. If an app requires special privileges, instead of granting the entire container image said capabilities, the maintainer of the image can also simply assign the privilege to the binary that requires said capability, reducing the attack surface even more. Combined with [distroless](https://github.com/11notes/RTFM/blob/main/linux/container/image/distroless.md) images, rootless is the best approach to secure images for all and not just a select few who know what they are doing.

**If you want better security for your container images, run them rootless from the start, not after they started.**

[^1]: Root inside a container is not the same as root on the host, but root does get a lot of system capabilities that can be misused for container exploitation. If the container image was started with the privileged flag, root can now give itself any and all privileges and actually access the host from within the container.