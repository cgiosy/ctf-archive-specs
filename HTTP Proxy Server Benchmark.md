# HTTP Proxy Server Benchmarks

`h2load`를 사용해 리버스 프록시 서버로서의 [H2O](https://github.com/h2o/h2o)와 [Nginx](https://www.nginx.com/)를 벤치마크 및 비교해 보았다. HTTP/2와 HTTP/3 각각에 대해 테스트하였고, 벤치마크 명령어를 다섯 번 실행하여 시간이 가장 적게 소모된 경우를 뽑았다.

## Configurations

### H2O

2022년 1월 8일 기준 [최신 커밋](https://github.com/h2o/h2o/commit/417923e4c031cd2b69ce58652c6b9f348a336392)을 빌드하였으며, [ECDH Curves](https://github.com/h2o/h2o/pull/1793) 관련 PR을 적용했다.

- `h2o.conf`

```text
send-server-name: OFF
access-log: /app/log/h2o-access.log
error-log: /app/log/h2o-error.log
error-log.emit-request-errors: ON

hosts:
  "localhost":
    listen: &www_ssl_listen
      port: 443
      ssl:
        certificate-file: /certs/localhost.pem
        key-file: /certs/localhost.key
        min-version: TLSv1.2
        ecdh-curves: X25519:P-521:P-384
        cipher-suite: ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256
        cipher-suite-tls1.3: [TLS_AES_256_GCM_SHA384, TLS_AES_128_GCM_SHA256, TLS_CHACHA20_POLY1305_SHA256]
    listen:
      type: quic
      <<: *www_ssl_listen
    compress:
      br: 6
    file.send-compressed: ON
    paths:
      "/":
        proxy.reverse.url: "http://localhost:3000/"
        proxy.timeout.first_byte: 600000
```

### Nginx

2022년 1월 8일 기준 3개월 전에 갱신된 [ranadeeppolavarapu/nginx-http3](https://hub.docker.com/r/ranadeeppolavarapu/nginx-http3)를 사용하였다. 버전은 1.19.5라는 것 같다.

- `nginx.conf`

```text
user                 nginx;
pid                  /var/run/nginx.pid;
worker_processes     auto;
worker_rlimit_nofile 65535;

events {
    multi_accept       on;
    worker_connections 65535;
}

http {
    upstream proxy {
        server 0.0.0.0:3000;
    }

    charset                utf-8;
    sendfile               on;
    tcp_nopush             on;
    tcp_nodelay            on;
    server_tokens          off;
    log_not_found          off;
    types_hash_max_size    2048;
    types_hash_bucket_size 64;
    client_max_body_size   16M;

    # MIME
    include                mime.types;
    default_type           application/octet-stream;

    # Logging
    access_log             /var/log/nginx/access.log;
    error_log              /var/log/nginx/error.log warn;

    # SSL
    ssl_session_timeout    1d;
    ssl_session_cache      shared:SSL:10m;
    ssl_session_tickets    off;

    # Mozilla Modern configuration
    ssl_protocols          TLSv1.3;

    # OCSP Stapling
    ssl_stapling           on;
    ssl_stapling_verify    on;
    resolver               1.1.1.1 1.0.0.1 valid=60s;
    resolver_timeout       2s;

    # Connection header for WebSocket reverse proxy
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ""      close;
    }

    map $remote_addr $proxy_forwarded_elem {

        # IPv4 addresses can be sent as-is
        ~^[0-9.]+$        "for=$remote_addr";

        # IPv6 addresses need to be bracketed and quoted
        ~^[0-9A-Fa-f:.]+$ "for=\"[$remote_addr]\"";

        # Unix domain socket names cannot be represented in RFC 7239 syntax
        default           "for=unknown";
    }

    server {
        listen 443 quic reuseport;
        listen 443 ssl http2;
        server_name localhost;

        http2_push_preload on;

        brotli_static on;
        brotli on;
        brotli_types text/plain text/css application/json application/javascript application/x-javascript text/javascript;
        brotli_comp_level 6;

        ssl_protocols TLSv1.2 TLSv1.3;

        ssl_certificate /etc/ssl/localhost.pem;
        ssl_certificate_key /etc/ssl/private/localhost.key;

        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 5m;
        ssl_session_tickets off;

        # Enable TLSv1.3's 0-RTT. Use $ssl_early_data when reverse proxying to
        # prevent replay attacks.
        #
        # @see: http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_early_data
        ssl_early_data on;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        add_header alt-svc 'h3-29=":443"; ma=86400, h3=":443"; ma=86400';
        # Debug 0-RTT.
        add_header X-Early-Data $tls1_3_early_data;

        add_header x-frame-options "deny";
        add_header Strict-Transport-Security "max-age=31536000" always;

        location / {
          proxy_pass http://proxy;
        }
    }

    map $ssl_early_data $tls1_3_early_data {
        "~." $ssl_early_data;
        default "";
    }
}
```

### Ubuntu

잘 알려진 몇 가지 간단한 트윅을 추가했다. [참고](https://blog.cloudflare.com/http-2-prioritization-with-nginx/)

- `/etc/sysctl.conf`

```text
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_notsent_lowat = 16384
```

### Web Server

Go 언어의 [Fiber 라이브러리](https://github.com/gofiber/fiber)를 사용해 1000B 길이의 단순 문자열을 출력하는 프로그램을 썼다. 문자열은 적당한 랜덤 문자열을 하드코딩했는데, 벤치마크에 더 적절한 방법이 있는지 궁금하다.

```go
package main

import "log"
import "github.com/gofiber/fiber/v2"

func main() {
    app := fiber.New()

    app.Get("/test", func(c *fiber.Ctx) error {
        return c.SendString("QHo>'r3ro;nX8[-)V`=Pz5._go4K\\MW! -Q;sc%F9,=\"oTc)r+6;]e#{e6CcViDy#5oS+L@Vbxha53+Wa.t=#hg>$!+hh!E0kg'Qpl@\"iDT{v\\)D]W!|OFv)}$$F/xvQf2#g_9BGlazCzP3W+?/7BNwGj{HG<Rk(?n../NSJF.xlA>TvGCZj7WkDjC,Q;w5`VTG>?Ntq})\\V0Os3m*l@vEy%K|#H5fs+Q5+-jf?*h|T@Bae;ubH'5bHwh`Cqh+h7\"DdkBIqU88AXtc/7SG%a<4`hwOOJI=Vzx(]0ESc0%1Y+6 ax &'PDj(Rfz\\,7!HuA(y'dFpm`bz6]n$^f\\OC^p\\,D}{Veh3IV(ZW`+n%`yAk8U_aElPwv5BI+NSFxJ>=C}go}[!f#{+6rbOrL^8:;NsBZu<zy9)de<JL%%\\Q|R,TaQDfo7K)q\"*'bn>iPLcwiXh7J[iim<HoyG[gkxJ:{|btsL<3knwz<aiE5N4J26\"PLS:C-)KUx>VzrBH(M8myJvKX{B#H]smsimNx`#)8X \"A 9PzbRxqDh1=+>24Yvk=v5q&d{s6D}L,I ^+hIDmow?P IQ^u]0C)c6goZe1FMbHtsWc_{\"tVt6<kn|x%(,#X=g0MRzo6MByN#0jS>U$^pX+4yU[c.WuC%sw90ppaWHP_&:_ZG!ZviYml7'02},3x$5iIT[%4M3$M+VG+fy0j}*{a+K_-@ P9;:<X`+zMlQ(3\\lX d}puIqVoBfATmi482e//k0n{z4]n``Bl/Sd9l{_T8ONy^9>e#<&`{N5vJO-[Rvn vYe`r%4WAcEtwKG5;r9>sDXZLI]YT6y_Ke2xz+\\@_2X>M%04 @en^s8^0&Rwc7TD_->d091j_cS:3hEX6)6SS;&apmo(nn+zRw0TA*>1gFr)u@sfNV=BR\\xF0BF2}9c6VO&M:%<fa\\#0u-ds#yq=Hp6zS&W3|\"T,IXuP2<lJt/Lp&%@mc@p\"3&Gd!c:Y)XB|rmKMBR+uW'E")
    })

    log.Fatal(app.Listen(":3000"))
}
```

## HTTP/2 (100 connections)

Command: `./h2load -t 2 -m 10 -c 100 -n 100000 https://localhost/test`

### H2O

```text
finished in 1.88s, 53256.45 req/s, 52.22MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 98.05MB (102812500) total, 985.16KB (1008800) headers (space savings 89.71%), 95.37MB (100000000) data
                     min         max         mean         sd        +/- sd
time for request:      218us    182.66ms     15.84ms      9.54ms    89.99%
time for connect:     5.01ms    165.09ms     60.79ms     37.48ms    70.00%
time to 1st byte:    21.02ms    192.11ms    136.26ms     34.47ms    69.00%
req/s           :     532.96      752.54      614.27       77.88    55.00%
```

### Nginx

```text
finished in 7.29s, 13717.50 req/s, 15.27MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 111.30MB (116706600) total, 14.21MB (14900000) headers (space savings 31.96%), 95.37MB (100000000) data
                     min         max         mean         sd        +/- sd
time for request:     2.18ms    415.23ms     61.96ms     26.88ms    86.04%
time for connect:     6.38ms    388.22ms    143.27ms    101.13ms    65.00%
time to 1st byte:    55.66ms    480.30ms    341.13ms    122.16ms    83.00%
req/s           :     137.45      727.51      194.18      157.12    92.00%
```

## HTTP/2 (1000 connections)

Command: `./h2load -t 2 -m 10 -c 1000 -n 100000 https://localhost/test`

### H2O

```text
finished in 32.61s, 3066.87 req/s, 3.01MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 98.14MB (102902322) total, 1.02MB (1065322) headers (space savings 89.13%), 95.37MB (100000000) data
                     min         max         mean         sd        +/- sd
time for request:       69us      30.93s       1.57s       5.51s    92.39%
time for connect:    53.18ms       1.67s    800.99ms    461.85ms    56.70%
time to 1st byte:   282.24ms      29.50s       6.04s       8.66s    81.50%
req/s           :       3.07       59.92       13.32       17.00    85.40%
```

### Nginx

```text
finished in 33.81s, 2957.42 req/s, 3.29MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 111.34MB (116749000) total, 14.21MB (14900000) headers (space savings 31.96%), 95.37MB (100000000) data
                     min         max         mean         sd        +/- sd
time for request:      200us      31.86s       1.65s       3.45s    95.34%
time for connect:    66.43ms       2.04s    908.46ms    468.91ms    62.10%
time to 1st byte:      1.65s      33.27s       7.26s       7.12s    78.40%
req/s           :       2.96       10.24        6.34        1.94    46.00%
```

## HTTP/3 (100 connections)

Command: `./h2load --npn-list h3 -t 2 -m 10 -c 100 -n 100000 https://localhost/test`

### H2O

```text
finished in 2.72s, 36792.61 req/s, 37.12MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 100.90MB (105800500) total, 5.05MB (5300000) headers (space savings 45.92%), 95.37MB (100000000) data
UDP datagram: 23465 sent, 102023 received
                     min         max         mean         sd        +/- sd
time for request:      349us    156.32ms     26.13ms      6.28ms    98.23%
time for connect:     4.79ms    144.78ms     58.79ms     32.01ms    69.00%
time to 1st byte:   103.59ms    167.34ms    134.65ms     17.36ms    66.00%
req/s           :     369.13      394.19      374.30        3.82    80.00%
```

### Nginx

```text
finished in 6.32s, 15811.57 req/s, 16.94MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 107.11MB (112308809) total, 10.97MB (11500000) headers (space savings 51.68%), 95.37MB (100000000) data
UDP datagram: 37347 sent, 110482 received
                     min         max         mean         sd        +/- sd
time for request:      917us    158.01ms     57.60ms     21.09ms    65.72%
time for connect:    98.08ms    201.08ms    154.52ms     25.34ms    70.00%
time to 1st byte:   150.34ms    294.50ms    233.62ms     36.59ms    73.00%
req/s           :     158.17      192.97      169.58       11.52    80.00%
```

## HTTP/3 (1000 connections)

Command: `./h2load --npn-list h3 -t 2 -m 10 -c 1000 -n 100000 https://localhost/test`

### H2O

```text
finished in 4.52s, 22142.04 req/s, 22.34MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 100.90MB (105805000) total, 5.05MB (5300000) headers (space savings 45.92%), 95.37MB (100000000) data
UDP datagram: 25928 sent, 104305 received
                     min         max         mean         sd        +/- sd
time for request:      179us       3.10s     92.30ms    191.70ms    94.60%
time for connect:    69.01ms       3.32s       1.34s       1.13s    62.40%
time to 1st byte:   431.29ms       4.19s       1.73s       1.10s    44.20%
req/s           :      22.65      112.64       54.87       26.70    51.90%
```

### Nginx

```text
finished in 24.36s, 4105.82 req/s, 4.40MB/s
requests: 100000 total, 100000 started, 100000 done, 100000 succeeded, 0 failed, 0 errored, 0 timeout
status codes: 100000 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 107.18MB (112388030) total, 10.97MB (11500000) headers (space savings 51.68%), 95.37MB (100000000) data
UDP datagram: 27441 sent, 115823 received
                     min         max         mean         sd        +/- sd
time for request:      370us      17.90s       1.39s       2.77s    94.08%
time for connect:   552.44ms       3.48s       1.67s       1.06s    63.90%
time to 1st byte:   634.64ms      18.49s       8.08s       6.71s    52.60%
req/s           :       4.11       13.38        7.24        2.60    48.80%
```
