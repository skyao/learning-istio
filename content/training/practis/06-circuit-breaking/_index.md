---
title: "熔断"
linkTitle: "熔断"
weight: 60
date: 2024-06-18
description: >
  熔断:“秒杀”场景下的过载保护是如何实现的?
---


参考官方文档：

- https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/

## 准备工作

### httpbin

启动 httpbin：

```bash
kubectl apply -f samples/httpbin/httpbin.yaml
```

为 httpbin 创建 DestinationRule：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 1
      interval: 1s
      baseEjectionTime: 3m
      maxEjectionPercent: 100
EOF
```

检验 DestinationRule 是否创建成功：

```bash
kubectl get destinationrule httpbin -o yaml
```

### 创建客户端

使用 fortio 作为简单的压力测试工具。

```bash
kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml
```

获取 FORTIO_POD：

```bash
export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
echo ${FORTIO_POD}
```

发一个请求测试一下：

```bash
kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/get
```

能看输出：

```json
HTTP/1.1 200 OK
server: envoy
date: Thu, 27 Jun 2024 22:52:15 GMT
content-type: application/json
content-length: 425
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 4

{
  "args": {}, 
  "headers": {
    "Host": "httpbin:8000", 
    "User-Agent": "fortio.org/fortio-1.60.3", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=ea0bf859014839adacf61dbd5d7184ffd30068e328d9ea3580c0ed010127ddae;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default"
  }, 
  "origin": "127.0.0.6", 
  "url": "http://httpbin:8000/get"
}
```

验证完成，开始。

## 演示

在 DestinationRule 设置中，您指定了 

- maxConnections：1 
- http1MaxPendingRequests： 1

这些规则表明，如果同时超过一个以上的连接和请求，当代理服务器为更多请求和连接打开时，就会出现一些故障:

### 演示1:并发连接数=2

使用两个并发连接 (-c 2) 调用服务，并发送 20 个请求 (-n 20)：

```bash
kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
```

输出为:

```bash
{"ts":1719529031.314366,"level":"info","r":1,"file":"logger.go","line":254,"msg":"Log level is now 3 Warning (was 2 Info)"}
Fortio 1.60.3 running at 0 queries per second, 4->4 procs, for 20 calls: http://httpbin:8000/get
Starting at max qps with 2 thread(s) [gomax 4] for exactly 20 calls (10 per thread + 0)
{"ts":1719529031.316773,"level":"warn","r":52,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
{"ts":1719529031.329077,"level":"warn","r":51,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529031.330502,"level":"warn","r":51,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
Ended after 16.412752ms : 20 calls. qps=1218.6
Aggregated Function Time : count 20 avg 0.0015931135 +/- 0.0008787 min 0.000134604 max 0.004188296 sum 0.031862271
# range, mid point, percentile, count
>= 0.000134604 <= 0.001 , 0.000567302 , 5.00, 1
> 0.001 <= 0.002 , 0.0015 , 75.00, 14
> 0.002 <= 0.003 , 0.0025 , 95.00, 4
> 0.004 <= 0.0041883 , 0.00409415 , 100.00, 1
# target 50% 0.00164286
# target 75% 0.002
# target 90% 0.00275
# target 99% 0.00415064
# target 99.9% 0.00418453
Error cases : count 3 avg 0.0011060533 +/- 0.0007001 min 0.000134604 max 0.001757492 sum 0.00331816
# range, mid point, percentile, count
>= 0.000134604 <= 0.001 , 0.000567302 , 33.33, 1
> 0.001 <= 0.00175749 , 0.00137875 , 100.00, 2
# target 50% 0.00118937
# target 75% 0.00147343
# target 90% 0.00164387
# target 99% 0.00174613
# target 99.9% 0.00175636
# Socket and IP used for each connection:
[0]   2 socket used, resolved to 10.96.89.19:8000, connection timing : count 2 avg 4.248e-05 +/- 6.103e-06 min 3.6377e-05 max 4.8583e-05 sum 8.496e-05
[1]   2 socket used, resolved to 10.96.89.19:8000, connection timing : count 2 avg 4.23905e-05 +/- 3.798e-06 min 3.8592e-05 max 4.6189e-05 sum 8.4781e-05
Connection time (s) : count 4 avg 4.243525e-05 +/- 5.083e-06 min 3.6377e-05 max 4.8583e-05 sum 0.000169741
Sockets used: 4 (for perfect keepalive, would be 2)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.96.89.19:8000: 4
Code 200 : 17 (85.0 %)
Code 503 : 3 (15.0 %)
Response Header Sizes : count 20 avg 195.5 +/- 82.13 min 0 max 230 sum 3910
Response Body/Total Sizes : count 20 avg 592.9 +/- 147.8 min 241 max 655 sum 11858
All done 20 calls (plus 0 warmup) 1.593 ms avg, 1218.6 qps
```

注意，20个请求中，17个正常返回200，3个请求返回 503

```bash
......
{"ts":1719529031.316773,"level":"warn","r":52,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
{"ts":1719529031.329077,"level":"warn","r":51,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529031.330502,"level":"warn","r":51,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
......
Code 200 : 17 (85.0 %)
Code 503 : 3 (15.0 %)
......
```

### 演示2:并发连接数=3

使并发连接数达到 3：

```bash
kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get
```

输出为:

```bash
{"ts":1719529422.266273,"level":"info","r":1,"file":"logger.go","line":254,"msg":"Log level is now 3 Warning (was 2 Info)"}
Fortio 1.60.3 running at 0 queries per second, 4->4 procs, for 30 calls: http://httpbin:8000/get
Starting at max qps with 3 thread(s) [gomax 4] for exactly 30 calls (10 per thread + 0)
{"ts":1719529422.268081,"level":"warn","r":14,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
{"ts":1719529422.268324,"level":"warn","r":15,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":2,"run":0}
{"ts":1719529422.269315,"level":"warn","r":14,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
{"ts":1719529422.269799,"level":"warn","r":14,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
{"ts":1719529422.269899,"level":"warn","r":13,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529422.270627,"level":"warn","r":14,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
{"ts":1719529422.271810,"level":"warn","r":15,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":2,"run":0}
{"ts":1719529422.273191,"level":"warn","r":14,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
{"ts":1719529422.273547,"level":"warn","r":13,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529422.274095,"level":"warn","r":14,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
{"ts":1719529422.274334,"level":"warn","r":15,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":2,"run":0}
{"ts":1719529422.275270,"level":"warn","r":15,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":2,"run":0}
{"ts":1719529422.275922,"level":"warn","r":15,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":2,"run":0}
{"ts":1719529422.276326,"level":"warn","r":13,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529422.276827,"level":"warn","r":13,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529422.277530,"level":"warn","r":13,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529422.277910,"level":"warn","r":13,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529422.278238,"level":"warn","r":13,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":0,"run":0}
{"ts":1719529422.278624,"level":"warn","r":15,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":2,"run":0}
{"ts":1719529422.279820,"level":"warn","r":14,"file":"http_client.go","line":1104,"msg":"Non ok http code","code":503,"status":"HTTP/1.1 503","thread":1,"run":0}
Ended after 13.176002ms : 30 calls. qps=2276.9
Aggregated Function Time : count 30 avg 0.0012081287 +/- 0.000993 min 0.000135735 max 0.003231557 sum 0.036243862
# range, mid point, percentile, count
>= 0.000135735 <= 0.001 , 0.000567868 , 60.00, 18
> 0.001 <= 0.002 , 0.0015 , 66.67, 2
> 0.002 <= 0.003 , 0.0025 , 90.00, 7
> 0.003 <= 0.00323156 , 0.00311578 , 100.00, 3
# target 50% 0.000847483
# target 75% 0.00235714
# target 90% 0.003
# target 99% 0.0032084
# target 99.9% 0.00322924
Error cases : count 20 avg 0.0005456627 +/- 0.0002905 min 0.000135735 max 0.001242578 sum 0.010913254
# range, mid point, percentile, count
>= 0.000135735 <= 0.001 , 0.000567868 , 90.00, 18
> 0.001 <= 0.00124258 , 0.00112129 , 100.00, 2
# target 50% 0.000593287
# target 75% 0.000847483
# target 90% 0.001
# target 99% 0.00121832
# target 99.9% 0.00124015
# Socket and IP used for each connection:
[0]   7 socket used, resolved to 10.96.89.19:8000, connection timing : count 7 avg 4.2301143e-05 +/- 1.593e-05 min 2.6914e-05 max 7.4179e-05 sum 0.000296108
[1]   7 socket used, resolved to 10.96.89.19:8000, connection timing : count 7 avg 4.4322714e-05 +/- 1.181e-05 min 3.1746e-05 max 6.3054e-05 sum 0.000310259
[2]   7 socket used, resolved to 10.96.89.19:8000, connection timing : count 7 avg 0.000132666 +/- 0.0002153 min 2.7545e-05 max 0.000657932 sum 0.000928662
Connection time (s) : count 21 avg 7.3096619e-05 +/- 0.0001317 min 2.6914e-05 max 0.000657932 sum 0.001535029
Sockets used: 21 (for perfect keepalive, would be 3)
Uniform: false, Jitter: false, Catchup allowed: true
IP addresses distribution:
10.96.89.19:8000: 21
Code 200 : 10 (33.3 %)
Code 503 : 20 (66.7 %)
Response Header Sizes : count 30 avg 76.666667 +/- 108.4 min 0 max 230 sum 2300
Response Body/Total Sizes : count 30 avg 379 +/- 195.2 min 241 max 655 sum 11370
All done 30 calls (plus 0 warmup) 1.208 ms avg, 2276.9 qps
```

此时，30个请求中成功的仅有10个，失败的(被sidecar熔断）多达20个：

```bash
Code 200 : 10 (33.3 %)
Code 503 : 20 (66.7 %)
```

查看一下 sidecar 的状态：

```bash
kubectl exec "$FORTIO_POD" -c istio-proxy -- pilot-agent request GET stats | grep httpbin | grep pending
```

输出中的 "upstream_rq_pending_overflow" 就是被熔断的总数量（演示1 3个 + 演示2 20个）：

```bash
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.remaining_pending: 1
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 23
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 28
```

## 清理

```bash
cd ~/work/soft/istio
kubectl delete destinationrule httpbin
kubectl delete -f samples/httpbin/sample-client/fortio-deploy.yaml
kubectl delete -f samples/httpbin/httpbin.yaml
```