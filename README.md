lIlIIll1O01o

# minio
```code
wget https://dl.min.io/server/minio/release/linux-amd64/minio

sudo chmod +x minio

sudo vi /etc/hosts

MINIO_CI_CD=true MINIO_ROOT_USER=admin MINIO_ROOT_PASSWORD=12345678 \
./minio server --address "0.0.0.0:6000" \
               --console-address "0.0.0.0:6001" \
               http://minio-node{1...3}/data{1...2}
```
# autorestic
## https://autorestic.vercel.app/docker
```code
docker run --rm -it \
       -v /home/user/.config/rclone/rclone.conf:/root/.config/rclone/rclone.conf:ro \
       -v $PWD/autorestic:/root/.config/autorestic
       cupcakearmy/autorestic /bin/bash

autorestic check
autorestic backup -a
autorestic exec -av -- snapshots
autorestic restore -l home \
                --from nas \
                --to ./restore
```

# syncthing
```code
version: '3'
services:
  syncthing: #唯一
    container_name: syncthing-st  #唯一
    image: syncthing/syncthing:latest
    hostname: syncthing #optional
    healthcheck:
      test: ['CMD','true'] #disable the healthcheck 强制测试通过
    environment:
      - STGUIADDRESS=0.0.0.0:8384 #docker只能使用这方式修改GUI端口
      - PUID=0  #不定义则默认1000用户，可能无法读取其他docker目录
      - PGID=0
    volumes:
      - ./var/syncthing:/var/syncthing
      - nas:/abc
    restart: unless-stopped
    ports:
      - 8384:8384
      - 22000:22000/tcp
      - 22000:22000/udp
      - 21027:21027/udp
    #network_mode: "host"    #主机模式.无需预先映射端口
volumes:
  nas:
    driver: rclone
    driver_opts:
      remote: 'minio:nas'
      allow_other: 'true'
      vfs_cache_mode: full
      poll_interval: 0
```

# RCLONE
## https://post.smzdm.com/p/am3wmk8v/
## https://rclone.org/downloads/
## rclone config
### .config/rclone/rclone.conf
```code
[nas]
type = s3
provider = Minio
env_auth = false
access_key_id = admin
secret_access_key = 12345678
endpoint = http://10.17.1.22:7000

rclone copy .config/rclone/rclone.conf nas:abc
rclone sync nas:abc ./abc

touch abc/a.txt
rclone sync ./abc minio:abc
```
# Volume
## https://valetzx.github.io/p/13d3.html
```code
version: '3'
services:
  heimdall:
    image: busybox:1.32
    command: ping 8.8.8.8
    volumes: [configdata:/config]
volumes:
  configdata:
    driver: rclone
    driver_opts:
      remote: 'minio:conf'
      allow_other: 'true'
      vfs_cache_mode: full
      poll_interval: 0
```

# Docker rcd 
## https://rclone.org/commands/rclone_rcd/
```code
docker run --rm -it --network=host -v \
       $PWD/abc/rclone.conf:/config/rclone/rclone.conf \
       rclone/rclone:latest \
       rcd --rc-baseurl "/rclone/" \
       --rc-addr=0.0.0.0:5555 \
       --rc-user uid --rc-pass pwd \
       --rc-web-gui \
       --rc-serve
```
http://10.17.1.26:5555/rclone/*
rclone rc --url http://localhost:5555/rclone/ --user uid --pass pwd job/list

## webapi
```code
!curl -X POST 'http://uid:pwd@10.17.1.26:5555/rclone/rc/noop?potato=1&sausage=2'
!curl -H "Content-Type: application/json" -X POST -d '{"potato":2,"sausage":1}' http://uid:pwd@10.17.1.26:5555/rclone/rc/noop
```

# mc
```code
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
mc alias set nas/ http://localhost:7000 admin 12345678

mc cp clear.sh nas/abc/
mc ls nas/abc/
mc rm nas/abc/clear.sh
```
## DataNode SideCar
```code
version: '3'

services:
  minio:
    image: minio/minio
    #restart: always
    environment:
      MINIO_ROOT_USER: admin
      MINIO_ROOT_PASSWORD: 12345678
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

    volumes:
      - ./ec/data1:/data1
      - ./ec/data2:/data2

    command: >
      server --address "0.0.0.0:9000" --console-address "0.0.0.0:9001"
      http://10.161.236.101:9000/data{1...2}
      http://10.161.236.102:9000/data{1...2}

    cap_add:
        - NET_ADMIN
    stdin_open: true
    tty: true
    network_mode: host

  zerotier:
    image: zerotier/zerotier
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun
    volumes:
      #- ./planet:/var/lib/zerotier-one/planet
      # docker cp minio-zerotier-1:/var/lib/zerotier-one ./one
      - ./one/:/var/lib/zerotier-one/
    command:
      - 677a4f5bc655fd0a
    depends_on:
      - minio
    network_mode: "service:minio"

```

## nginx LB
```code
    upstream minio {
        server minio1:9000;
        server minio2:9000;
        server minio3:9000;
        server minio4:9000;
    }

    server {
        listen       9000;
        listen  [::]:9000;
        server_name  localhost;

        # To allow special characters in headers
        ignore_invalid_headers off;
        # Allow any size file to be uploaded.
        # Set to a value such as 1000m; to restrict file size to a specific value
        client_max_body_size 0;
        # To disable buffering
        proxy_buffering off;

        location / {
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_connect_timeout 300;
            # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            chunked_transfer_encoding off;

            proxy_pass http://minio;
        }
    }
```
