![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# WHY BUILD CUSTOM IMAGES AND NOT JUST UPDATE THE OFFICIAL ONES?

*People are quick to tell other developers to just create a PR (pull request) like on github, but what are the actual implications of this and why is it rarely done?*

# SYNOPSIS 📖

People who don’t write code or barely any code often quickly point out that a simple PR to the original image would have been the better approach than creating another custom image for the same app. They see the redundancy and automatically think this is bad. They do not see that the same problem can be approach from different angles. Just like you can grill your favourite cut in many different ways, so can you create container images in many different flavoures. That’s a good thing, not a bad thing. It gives people a choice. They can either use the original image, provided by the developer or they can use another version of this image with some twists and benefits, stuff the developer has not thought about. If you care about these benefits is completely up to you and not open for debate.

# BUT WHY NOT UPDATE THE ORIGINAL IMAGE WITH THESE CHANGES?

That’s very simple: **It requires a lot of effort.** Most changes to a container image to repackage an app as a custom image are severe. This means they can’t just be pushed to the original image without disrupting how the original image is created. Many things in the original image creation process (CI/CD[^1]) would have to be changed and these many, many changes are seldom seen as a benefit. Think the debate of [rootless](https://github.com/11notes/RTFM/blob/main/linux/container/image/rootless.md) and [distroless](https://github.com/11notes/RTFM/blob/main/linux/container/image/distroless.md). Many developers don’t see a problem in running their app as root, they even see it as a benefit, so that every user can use it, without worrying about file permissions or kernel restrictions. Spending many hours to create a custom CI/CD process to make their original container image better, only for these changes to be rejected is a waste of everyone’s time.

The next issue with this simple expression *"just create a PR, duh!"* is politics. Many projects do not allow changes from outsiders, by default. Others require you to sign off on your contribution, meaning you will not get credited for all the work you did. Bringing politics into programming has almost never benefits. Structure is important, but why fight political battles in why rootless is better when you can instead just create your rootless image yourself, without everyone fighting you along the way? People and especially developers do not like change. An outsider telling them what they do is wrong will automatically generate pushback and nothing else. Time spent fighting those pushbacks can be better used to develop a better container image.

# PERSONAL EXPERIENCE

I’ve had my fair share of all these political interactions. I’ve had developers denying my PR because the indent was wrong or because I added comments in a different style that they wanted. Some started the old debate of Alpine vs. Debian instead of focusing a lightweight image for their users. Code was never accepted as it was. It was always scrutinized for reasons that are either pure vanity or pride. These hundreds of hours wasted are proof that creating a simple PR request to make the original image better, is a lie.

# CONCULUSION

The next time you see someone pointing out to *"just create a PR"* think about how hard this actually is and what a can of worms in terms of politics this opens. All this just to switch a container image from root to rootless.

**If you want things done right, you have to do it yourself.**

[^1]: CI/CD is a software development process, you can read more about it on [Wikipedia](https://en.wikipedia.org/wiki/CI/CD)