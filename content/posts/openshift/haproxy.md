---
title: "Load balancer config"
date: 2022-12-13
---
  
  
  #### Default haproxy config

  * Inter 1s (The "inter" parameter sets the interval between two consecutive health checks  to <delay> milliseconds.)

  * Fall 2 (The "fall" parameter states that a server will be considered as dead after  <count> consecutive unsuccessful health checks.)

  * Rise 3 (The "rise" parameter states that a server will be considered as operational  after <count> consecutive successful health checks.)

  * HttpCheck GET /readyz HTTP/1.0


```
global
  stats socket /var/lib/haproxy/run/haproxy.sock  mode 600 level admin expose-fd listeners
defaults
  maxconn 20000
  mode    tcp
  log     /var/run/haproxy/haproxy-log.sock local0
  option  dontlognull
  retries 3
  timeout http-request 30s
  timeout queue        1m
  timeout connect      10s
  timeout client       86400s
  timeout server       86400s
  timeout tunnel       86400s
frontend  main
  bind :::9445 v4v6
  default_backend masters
listen health_check_http_url
  bind :::9444 v4v6
  mode http
  monitor-uri /haproxy_ready
  option dontlognull
listen stats
  bind localhost:29445
  mode http
  stats enable
  stats hide-version
  stats uri /haproxy_stats
  stats refresh 30s
  stats auth Username:Password
backend masters
   option  httpchk GET /readyz HTTP/1.0
   option  log-health-checks
   balance roundrobin
   server master-0 10.10.0.209:6443 weight 1 verify none check check-ssl inter 1s fall 2 rise 3
   server master-2 10.10.0.228:6443 weight 1 verify none check check-ssl inter 1s fall 2 rise 3
   server master-1 10.10.0.250:6443 weight 1 verify none check check-ssl inter 1s fall 2 rise 3

```