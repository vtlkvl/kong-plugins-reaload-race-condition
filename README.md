# Synopsis
We run Kong in the DB-less mode in k8s. Our microservices run on spot instances and their IP 
addresses are changing pretty often. This triggers declarative reconfiguration and once every few 
days one of the pods gets stuck on `Could not claim instance_id for` error and becomes completely
unresponsive, i.e. user requests continue coming in, but they fall into a trap of the retry loop and
then eventually fail.

### Environment
1. [Kong 3.3.0 (Ubuntu)](https://hub.docker.com/layers/library/kong/3.3.0-ubuntu/images/sha256-f305f85491b60c6139464ead7fe0b7289d3469a45224f4fada8ba16e909ce8ef?context=explore)
2. [Kong PDK 0.33](https://pypi.org/project/kong-pdk/)
3. Python 3.10.6

### Prerequisites
Running Kong locally will require a pretty standard toolkit:
1. [Docker Desktop](https://www.docker.com/products/docker-desktop/)
2. [KinD](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
3. [Helm](https://helm.sh/docs/intro/install/)

## Test
We will run Kong locally with 10 plugins and simulate 20 concurrent users. During the test we will
scale the service up and down to trigger declarative reconfiguration.
1. Start Kong.
```sh
./kong-local.sh stop start
...
Image: "kong-gateway:1.0.0" with ID "sha256:f3da8db5cb07db5e9cf568519492e15bacd5744f650d54e0910dbe92596fdd47" not yet present on node "kind-control-plane", loading...
Getting updates for unmanaged Helm repositories...
...Successfully got an update from the "https://charts.konghq.com" chart repository
Saving 1 charts
Downloading kong from repo https://charts.konghq.com
Deleting outdated charts
NAME: kong
LAST DEPLOYED: Wed Jul  5 11:06:43 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
Forwarding from 0.0.0.0:8080 -> 8000
```
2. Run `siege` with 50 concurrent users for 10 minutes and dump heap before and after. Repeat it a few times.
```shell
$ siege -c 50 -t 600S -i http://localhost:8080/echo
```
3. Scale `echo` deployment up and down a couple of times every few seconds:
```shell
$ kubectl scale deployment echo --replicas=2
deployment.apps/echo scaled
$ kubectl scale deployment echo --replicas=3
deployment.apps/echo scaled
$ kubectl scale deployment echo --replicas=4
deployment.apps/echo scaled
$ kubectl scale deployment echo --replicas=5
deployment.apps/echo scaled
```
4. Check the logs:
```
$ kubectl logs -n kong -f -c proxy kong-kong-557cffcf5-glh69
2023/07/05 09:31:51 [notice] 1267#0: *308 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, client: 127.0.0.1, server: kong_admin, request: "POST /config?check_hash=1 HTTP/2.0", host: "localhost:8444"
2023/07/05 09:31:51 [notice] 1267#0: *308 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, client: 127.0.0.1, server: kong_admin, request: "POST /config?check_hash=1 HTTP/2.0", host: "localhost:8444"
2023/07/05 09:31:51 [info] 1267#0: *4 [lua] handler.lua:604: declarative reconfigure was started on worker #0, context: ngx.timer
2023/07/05 09:31:51 [info] 1268#0: *154 [lua] handler.lua:604: declarative reconfigure was started on worker #1, context: ngx.timer
2023/07/05 09:31:51 [info] 1268#0: *154 [lua] handler.lua:639: building a new router took 1 ms on worker #1, context: ngx.timer
2023/07/05 09:31:51 [info] 1268#0: *154 [lua] handler.lua:652: building a new plugins iterator took 0 ms on worker #1, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "POST /config?check_hash=1 HTTP/2.0" 201 5412 "-" "Go-http-client/2.0"
2023/07/05 09:31:51 [notice] 1268#0: *154 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:31:51 [notice] 1268#0: *154 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *4 [lua] handler.lua:639: building a new router took 2 ms on worker #0, context: ngx.timer
2023/07/05 09:31:51 [info] 1268#0: *154 [lua] handler.lua:692: declarative reconfigure took 4 ms on worker #1, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *4 [lua] handler.lua:652: building a new plugins iterator took 0 ms on worker #0, context: ngx.timer
2023/07/05 09:31:51 [notice] 1267#0: *4 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:31:51 [notice] 1267#0: *4 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *4 [lua] handler.lua:692: declarative reconfigure took 5 ms on worker #0, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #257 of plugin_9 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #258 of plugin_8 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #259 of plugin_9 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #260 of plugin_7 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #261 of plugin_6 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #262 of plugin_8 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #263 of plugin_5 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] WARN - [17:31:51] rpc: #44165 error: no plugin instance #244, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #264 of plugin_7 started, context: ngx.timer
2023/07/05 09:31:51 [warn] 1267#0: *7106 [kong] mp_rpc.lua:343 [plugin_5] no plugin instance #244, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #265 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #266 of plugin_4 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #267 of plugin_6 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #268 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] WARN - [17:31:51] rpc: #44196 error: no plugin instance #265, context: ngx.timer
2023/07/05 09:31:51 [warn] 1267#0: *7106 [kong] mp_rpc.lua:343 [plugin_5] no plugin instance #265, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #269 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #270 of plugin_3 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #271 of plugin_5 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #272 of plugin_2 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #273 of plugin_10 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #274 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] WARN - [17:31:51] rpc: #44210 error: no plugin instance #251, context: ngx.timer
2023/07/05 09:31:51 [warn] 1267#0: *7107 [kong] mp_rpc.lua:343 [plugin_10] no plugin instance #251, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #275 of plugin_10 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] WARN - [17:31:51] rpc: #44211 error: no plugin instance #269, context: ngx.timer
2023/07/05 09:31:51 [warn] 1267#0: *7106 [kong] mp_rpc.lua:343 [plugin_5] no plugin instance #269, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #276 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #277 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:51 [info] 1267#0: *149 [python:1269] INFO - [17:31:51] instance #278 of plugin_4 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:52 [info] 1267#0: *149 [python:1269] WARN - [17:31:52] rpc: #44239 error: no plugin instance #276, context: ngx.timer
2023/07/05 09:31:52 [warn] 1267#0: *7106 [kong] mp_rpc.lua:343 [plugin_5] no plugin instance #276, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:54 [notice] 1267#0: *308 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, client: 127.0.0.1, server: kong_admin, request: "POST /config?check_hash=1 HTTP/2.0", host: "localhost:8444"
2023/07/05 09:31:54 [notice] 1267#0: *308 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, client: 127.0.0.1, server: kong_admin, request: "POST /config?check_hash=1 HTTP/2.0", host: "localhost:8444"
2023/07/05 09:31:54 [info] 1268#0: *154 [lua] handler.lua:604: declarative reconfigure was started on worker #1, context: ngx.timer
2023/07/05 09:31:54 [info] 1268#0: *154 [lua] handler.lua:639: building a new router took 0 ms on worker #1, context: ngx.timer
2023/07/05 09:31:54 [info] 1268#0: *154 [lua] handler.lua:652: building a new plugins iterator took 0 ms on worker #1, context: ngx.timer
2023/07/05 09:31:54 [notice] 1268#0: *154 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:31:54 [notice] 1268#0: *154 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:31:54 [info] 1268#0: *154 [lua] handler.lua:692: declarative reconfigure took 3 ms on worker #1, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:54 +0000] "POST /config?check_hash=1 HTTP/2.0" 201 3991 "-" "Go-http-client/2.0"
2023/07/05 09:31:54 [info] 1267#0: *4 [lua] handler.lua:604: declarative reconfigure was started on worker #0, context: ngx.timer
2023/07/05 09:31:54 [info] 1267#0: *4 [lua] handler.lua:639: building a new router took 1 ms on worker #0, context: ngx.timer
2023/07/05 09:31:54 [info] 1267#0: *4 [lua] handler.lua:652: building a new plugins iterator took 1 ms on worker #0, context: ngx.timer
2023/07/05 09:31:54 [notice] 1267#0: *4 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:31:54 [notice] 1267#0: *4 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:31:54 [info] 1267#0: *4 [lua] handler.lua:692: declarative reconfigure took 5 ms on worker #0, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:55 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #279 of plugin_1 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #280 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #281 of plugin_10 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #282 of plugin_3 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #283 of plugin_9 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #284 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #285 of plugin_2 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] WARN - [17:31:55] rpc: #44282 error: no plugin instance #280, context: ngx.timer
2023/07/05 09:31:55 [warn] 1267#0: *7106 [kong] mp_rpc.lua:343 [plugin_5] no plugin instance #280, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #286 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #287 of plugin_8 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #288 of plugin_5 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:55 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:55 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] WARN - [17:31:55] rpc: #44300 error: no plugin instance #286, context: ngx.timer
2023/07/05 09:31:55 [warn] 1267#0: *7106 [kong] mp_rpc.lua:343 [plugin_5] no plugin instance #286, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #289 of plugin_5 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #290 of plugin_10 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #291 of plugin_4 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #292 of plugin_9 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] WARN - [17:31:55] rpc: #44309 error: no plugin instance #266, context: ngx.timer
2023/07/05 09:31:55 [warn] 1267#0: *7130 [kong] mp_rpc.lua:343 [plugin_4] no plugin instance #266, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] WARN - [17:31:55] rpc: #44308 error: no plugin instance #266, context: ngx.timer
2023/07/05 09:31:55 [warn] 1267#0: *7129 [kong] mp_rpc.lua:343 [plugin_4] no plugin instance #266, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #293 of plugin_4 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #294 of plugin_4 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #295 of plugin_4 started, context: ngx.timer
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] WARN - [17:31:55] rpc: #44313 error: no plugin instance #266, context: ngx.timer
2023/07/05 09:31:55 [warn] 1267#0: *7116 [kong] mp_rpc.lua:343 [plugin_4] no plugin instance #266, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:31:55 [info] 1267#0: *149 [python:1269] INFO - [17:31:55] instance #296 of plugin_4 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:31:55 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:31:55 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:31:56 [info] 1267#0: *149 [python:1269] WARN - [17:31:56] rpc: #44320 error: no plugin instance #291, context: ngx.timer
2023/07/05 09:31:56 [warn] 1267#0: *7106 [kong] mp_rpc.lua:343 [plugin_4] no plugin instance #291, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:31:57 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:00 [info] 1267#0: *149 [python:1269] INFO - [17:32:00] instance #297 of plugin_1 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:00 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
127.0.0.1 - - [05/Jul/2023:09:32:01 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:01 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:03 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #298 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #299 of plugin_7 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #300 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #301 of plugin_8 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #302 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #303 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #304 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #305 of plugin_7 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #306 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #307 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #308 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #309 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #310 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #311 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:04 [info] 1267#0: *149 [python:1269] INFO - [17:32:04] instance #312 of plugin_4 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:05 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:05 [error] 1267#0: *7128 [kong] mp_rpc.lua:347 [plugin_5] Could not claim instance_id for plugin_5 (key: e134ea38-2b10-5b3a-9a35-631b6fe01e78), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:06 [notice] 1267#0: *308 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, client: 127.0.0.1, server: kong_admin, request: "POST /config?check_hash=1 HTTP/2.0", host: "localhost:8444"
2023/07/05 09:32:06 [notice] 1267#0: *308 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, client: 127.0.0.1, server: kong_admin, request: "POST /config?check_hash=1 HTTP/2.0", host: "localhost:8444"
127.0.0.1 - - [05/Jul/2023:09:32:06 +0000] "POST /config?check_hash=1 HTTP/2.0" 201 4230 "-" "Go-http-client/2.0"
2023/07/05 09:32:06 [info] 1268#0: *154 [lua] handler.lua:604: declarative reconfigure was started on worker #1, context: ngx.timer
2023/07/05 09:32:06 [info] 1268#0: *154 [lua] handler.lua:639: building a new router took 0 ms on worker #1, context: ngx.timer
2023/07/05 09:32:06 [info] 1268#0: *154 [lua] handler.lua:652: building a new plugins iterator took 0 ms on worker #1, context: ngx.timer
2023/07/05 09:32:06 [notice] 1268#0: *154 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:32:06 [notice] 1268#0: *154 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:32:06 [info] 1268#0: *154 [lua] handler.lua:692: declarative reconfigure took 3 ms on worker #1, context: ngx.timer
2023/07/05 09:32:06 [info] 1267#0: *4 [lua] handler.lua:604: declarative reconfigure was started on worker #0, context: ngx.timer
2023/07/05 09:32:06 [info] 1267#0: *4 [lua] handler.lua:639: building a new router took 1 ms on worker #0, context: ngx.timer
2023/07/05 09:32:06 [info] 1267#0: *4 [lua] handler.lua:652: building a new plugins iterator took 1 ms on worker #0, context: ngx.timer
2023/07/05 09:32:06 [notice] 1267#0: *4 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:32:06 [notice] 1267#0: *4 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:32:06 [info] 1267#0: *4 [lua] handler.lua:692: declarative reconfigure took 12 ms on worker #0, context: ngx.timer
2023/07/05 09:32:06 [info] 1267#0: *149 [python:1269] WARN - [17:32:06] rpc: #44384 error: no plugin instance #309, context: ngx.timer
2023/07/05 09:32:06 [warn] 1267#0: *7106 [kong] mp_rpc.lua:343 [plugin_2] no plugin instance #309, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:07 [error] 1267#0: *7123 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:07 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #313 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #314 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #315 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #316 of plugin_9 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #317 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #318 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #319 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #320 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #321 of plugin_8 started, context: ngx.timer
2023/07/05 09:32:08 [info] 1267#0: *149 [python:1269] INFO - [17:32:08] instance #322 of plugin_10 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:08 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:09 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:09 [error] 1267#0: *7117 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:09 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:10 [error] 1267#0: *7126 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:10 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:10 [error] 1267#0: *7131 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:10 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:10 [error] 1267#0: *7130 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:10 [error] 1267#0: *7116 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:10 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:10 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:10 [error] 1267#0: *7127 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:10 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:12 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:13 [info] 1267#0: *149 [python:1269] INFO - [17:32:13] instance #323 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:13 [info] 1267#0: *149 [python:1269] INFO - [17:32:13] instance #324 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:13 [info] 1267#0: *149 [python:1269] INFO - [17:32:13] instance #325 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:13 [info] 1267#0: *149 [python:1269] INFO - [17:32:13] instance #326 of plugin_7 started, context: ngx.timer
2023/07/05 09:32:13 [info] 1267#0: *149 [python:1269] INFO - [17:32:13] instance #327 of plugin_9 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:15 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:17 [info] 1267#0: *149 [python:1269] INFO - [17:32:17] instance #328 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:18 [notice] 1267#0: *308 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, client: 127.0.0.1, server: kong_admin, request: "POST /config?check_hash=1 HTTP/2.0", host: "localhost:8444"
2023/07/05 09:32:18 [notice] 1267#0: *308 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, client: 127.0.0.1, server: kong_admin, request: "POST /config?check_hash=1 HTTP/2.0", host: "localhost:8444"
2023/07/05 09:32:18 [info] 1268#0: *154 [lua] handler.lua:604: declarative reconfigure was started on worker #1, context: ngx.timer
2023/07/05 09:32:18 [info] 1268#0: *154 [lua] handler.lua:639: building a new router took 0 ms on worker #1, context: ngx.timer
2023/07/05 09:32:18 [info] 1268#0: *154 [lua] handler.lua:652: building a new plugins iterator took 1 ms on worker #1, context: ngx.timer
2023/07/05 09:32:18 [notice] 1268#0: *154 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:32:18 [notice] 1268#0: *154 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:32:18 [info] 1268#0: *154 [lua] handler.lua:692: declarative reconfigure took 3 ms on worker #1, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:18 +0000] "POST /config?check_hash=1 HTTP/2.0" 201 4467 "-" "Go-http-client/2.0"
2023/07/05 09:32:18 [info] 1267#0: *4 [lua] handler.lua:604: declarative reconfigure was started on worker #0, context: ngx.timer
2023/07/05 09:32:18 [info] 1267#0: *4 [lua] handler.lua:639: building a new router took 1 ms on worker #0, context: ngx.timer
2023/07/05 09:32:18 [info] 1267#0: *4 [lua] handler.lua:652: building a new plugins iterator took 1 ms on worker #0, context: ngx.timer
2023/07/05 09:32:18 [notice] 1267#0: *4 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:32:18 [notice] 1267#0: *4 [lua] init.lua:240: purge(): [DB cache] purging (local) cache, context: ngx.timer
2023/07/05 09:32:18 [info] 1267#0: *4 [lua] handler.lua:692: declarative reconfigure took 5 ms on worker #0, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:19 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:21 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #329 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #330 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #331 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #332 of plugin_8 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #333 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #334 of plugin_9 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #335 of plugin_7 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #336 of plugin_8 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #337 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #338 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #339 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] ERROR - [17:32:22] rpc: #44476 exception: Traceback (most recent call last):, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269]   File "/home/kong/.venv/lib/python3.10/site-packages/kong_pdk/listener.py", line 68, in handle, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269]     ret = getattr(self.ps, cmd_r)(*args), context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269]   File "/home/kong/.venv/lib/python3.10/site-packages/kong_pdk/server.py", line 49, in wrapper, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269]     raise ex, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269]   File "/home/kong/.venv/lib/python3.10/site-packages/kong_pdk/server.py", line 46, in wrapper, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269]     r = fn(*args, **kwargs), context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269]   File "/home/kong/.venv/lib/python3.10/site-packages/kong_pdk/server.py", line 273, in handle_event, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269]     iid = event['InstanceId'], context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] KeyError: 'InstanceId', context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:22 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:22 [error] 1267#0: *7132 [kong] mp_rpc.lua:347 [plugin_2] 'InstanceId', client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #340 of plugin_7 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #341 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] WARN - [17:32:22] rpc: #44485 error: no plugin instance #333, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] WARN - [17:32:22] rpc: #44497 error: no plugin instance #338, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #342 of plugin_9 started, context: ngx.timer
2023/07/05 09:32:22 [warn] 1267#0: *7128 [kong] mp_rpc.lua:343 [plugin_10] no plugin instance #333, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:22 [warn] 1267#0: *7149 [kong] mp_rpc.lua:343 [plugin_10] no plugin instance #338, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #343 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #344 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #345 of plugin_10 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:22 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #346 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #347 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] WARN - [17:32:22] rpc: #44504 error: no plugin instance #341, context: ngx.timer
2023/07/05 09:32:22 [warn] 1267#0: *7132 [kong] mp_rpc.lua:343 [plugin_10] no plugin instance #341, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #348 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #349 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #350 of plugin_8 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #351 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] WARN - [17:32:22] rpc: #44512 error: no plugin instance #330, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] WARN - [17:32:22] rpc: #44500 error: no plugin instance #330, context: ngx.timer
2023/07/05 09:32:22 [warn] 1267#0: *7161 [kong] mp_rpc.lua:343 [plugin_6] no plugin instance #330, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:22 [warn] 1267#0: *7159 [kong] mp_rpc.lua:343 [plugin_6] no plugin instance #330, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:22 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] WARN - [17:32:22] rpc: #44521 error: no plugin instance #348, context: ngx.timer
2023/07/05 09:32:22 [info] 1267#0: *149 [python:1269] INFO - [17:32:22] instance #352 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:22 [warn] 1267#0: *7132 [kong] mp_rpc.lua:343 [plugin_10] no plugin instance #348, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:24 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
127.0.0.1 - - [05/Jul/2023:09:32:24 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #353 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #354 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #355 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #356 of plugin_8 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #357 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #358 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #359 of plugin_7 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #360 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #361 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #362 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #363 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #364 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #365 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:27 [info] 1267#0: *149 [python:1269] INFO - [17:32:27] instance #366 of plugin_7 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:27 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
127.0.0.1 - - [05/Jul/2023:09:32:30 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
127.0.0.1 - - [05/Jul/2023:09:32:31 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:31 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:32 [info] 1267#0: *149 [python:1269] INFO - [17:32:32] instance #367 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:32 [info] 1267#0: *149 [python:1269] INFO - [17:32:32] instance #368 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:32 [info] 1267#0: *149 [python:1269] INFO - [17:32:32] instance #369 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:32 [info] 1267#0: *149 [python:1269] INFO - [17:32:32] instance #370 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:32 [info] 1267#0: *149 [python:1269] INFO - [17:32:32] instance #371 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:32 [info] 1267#0: *149 [python:1269] INFO - [17:32:32] instance #372 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:32 [error] 1267#0: *7149 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:32 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:33 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269] ERROR - [17:32:33] rpc: #44589 exception: Traceback (most recent call last):, context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269]   File "/home/kong/.venv/lib/python3.10/site-packages/kong_pdk/listener.py", line 68, in handle, context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269]     ret = getattr(self.ps, cmd_r)(*args), context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269]   File "/home/kong/.venv/lib/python3.10/site-packages/kong_pdk/server.py", line 49, in wrapper, context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269]     raise ex, context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269]   File "/home/kong/.venv/lib/python3.10/site-packages/kong_pdk/server.py", line 46, in wrapper, context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269]     r = fn(*args, **kwargs), context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269]   File "/home/kong/.venv/lib/python3.10/site-packages/kong_pdk/server.py", line 273, in handle_event, context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269]     iid = event['InstanceId'], context: ngx.timer
2023/07/05 09:32:33 [info] 1267#0: *149 [python:1269] KeyError: 'InstanceId', context: ngx.timer
2023/07/05 09:32:33 [error] 1267#0: *7161 [kong] mp_rpc.lua:347 [plugin_5] 'InstanceId', client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:36 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:36 [error] 1267#0: *7148 [kong] mp_rpc.lua:347 [plugin_10] Could not claim instance_id for plugin_10 (key: 1628c104-ae90-501a-981e-2a0a3eacf87e), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:36 [error] 1267#0: *7151 [kong] mp_rpc.lua:347 [plugin_10] Could not claim instance_id for plugin_10 (key: 1628c104-ae90-501a-981e-2a0a3eacf87e), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:36 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #373 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #374 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #375 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #376 of plugin_6 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #377 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #378 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #379 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #380 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #381 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] INFO - [17:32:37] instance #382 of plugin_5 started, context: ngx.timer
2023/07/05 09:32:37 [info] 1267#0: *149 [python:1269] WARN - [17:32:37] rpc: #44620 error: no plugin instance #374, context: ngx.timer
2023/07/05 09:32:37 [warn] 1267#0: *7166 [kong] mp_rpc.lua:343 [plugin_4] no plugin instance #374, client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:38 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:39 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
127.0.0.1 - - [05/Jul/2023:09:32:41 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:41 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:41 [error] 1267#0: *7159 [kong] mp_rpc.lua:347 [plugin_4] Could not claim instance_id for plugin_4 (key: 8082d69c-8447-5dae-81e7-27d7ef17fef8), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:42 [info] 1267#0: *149 [python:1269] INFO - [17:32:42] instance #383 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:42 [info] 1267#0: *149 [python:1269] INFO - [17:32:42] instance #384 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:42 [info] 1267#0: *149 [python:1269] INFO - [17:32:42] instance #385 of plugin_2 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:42 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:43 [error] 1267#0: *7186 [kong] mp_rpc.lua:347 [plugin_4] Could not claim instance_id for plugin_4 (key: 8082d69c-8447-5dae-81e7-27d7ef17fef8), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:44 [error] 1267#0: *7161 [kong] mp_rpc.lua:347 [plugin_4] Could not claim instance_id for plugin_4 (key: 8082d69c-8447-5dae-81e7-27d7ef17fef8), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:45 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
127.0.0.1 - - [05/Jul/2023:09:32:46 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:46 [error] 1267#0: *7187 [kong] mp_rpc.lua:347 [plugin_4] Could not claim instance_id for plugin_4 (key: 8082d69c-8447-5dae-81e7-27d7ef17fef8), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:46 [error] 1267#0: *7195 [kong] mp_rpc.lua:347 [plugin_4] Could not claim instance_id for plugin_4 (key: 8082d69c-8447-5dae-81e7-27d7ef17fef8), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:46 [error] 1267#0: *7191 [kong] mp_rpc.lua:347 [plugin_4] Could not claim instance_id for plugin_4 (key: 8082d69c-8447-5dae-81e7-27d7ef17fef8), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:46 [error] 1267#0: *7189 [kong] mp_rpc.lua:347 [plugin_4] Could not claim instance_id for plugin_4 (key: 8082d69c-8447-5dae-81e7-27d7ef17fef8), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #386 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #387 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #388 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #389 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #390 of plugin_4 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #391 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #392 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #393 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #394 of plugin_10 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:47 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #395 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:47 [info] 1267#0: *149 [python:1269] INFO - [17:32:47] instance #396 of plugin_3 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:48 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
127.0.0.1 - - [05/Jul/2023:09:32:48 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:51 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:51 [error] 1267#0: *7162 [kong] mp_rpc.lua:347 [plugin_1] Could not claim instance_id for plugin_1 (key: 9b25ecd6-c3e4-5efc-84ce-b5d6ef3e57ba), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:51 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
2023/07/05 09:32:52 [info] 1267#0: *149 [python:1269] INFO - [17:32:52] instance #397 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:52 [info] 1267#0: *149 [python:1269] INFO - [17:32:52] instance #398 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:52 [error] 1267#0: *7166 [kong] mp_rpc.lua:347 [plugin_3] Could not claim instance_id for plugin_3 (key: 4fc57e8d-380b-532b-9f98-9ee8489b02e4), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
127.0.0.1 - - [05/Jul/2023:09:32:53 +0000] "GET /echo HTTP/1.1" 200 1301 "-" "Mozilla/5.0 (apple-arm-darwin21.6.0) Siege/4.1.6"
127.0.0.1 - - [05/Jul/2023:09:32:54 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
2023/07/05 09:32:57 [error] 1267#0: *7187 [kong] mp_rpc.lua:347 [plugin_3] Could not claim instance_id for plugin_3 (key: 4fc57e8d-380b-532b-9f98-9ee8489b02e4), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:57 [error] 1267#0: *7195 [kong] mp_rpc.lua:347 [plugin_3] Could not claim instance_id for plugin_3 (key: 4fc57e8d-380b-532b-9f98-9ee8489b02e4), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:57 [error] 1267#0: *7191 [kong] mp_rpc.lua:347 [plugin_3] Could not claim instance_id for plugin_3 (key: 4fc57e8d-380b-532b-9f98-9ee8489b02e4), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:57 [error] 1267#0: *7189 [kong] mp_rpc.lua:347 [plugin_3] Could not claim instance_id for plugin_3 (key: 4fc57e8d-380b-532b-9f98-9ee8489b02e4), client: 127.0.0.1, server: kong, request: "GET /echo HTTP/1.1", host: "localhost:8080"
2023/07/05 09:32:57 [info] 1267#0: *149 [python:1269] INFO - [17:32:57] instance #399 of plugin_1 started, context: ngx.timer
2023/07/05 09:32:57 [info] 1267#0: *149 [python:1269] INFO - [17:32:57] instance #400 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:57 [info] 1267#0: *149 [python:1269] INFO - [17:32:57] instance #401 of plugin_10 started, context: ngx.timer
2023/07/05 09:32:57 [info] 1267#0: *149 [python:1269] INFO - [17:32:57] instance #402 of plugin_2 started, context: ngx.timer
2023/07/05 09:32:57 [info] 1267#0: *149 [python:1269] INFO - [17:32:57] instance #403 of plugin_3 started, context: ngx.timer
2023/07/05 09:32:57 [info] 1267#0: *149 [python:1269] INFO - [17:32:57] instance #404 of plugin_10 started, context: ngx.timer
127.0.0.1 - - [05/Jul/2023:09:32:57 +0000] "GET /status HTTP/2.0" 200 1074 "-" "Go-http-client/2.0"
```

# Conclusion
It looks like there is a race condition in [get_instance_id](https://github.com/Kong/kong/blob/master/kong/runloop/plugin_servers/init.lua#L199)
in the result of which non-initialized plugin instance may not be cleaned up from `running_instances`
and any subsequent calls of `get_instance_id` will fail. Another observation that was made during
the test is that `plugins_hash` is always nil when `/config?check_hash=1` is triggered, and it
results in [rebuilding plugins iterator](https://github.com/Kong/kong/blob/master/kong/runloop/handler.lua#L640)
on every reconfigure, although plugins never change. This increases the likelihood of the race
condition described earlier.