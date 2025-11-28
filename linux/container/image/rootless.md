# KNOW-HOW - COMMUNITY EDUCATION
This RTFM is part of a know-how and how-to section for the community to improve or brush up your knowledge. Selfhosting requires some decent understanding of the underlying technologies and their implications. These RTFMs try to educate the community on best practices and best hygiene habits to run each and every selfhosted application as secure and smart as possible. These RTFMs never cover all aspects of every topic, but focus on a small part. Security is not a single solution, but a multitude of solutions and best practices working together. This is a puzzle piece; you have to build the puzzle yourself. You'll find more resources and info’s at the end of the RTFM. Here is the list of current RTFMs:

- 📖 Know-How: Distroless container images, why you should use them all the time if you can! **[>>](https://github.com/11notes/RTFM/blob/main/linux/container/image/distroless.md)**

# ROOTLESS - WHAT IS THAT?
Everybody knows root and who he is, at least everybody that is using Linux. If you don’t, read the [wiki article](https://en.wikipedia.org/wiki/Superuser) about him first, then come back to this RTFM. Most associate root with evil, which can be correct but is not necesseraly true. So what does root have to do with rootless? A container image runs a process (preferable only a single process, but there can be exceptions). That process needs to be run as some user, just like any other process does. Now here is where the problem starts. What user is used to run a process within a container is dependend on the container runtime. You may ask what the hell a container runtime is, well, these things here:

-	Docker
-	Podman
-	Sysbox
-	LXC
-	k8s (k3s, k0s, Rancher, Talos, etc)

The experts in the audience will now point out that most of these are **not container runtimes** but *container orchestrators*, which of course, is correct, but for the sake of the argument, pretend that these are just container runtimes. Each of these will execute a process within a container with a default user and will use that user in some special way. Since the majority of users on this sub use Docker, we focus **only on Docker**, and the issues associated with it and rootless. If you are running any of the other *"runtimes"* you can ignore this know-how and go back to your previous task, thank you.

**I run Docker rootless so why should I care about this know-how?** Good point, you don’t. You too can go to your previous task and ignore this know-how.

# ROOTLESS - THE EVIL WITHIN
Docker will start each and every process inside a container **as root**, unless the creator of the container image you are using told Docker to do otherwise or you yourself told Docker to do otherwise. Now wait a minute, didn’t your friend tell you containers are more secure and that’s why you should always use them, is your friend wrong? Partially yes, but as always, it depends. You see, if no one told Docker to use any other user, Docker will happily start the process in the container as root, but not as the super user root, more like a crippled disabled version of root. Still root, still somehow super, but with less privileges on your system. We can easily check this by comparing the [Linux capabillities]() of root on the host vs. root inside a container:

**root on the Docker host**
```
Current: =ep
Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read,cap_perfmon,cap_bpf,cap_checkpoint_restore
```

vs. 

**root inside a container on the same host**
```
Current: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap=ep
Bounding set =cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```

vs. 

**a normal user account (doesn't have to exist)**
```
Current: =
Bounding set =
```

We can see that root inside a container has a lot less [caps](https://man7.org/linux/man-pages/man7/capabilities.7.html) than root on the host, but why is that? Who is the decider for this? Well it’s Docker. Docker has a [default set of caps](https://docs.docker.com/engine/containers/run/#runtime-privilege-and-linux-capabilities) that it will automatically grant to root inside a container. Why does Docker do this? Because if you start looking at the granted caps, you see that most of these are not exactly dangerous in the first place. ```cap_chown``` for instance gives root the ability to chown, pretty obvious stuff. ```cap_net_raw``` might be a little too much on the other hand, since it allows root to basically see all traffic on all interfaces assigned to the container. If you by any chance copied from a compose the setting ```network_mode: host```, then root can see all network traffic of the entire host. Not something you want. It gets worse if you for some reason copy/pasted ```privileged:true```, you give root the option to escape on the host and then do whatever as actual root on the host. We also see that the normal user has no caps at all, nada, and that’s actually what we want! Not a handicapped root, but **no root at all**.

It is reasonable that you don’t want that a process within the container is run as root, but how do you do that or better how do you, the end user, make sure the image provider didn’t set it up that way?

# ROOTLESS - DROP ROOT
Two options are at your disposal; For the users who don’t run Docker as mentioned in the intro: go away, we know that you know of the third way:

-	Setting the user yourself
-	Hoping the image maintainer set another user

**Setting it yourself** is actually very easy to do. Edit your compose and add this to it:
```
services:
  alpine:
    image: "alpine"
    user: "11420:11420"
```

Now docker will execute all processes in the container as **11420:11420** and not as root. Set and done. This only works if you take care of all permissions as well. Remember the process in the container will use this UID/GID, meaning if you mount a share, this UID/GID needs to have access to this share or you will run into simple permission problems.

**Hoping the image maintainer set another user** is a bit harder to check and also you need to trust the maintainer with this. How do you check what user was set in the container image? Easy, a container build file has a directive called ```USER``` which allows the image maintainer to set any user they like. It’s usually the last line in any build file. [Here](https://github.com/11notes/docker-qbittorrent/blob/master/arch.dockerfile#L212) is an example of this practice. For those too lazy to click on a link:

```
# :: EXECUTE
  USER ${APP_UID}:${APP_GID}
  ENTRYPOINT ["/usr/local/bin/qbittorrent"]
  CMD ["--profile=/opt"]
```

Where ```APP_UID``` and ```APP_GID``` are variables defined as 1000 and 1000. This means this image will by default always start as **1000:1000** unless you overwrite this setting with the above mentioned ```user:``` setting in your compose.

**Uh, I have an actual user on my server that is using 1000:1000, so WTF?** Don’t worry about this scenario. Unless you accidentally mount that users home directory or any other directory that user has access to into the container using the same UID/GID, there is no problem in having an actual user with the same UID/GID as a process inside a container. Remember: Containers are isolated namespaces. The can't interact with a process started by a user on the same host.

**I don’t need any of this, I use PUID and PGID thank you.** Well, you do actually. Using PUID/PGID which is not a Docker thing, but a habit that certain image providers perpetuate with their images, still starts the image as root. Yes, root will then drop its privileges down to another user, the one you specified via PUID/PGID, but there is still a process in there running as root. True rootless has no process run as root and doesn’t start as root. Even if root is only used briefly, why open yourself up to that brief risk when you can mitigate it very easily by using rootless images in the first place?

**Bonus: security_opt** can be used to prevent a container image from gaining new privileges by privilege escallation (granting itself mor caps since the image has default caps granted to the root user in the image). This can easily be done by adding this to each of your compose:

```
security_opt:
    - "no-new-privileges=true"
```

# ROOTLESS - SO ANY IMAGE IS ROOTLESS?
Sadly no. Actually **most images use root**. Basically, all images for the most popular images all use root, but why is that? **Convenience**. Using root means you can use ```cap_chown``` remember? This means you can chown folders and fix permission issues before the user of the image even notices that he forgot something. The sad part is you trade convenience for security, as you basically always do. Your node based app is now running as root and has ```cap_net_raw``` even though it does not need that, so why give it that cap in the first place? Many images break when you switch from root to any combination of UID/GID, because the creators of these images did not anticipate you doing so or simply ignored the fact that some users like security more than they like convenience. It is best you use images that are **by default already rootless**, meaning they don’t start as root and they never use root at all. There are some image providers that do by default only provide such images, others provide by default images that run as root but can be run rootless, when using advanced configurations.

That’s another issue we need to mention. If an image can be run rootless in the first place, why is that not the default method of running said image? Why does the end user have to jump through hoops to run the image rootless? We come again to the same answer: **Convenience**. Said image providers who do this, want that their images run on first try, no permission errors or missing caps. Presenting users with advanced compose files to make the image run rootless, is too advanced for the normal user, at least that’s what they think. I don’t think that. **I think every user deserves a rootless image by default** and only if special configurations require elevated privileges, these can be used and highlighted in an advanced way. Not providing rootless images by default basically robs the normal users of their security. Everyone deserves security, not just the greybeards that know how to do it.

# ROOTLESS - CONCLUSION
Use rootless images, prefer rootless images. Do not trade your convenience for security. Even if you are not a greybeard, you deserve secure images. Running rootless images is no hassle, if anything, you learn how Linux file permission work and how you mount a CIFS share with the correct UID/GID. Do not bow down and simply accept that your image runs as root but could be run rootless. Demand rootless images as default, not as an option! Take back your right for security!

I hope you enjoyed this short and brief educational know-how guide. If you are interested in more topics, feel free to ask for them. I will make more such RTFMs in the future.

**Stay safe, stay rootless!**

# ROOTLESS - SOURCES
- [NIST SP 800-123/4.2.3 - Configure Resource Controls Appropriately](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-123.pdf)
- [NIST SP 800-190/3.1.2 - Image configuration defects](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf)