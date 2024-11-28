# minio
```code
wget https://dl.min.io/server/minio/release/linux-amd64/minio

sudo chmod +x minio

MINIO_ROOT_USER=admin MINIO_ROOT_PASSWORD=12345678 \
./minio server --address "0.0.0.0:6000" \
               --console-address "0.0.0.0:6001" \
               http://minio-node{1...3}/data{1...2}
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
