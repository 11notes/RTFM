![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# HOW-TO CHANGE UID/GID on 11notes container images

*I don’t want to use the default 1000:1000 UID/GID pair and use my own UID/GID how can I do this?*

# TL;DR - FOR BEGINNERS
**You don’t.** If using an image with a fixed UID/GID, all you need to do is to make sure that UID/GID has access to the files it needs, that’s it.

# SYNOPSIS 📖
There is basically absolutely no reason to change the UID/GID to anything else. There is no security risk when multiple images run the exact same UID/GID pair (you do not need to specify a different UID/GID for each container). There is also no risk if you have an actual user with said UID/GID on your system, except that said user can read/write all files with that UID/GID. Which from withing a container is impossible and only works if said user accesses the host.

## CHANGE UID/GID THE CORRECT WAY

All my container images have a parent folder that has the same name as the app inside. All you need to do is to chown said parent folder **once** with the UID/GID you want to use. You can either do this manually, by creating the volumes before starting the images and then chown said parent folder or you can do this directly in your compose:

```yaml
name: "arr"

x-lockdown: &lockdown
  # prevents write access to the image itself
  read_only: true
  # prevents any process within the container to gain more privileges
  security_opt:
    - "no-new-privileges=true"

services:
  # This image will mount the volume and then simply execute chown on the parent folder for the UID/GID specified.
  chown:
    image: "alpine"
    entrypoint: ["/bin/ash", "-c"]
    command:
      - |
        chown -R ${UID}:${GID} /sonarr
    volumes:
      - "sonarr.etc:/sonarr/etc"

  sonarr:
    depends_on:
      chown:
        condition: service_completed_successfully
    image: "11notes/sonarr:4.0.15"
    <<: *lockdown
    user: "${UID}:${GID}"
    environment:
      TZ: "Europe/Zurich"
    volumes:
      - "sonarr.etc:/sonarr/etc"
    tmpfs:
      - "/tmp:uid=${UID},gid=${GID}"
    ports:
      - "3000:8989/tcp"
    networks:
      frontend:
    restart: "always"

volumes:
  sonarr.etc:

networks:
  frontend:
```

```shell
UID=11420 GID=11420 docker compose up -d
docker exec arr-sonarr-1 id
# uid=11420 gid=11420 groups=11420
```

# CONCLUSION

Do not try to change the UID/GID unless you have a really good reason for this. If you do have one, simply use the compose method to change it.

**Never change a running system, unless you really have to**