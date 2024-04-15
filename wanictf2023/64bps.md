# 64bps upsolve

Dockerfile, flag.txt, nginx.confのみが渡されます。

*Dockerfile*
```dockerfile
FROM nginx:1.23.3-alpine-slim

COPY nginx.conf /etc/nginx/nginx.conf
COPY flag.txt /usr/share/nginx/html/flag.txt

RUN cd /usr/share/nginx/html && \
    dd if=/dev/random of=2gb.txt bs=1M count=2048 && \
    cat flag.txt >> 2gb.txt && \
    rm flag.txt
```

*nginx.conf*
```txt
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    keepalive_timeout  65;
    gzip               off;
    limit_rate         8; # 8 bytes/s = 64 bps

    server {
        listen       80;
        listen  [::]:80;
        server_name  localhost;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }
}
```

2gb.txtを読み取れば良さそうですが、flagをちょくで狙わないといけなさそう。

curlのrangeを使えば良さそう。

```bash
 curl http://localhost/2gb.txt --range 2147483648-
```
