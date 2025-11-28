![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# KNOW-HOW - COMMUNITY EDUCATION
This RTFM is part of a know-how and how-to section for the community to improve or brush up your knowledge. Selfhosting requires some decent understanding of the underlying technologies and their implications. These RTFMs try to educate the community on best practices and best hygiene habits to run each and every selfhosted application as secure and smart as possible. These RTFMs never cover all aspects of every topic, but focus on a small part. Security is not a single solution, but a multitude of solutions and best practices working together. This is a puzzle piece; you have to build the puzzle yourself. You'll find more resources and info’s at the end of the RTFM. Here is the list of current RTFMs:

- 📖 Know-How: Rootless container images, why you should use them all the time if you can! **[>>](https://github.com/11notes/RTFM/blob/main/linux/container/image/rootless.md)**

# DISTROLESS - WHAT IS THAT?
Most on this sub know what a distro is, if not, please read the [wiki article](https://en.wikipedia.org/wiki/Linux_distribution) about it and return back to this guide. So, what shall distroless mean? Another buzzword from the cloud? No. It simply means that no binaries (executable programs) are present that are specifically tied to a Linux distribution. Container images, are nothing more than like a compressed archive, a zip file, containing everything the application within needs to work. The question is, how much junk is in that zip file? A distroless image has **all junk removed** from its image. This means that your zip file contains only what the application needs to run, not one bit more. This does not only make the image several times lighter on your hard drive but also by default more secure. It should be noted that distroless is not the solution to the cyber security problem, but another advanced layer and puzzle piece to complete the whole picture. This know-how does not focus on the other aspects which are equally important to run images as safe and sound as possible. More information and more puzzle pieces will follow in other know-how RTFMs.

**Why does it make it by default more secure?** Well, simply put, if there is less to attack, you have a harder time attacking something. That’s why all ports on your firewall are by default closed. If all ports would be open, someone could find maybe something to exploit and attack you. The same is true for a container image. Why add a shell or curl to your image when your application doesn’t need them to work? There is no benefit in having curl, ls, git, sh, wget and many more in your container image, but there could be a potential downside if any of these have a zero day or known CVE that can be exploited.

Someone might tell you: **"This does not matter!"**, since you run your app and not git. That is not entirely true. The app you run, could have an exploit but not offer much in terms of functionality. For instance, the app can’t make a web request (there is simply no function for this within the app), but the attacker gained access to the container's file system, hence he can now use curl or wget inside your image, to further download more tools to exploit and continue his malicious work. This is especially useful for automated attacks, where known CVEs or science forbid, zero days, are used to exploit your app you are running in an automated way. These are commands that will try to download additional malicious code with tools available which the exploit thinks are present in any image (like curl, wget or sh). If these tools are not available, the attack will already fail and the target will be marked as not vulnerable (to not waste time).

**Nothing will protect you from a targeted attack!** If you are a target of an exploit or hacker group there is basically nothing you can do to protect yourself. You can only mitigate, but not prevent! Don't believe me, believe [the shadow brokers](https://en.wikipedia.org/wiki/The_Shadow_Brokers).

# DISTROLESS - TINY HEROES
Another advantage of a distroless image is its physical size. This is not a very important factor, but a welcome one none the less. Since a distroless image has nothing in it that’s not required to run the app, you save a lot of disk space in addition to reducing your attack surface. Don’t believe me? Well, here is an infamous example:

| **image** | **size on disk** | **init default as** | **[distroless](https://github.com/11notes/RTFM/blob/main/linux/container/image/distroless.md)** | supported architectures
| ---: | ---: | :---: | :---: | :---: |
| 11notes/qbittorrent | 27MB | 1000:1000 | ✅ | amd64, arm64, armv7 |
| home-operations/qbittorrent | 111MB | 65534:65533 | ❌ | amd64, arm64 |
| hotio/qbittorrent | 159MB | 0:0 | ❌ | amd64, arm64 |
| qbittorrentofficial/qbittorrent-nox | 167MB | 0:0 | ❌ | 386, amd64, arm64, armv6, armv7, riscv64 |
| linuxserver/qbittorrent | 503MB | 0:0 | ❌ | amd64, arm64 |

There are two important take aways from this table. First is the **size on disk**. Images are compressed when you download them, but will then be uncompressed on your container host. That’s the actual image size, not the size while it is still compressed on the registry. Second, the space savings and also download, unpacking savings are enormous. Up to a factor of multiples enormous, without any drawbacks or cutbacks. Projects like [eStargz](https://github.com/containerd/stargz-snapshotter) try to solve the rampant container image growth by lazy loading images during download, instead of focusing on creating small images in the first place. The solution is distroless, not lazy loading.

Somene might yell at you: **"Size of an image doesn’t matter!"**, since storage is cheap, and why bother saving a few hundred MB in image size? Let’s not forget that the size of the image is an additional benefit, not the only benefit. The idea is still to have less binaries and libraries in the image that could be exploited. It doesn’t matter how cheap storage is, if you run an image that is full of unpatched, unmaintained binaries that you actually don’t need, you open yourself up to additional security risks for no real reasons. **Do not confuse distroless with just image size!**.

# DISTROLESS - HOW CAN I USE IT?
That’s the easiest part. Simply find a distroless image for the application you need. There aren’t many distroless image providers available sadly, because creating a distroless image is a lot more work for the provider than it is for you to use it. You will basically never get a distroless image from the actual developer of the app. They ship their app often run as root and with a distro like Debian or Alpine. This is done for easy adoption of their app, but leaves you with a poor image in terms of security.

So, what can you do? Simply request the image in question from the provider you prefer. The more demand there is for distroless images, the more will hopefully exist. I myself provide many distroless images for this community. If you are interested you can check them out yourself.

# DISTROLESS - I GOT NO SHELL, WHAT NOW?
Since distroless containers have no shell, you can’t ```docker exec -ti``` into them. Instead, enter the world of [nsenter](https://man7.org/linux/man-pages/man1/nsenter.1.html). A Linux command that lets you enter any namespace of any process and lets you execute binaries from the host within that namespace. Here is an example command from my own educational RTFM:

```
nsenter -t $(docker inspect -f '{{.State.Pid}}' adguard-server-1) -n netstat -tulpn
```

This will execute netstat attached to the defined PID *(-t)* in the namespace network *(-n)*, even though the image does not have netstat installed. Like this you can still debug your images like you would if they would have a shell, just safer and more elegant. You have also the added benefit that you can execute any binary from the host, so you don’ t need to install debug tools into the image itself. Of course, to use nsenter, you must have the correct privileges. If you use a rootless container runtime, make sure you have set the correct permissions for the user you are using nsenter with.

# DISTROLESS - I USE PODMAN, SO NO THANK YOU!
Distroless images are useful regardless what container runtime you use. A slimmed down attack surface helps everyone, even if your images are not executed as root and use a UID/GID mapping that is safer. Not running as root does not mean an exploited image can’t be used to attack other images or even the host. The less there is to attack, the better!

# DISTROLESS - LIMITATIONS
In a perfect world, every app could be run as distroless image, sadly that’s not the case. The reason for that is simple: Some apps require external libraries to be loaded at runtime, dynamically. This makes it impossible to convert them to a distroless image, unless the developer of the app would change their code to not dynamically load additional content at runtime. What are common signs you can’t request a distroless image from an app?

-	App is based on Python
-	App is based on node/deno with dynamic loaded libraries
-	App is based on .NET core with inline Assembly calls

# DISTROLESS - CONCLUSION
The benefits are many, the downsides only a few and are not tied to actual distroless images but apps that can’t be converted to distroless. This sounds like one of these things that is too good to be true, and it somehow is, otherwise everyone would create and use them. I hope this RTFM could educate and inform you more what is possible and what developers actually could do. Why it is not done that way as the best practice and normal way, you have to figure out for yourself. If you have further questions, feel free to ask anything you did not understand or if you need more information about some aspect. 

I hope you enjoyed this short and brief educational know-how guide. If you are interested in more topics, feel free to ask for them. I will make more such RTFMs in the future.

**Stay safe, stay distroless!**

# DISTROLESS - SOURCES
- [NIST SP 800-123/4.2.1 - Remove or Disable Unnecessary Services, Applications, and Network Protocols](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-123.pdf)
- [Docker Docs - Don't install unnecessary packages](https://docs.docker.com/build/building/best-practices/#dont-install-unnecessary-packages)
- [Docker Blog - Is Your Container Image Really Distroless?](https://www.docker.com/blog/is-your-container-image-really-distroless/)