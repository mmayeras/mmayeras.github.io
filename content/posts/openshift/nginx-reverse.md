---
title: "NGINX reverse"
date: 2023-02-15
---

### Use nginx as reverse proxy in front of multiple clusters
`$ dnf install -y nginx nginx-mod-stream.x86_64`

Add in nginx.conf
`include /etc/nginx/passthrough.conf;`


passthrough.conf
```
stream {

    map $ssl_preread_server_name $internalport {
	hostnames;
        *.apps.sno1.domain     9441;
        *.apps.sno2.domain      9442;
        api.sno1.domain      6441;
        api.sno2.domain      6442;
    }


    upstream sno2_api {
        server 192.168.0.109:6443 max_fails=3 fail_timeout=10s;
    }
    upstream sno2_ingress {
        server 192.168.0.109:443 max_fails=3 fail_timeout=10s;
    }
    upstream sno1_api {
        server 192.168.0.110:6443 max_fails=3 fail_timeout=10s;
    }
    upstream sno1_ingress {
        server 192.168.0.110:6443 max_fails=3 fail_timeout=10s;
    }

log_format basic '$remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time "$upstream_addr" '
                 '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

    access_log /var/log/nginx/access.log basic;
    error_log  /var/log/nginx/error.log;
    server {
        listen                  443;
        ssl_preread             on;
        proxy_connect_timeout   20s;  # max time to connect to pserver
        proxy_timeout           30s;  # max time between successive reads or writes
        proxy_pass              127.0.0.1:$internalport;
    } 
    server {
        listen                  6443;
        ssl_preread             on;
        proxy_connect_timeout   20s;  # max time to connect to pserver
        proxy_timeout           30s;  # max time between successive reads or writes
        proxy_pass              127.0.0.1:$internalport;
    }
    server {
        listen 9441;
        proxy_pass sno1_ingress;
        proxy_next_upstream on;
    }
    server {
        listen 9442;
        proxy_pass sno2_ingress;
        proxy_next_upstream on;
    }
    server {
        listen 6441;
        proxy_pass sno1_api;
        proxy_next_upstream on;
    }
    server {
        listen 6442;
        proxy_pass sno2_api;
        proxy_next_upstream on;
    }
}
```