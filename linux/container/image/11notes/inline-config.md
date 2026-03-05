![banner](https://github.com/11notes/static/blob/main/img/banner/README.png?raw=true)

# INLINE CONFIGS
*Why bother with files when you can use environment variables?*

# TL;DR - FOR BEGINNERS
Instead of providing your config as an actual file via a mounted volume, you can provide it as an environment variable and even have the benefits of using the environment variables from your host or pass them to your compose, neat!

# SYNOPSIS 📖
Simple config files for an app can be very annoying if they have to be provided via an actual file, that needs to be mounted inside the container to provide the needed configuration. An alternative method, if the image supports it, is to provide the configuration inline, meaning inside the compose itself, not as a file, but as simple text. The image will then convert the content of your variable into the actual config file, instead of mounting it in. This can have multiple benefits vs. traditional file mounting, including the ability to add dynamic information via variables from compose itself (host environment or environment passed to compose).

## SET INLINE CONFIG
How to provide a proper inline config? Check out this example:

```yaml
name: "dns"

x-lockdown: &lockdown
  # prevents write access to the image itself
  read_only: true
  # prevents any process within the container to gain more privileges
  security_opt:
    - "no-new-privileges=true"

services:
  adguard:
    image: "11notes/adguard:0.107.72"
    <<: *lockdown
    environment:
      TZ: "Europe/Zurich"
      ADGUARD_CONFIG: |-
        dns:
          bind_hosts:
            - 0.0.0.0
          ratelimit: 0
          aaaa_disabled: true
          all_servers: true
          upstream_dns:
            - 9.9.9.10
          bootstrap_dns:
            - 9.9.9.10
          cache_size: 1073741824
          max_goroutines: 1024
          hostsfile_enabled: false
        dhcp:
          enabled: false
        http:
          address: 0.0.0.0:3000
          session_ttl: 720h
        querylog:
          enabled: true
          file_enabled: true
          size_memory: 8
          dir_path: /adguard/var
        users:
          - name: admin
            password: ${MY_OWN_PASSWORD}
        auth_attempts: 3
        block_auth_min: 60
        filtering:
          blocking_mode: nxdomain
          cache_time: 1440
          filters_update_interval: 24
          blocked_response_ttl: 3660
          protection_enabled: true
        clients:
          persistent:
            - name: dnslookup
              ids:
                - 127.0.0.1
              ignore_querylog: true
              ignore_statistics: true
        log:
          enabled: true
          file: ""
          max_backups: 0
          max_size: 100
          max_age: 3
          compress: false
          local_time: true
          verbose: false
        schema_version: 29
    volumes:
      - "adguard.etc:/adguard/etc"
      - "adguard.var:/adguard/var"
    tmpfs:
      # needed for read-only
      - "/adguard/run:uid=1000,gid=1000"
    ports:
      - "53:53/udp"
      - "53:53/tcp"
      - "3000:3000/tcp"
    networks:
      frontend:
    sysctls:
      # allow rootless container to access ports < 1024
      net.ipv4.ip_unprivileged_port_start: 53
    restart: "always"

volumes:
  adguard.etc:
  adguard.var:

networks:
  frontend:
```

As you can see, you can even use ```${MY_OWN_PASSWORD}``` to set dynamic variables from your host or what you have passed to your compose/k8s to set the password directly inside the config if need be.

# CONCLUSION
If the image supports it, inline configurations via environment variables are a better and more dynamic way than static files that need to be edited.

**File are so outdated, am I right?**