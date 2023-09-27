### Searching for the HTTP Teapot

It all started by making an accidental HTTP GET request with a relatively long path to the biggest social network (namely vk.com) in my country. I'm not sure how I ended up like this; probably was just pressing on a single key in frustration.

You all know about HTTP 418 response code which was proposed on *April first* at 1998:

> 2.3.2 418 I'm a teapot
>
>   Any attempt to brew coffee with a teapot should result in the error
>   code "418 I'm a teapot". The resulting entity body MAY be short and
>   stout.

(You can see the entire RFC2324 [here](https://www.rfc-editor.org/rfc/rfc2324#section-2.3.2))

Anyway, I was kind of surprised there is at least one **huge** production serving HTTP 418 -- despite that's obviously an Easter egg.

I started thinking if there are more popular services that return 418 in the case of large URIs. Soon, I generalized the problem: what are the most common response codes services use for this kind of request? Are there unusual/unexpected responses? To figure it out, wel'll have to make some HTTP requests.

It's obvious that most, if not all, of the requests will be handled by CDNs and reverse-proxies. For instance, nginx responds with `HTTP 414 URI Too Long` if URI bytesize exceeds the `large_client_header_buffers` directive value (which defaults is 8KB).

It actually makes sense to collect statistics from the most popular sites. There are some lists available on the Internet, let's grab one from
[CloudFlare Radar](https://radar.cloudflare.com/domains):

```bash
# for some reasons there are 10001 sites
$ cat sites.csv | wc -l
10001

$ cat sites.csv | grep yandex
yandex-bank.net
yandex.by
yandex.com
yandex.com.tr
yandex.kz
yandex.md
yandex.net
yandex.ru
yandexadexchange.net
yandexcloud.net
yandexmetrica.com
```

To figure it out we're going to use a simple script:

```bash
#!/usr/bin/env bash

set -euo pipefail

echo "$PATH_LENGTH" > /dev/null

PATH_LENGTH=$(cat /dev/random | tr -dc '[:alpha:]' | fold -w ${1:-$PATH_LENGTH} | head -n 1)

cat sites.csv | parallel -j 8 \
 "curl \
   --connect-timeout 15 \
   --max-time 40 \
   --silent \
   --insecure \
   --output /dev/null \
   -w '%{http_code}\n' \
   -L \
   {}/$PATH_LENGTH"
```

<details>
  <summary>Explanation</summary>

  <p>After some basic shebang and shell options related setup we generate URI path via /dev/urandom pseudorandom source.
  We dont have to use separete paths for each requested domain so we'll single pregenerated one.</p>

  <p>Then we make requests using curl in parallel (via GNU parallel).
  We explicitly set output format (`-w '%{http_code}\n'`, HTTP response code only), some timeouts and `--insecure` option to allow HTTP.</p>
</details>

Running with `PATH_LENGTH=1000` yields the following results: (NOTE: it took ~20 minutes on 12 vCPUs and bad network bandwidth)

```bash
$ cat results-1k.csv  | sort | uniq -c
   3072 000
   1283 200
      3 202
     22 204
     95 301
     37 302
      2 307
      1 308
    270 400
     24 401
   1607 403
   3336 404
      3 405
     16 406
      2 409
     13 410
      2 412
      4 414
      5 418
      1 422
      6 429
      3 439
      1 464
      1 475
     66 500
      3 501
     29 502
     32 503
      8 504
      7 520
      6 521
     29 522
      2 523
      1 525
      3 526
      1 529
      4 530
      1 555
```

Woah! We got 38 (including `000` which stands for curl internal error or host unreachablilty) different response codes.

Most common codes are actually quite expected:

```bash
$ cat results-1k.csv | sort | uniq -c | sort -n -r | head -n 16
   3336 404
   3072 000
   1607 403
   1283 200
    270 400
     95 301
     66 500
     37 302
     32 503
     29 522
     29 502
     24 401
     22 204
     16 406
     13 410
      8 504
```

There are also some rare (less than 10) items which are quite OK too: 307, 308, 405, 409, 412, 422, 501 

However, the rest leaves many questinos:

```bash
$ cat results-1k.csv | sort | uniq -c | sort -n -r | tail -n +17
      7 520
      6 521
      6 429
      5 418
      4 530
      4 414
      3 526
      3 439
      3 202
      2 523
      2 409
      1 555
      1 529
      1 525
      1 475
      1 464
      1 422
```

There are some groups:

Non-stadard HTTP codes: (CF stands for CloudFlare)
  - 520 (CF, Web Server Returned an Unknown Error)
  - 521 (CF, Web Server Is Down)
  - 530 (CF, ?)
  - 526 (CF, Invalid SSL Certificate)
  - 439 (?)
  - 523 (CF, Origin Is Unreachable)
  - 555 (?)
  - 529 (?)
  - 525 (CF, SSL Handshake Failed)
  - 475 (?)
  - 464 (?)

Unexpected:
  - 202 (Accepted): seems like there's no reasons to responds with "accepted for future processing"
  - 429 (Too Many Request): actually that was the first and the only request

Funny:
  - 414 (URI Too Long): only 4 responses with *probably* the most suitable code
  - 418 (I'm a Teapot): no comments needed

So, we didn't find any other teapots except the first one. :(
