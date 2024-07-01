---
title: "服务入口"
linkTitle: "服务入口"
weight: 40
date: 2024-06-18
description: >
  服务入口: 用 Service Entry 扩展你的网格服务
---


参考官方文档：

- https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/


## 准备工作

### 相关的应用

```bash
kubectl apply -f samples/sleep/sleep.yaml
```

```bash
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})
```

### 开启 envoy access logging

除了正常安装 istio 之外，还需要打开 access logging 以查看日志。

```bash
kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  accessLogging:
    - providers:
      - name: envoy
EOF
```

### 本地验证

先本地跑几个curl命令看看输出：

```bash
$ curl http://www.baidu.com
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
```

```bash
curl -I http://www.baidu.com 
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
Connection: keep-alive
Content-Length: 277
Content-Type: text/html
Date: Wed, 26 Jun 2024 03:15:58 GMT
Etag: "575e1f71-115"
Last-Modified: Mon, 13 Jun 2016 02:50:25 GMT
Pragma: no-cache
Server: bfe/1.0.8.18
```

### 验证可在集群内访问外部地址

```bash
kubectl exec "$SOURCE_POD" -c sleep -- curl http://www.baidu.com
```

```html
kubectl exec "$SOURCE_POD" -c sleep -- curl http://www.baidu.com
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2381  100  2381    0     0   137k      0 --:--:-- --:--:-- --:--:--  145k
<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
```

## 演示1: 访问外部服务

### 创建 Service Entry

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: baidu-ext
spec:
  hosts:
  - baidu.com
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

### 访问外部服务

```bash
kubectl exec "$SOURCE_POD" -c sleep -- curl http://www.baidu.com
```

输出为:

```bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html>
<!--STATUS OK--><html> <head><meta http-equiv=content-type content=text/html;charset=utf-8><meta http-equiv=X-UA-Compatible content=IE=Edge><meta content=always name=referrer><link rel=stylesheet type=text/css href=http://s1.bdstatic.com/r/www/cache/bdorz/baidu.min.css><title>百度一下，你就知道</title></head> <body link=#0000cc> <div id=wrapper> <div id=head> <div class=head_wrapper> <div class=s_form> <div class=s_form_wrapper> <div id=lg> <img hidefocus=true src=//www.baidu.com/img/bd_logo1.png width=270 height=129> </div> <form id=form name=f action=//www.baidu.com/s class=fm> <input type=hidden name=bdorz_come value=1> <input type=hidden name=ie value=utf-8> <input type=hidden name=f value=8> <input type=hidden name=rsv_bp value=1> <input type=hidden name=rsv_idx value=1> <input type=hidden name=tn value=baidu><span class="bg s_ipt_wr"><input id=kw name=wd class=s_ipt value maxlength=255 autocomplete=off autofocus></span><span class="bg s_btn_wr"><input type=submit id=su value=百度一下 class="bg s_btn"></span> </form> </div> </div> <div id=u1> <a href=http://news.baidu.com name=tj_trnews class=mnav>新闻</a> <a href=http://www.hao123.com name=tj_trhao123 class=mnav>hao123</a> <a href=http://map.baidu.com name=tj_trmap class=mnav>地图</a> <a href=http://v.baidu.com name=tj_trvideo class=mnav>视频</a> <a href=http://tieba.baidu.com name=tj_trtieba class=mnav>贴吧</a> <noscript> <a href=http://www.baidu.com/bdorz/login.gif?login&amp;tpl=mn&amp;u=http%3A%2F%2Fwww.baidu.com%2f%3fbdorz_come%3d1 name=tj_login class=lb>登录</a> </noscript> <script>document.write('<a href="http://www.baidu.com/bdorz/login.gif?login&tpl=mn&u='+ encodeURIComponent(window.location.href+ (window.location.search === "" ? "?" : "&")+ "bdorz_come=1")+ '" name="tj_login" class="lb">登录</a>');</script> <a href=//www.baidu.com/more/ name=tj_briicon class=bri style="display: block;">更多产品</a> </div> </div> </div> <div id=ftCon> <div id=ftConw> <p id=lh> <a href=http://home.baidu.com>关于百度</a> <a href=http://ir.baidu.com>About Baidu</a> </p> <p id=cp>&copy;2017&nbsp;Baidu&nbsp;<a href=http://www.baidu.com/duty/>使用百度前必读</a>&nbsp; <a href=http://jianyi.baidu.com/ class=cp-feedback>意见反馈</a>&nbsp;京ICP证030173号&nbsp; <img src=//www.baidu.com/img/gs.gif> </p> </div> </div> </div> </body> </html>
100  2381  100  2381    0     0   146k      0 --:--:-- --:--:-- --:--:--  155k
```

检查 envoy 的日志:

```bash
kubectl logs "$SOURCE_POD" -c istio-proxy | tail
```

可以看到类似的记录：

```bash
[2024-06-26T03:30:14.684Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 2381 12 12 "-" "curl/8.8.0" "599df0d8-0c22-9daa-820e-b4b03dd12daf" "www.baidu.com" "183.2.172.185:80" PassthroughCluster 10.244.96.245:47908 183.2.172.185:80 10.244.96.245:47906 - allow_any
```

验证一下，故意发一个错误的url，能看到 404 404 Not Found：

```bash
kubectl exec "$SOURCE_POD" -c sleep -- curl http://www.baidu.com/aaa
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   201  100   201    0     0   8519      0 --:--:-- --:--:-- --:--:--  8739
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>404 Not Found</title>
</head><body>
<h1>Not Found</h1>
<p>The requested URL /aaa was not found on this server.</p>
</body></html>
```

日志里面也会多出来一行这样的内容：

```bash
[2024-06-26T03:31:52.780Z] "GET /aaa HTTP/1.1" 404 - via_upstream - "-" 0 201 19 19 "-" "curl/8.8.0" "5f558197-cf05-969f-b82a-addea5e0e161" "www.baidu.com" "183.2.172.42:80" PassthroughCluster 10.244.96.245:36004 183.2.172.42:80 10.244.96.245:35994 - allow_any
```

这就验证了，在这个 pod 内如果发起 http 请求访问 www.baidu.com 这样的URL，则会被 istio 劫持并转发。

## 演示2: 管理外部服务

### 准备工作

前面的演示只是简单的可以访问，但外部服务被纳入 istio 管理之后，就可以使用 istio 提供的一些高级功能了。

为了演示方便，使用 http://httpbin.org/ 这个网站来代替 www.baidu.com 作为测试的外部服务器。

类似的，为 httpbin.org 创建 Service Entry：

```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
EOF
```

访问这个外部服务，但注意这个URL是会延迟5秒返回：

```bash
kubectl exec "$SOURCE_POD" -c sleep -- curl http://httpbin.org/delay/5
```

输出为:

```bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:05 --:--:--     0{
  "args": {}, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin.org", 
    "User-Agent": "curl/8.8.0", 
    "X-Amzn-Trace-Id": "Root=1-667ba89d-5f26009b6c33ff87227f51bd", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Envoy-Decorator-Operation": "httpbin.org:80/*", 
    "X-Envoy-Peer-Metadata": "ChkKDkFQUF9DT05UQUlORVJTEgcaBXNsZWVwChoKCkNMVVNURVJfSUQSDBoKS3ViZXJuZXRlcwofCgxJTlNUQU5DRV9JUFMSDxoNMTAuMjQ0Ljk2LjI0NQoZCg1JU1RJT19WRVJTSU9OEggaBjEuMjIuMQqhAQoGTEFCRUxTEpYBKpMBCg4KA2FwcBIHGgVzbGVlcAokChlzZWN1cml0eS5pc3Rpby5pby90bHNNb2RlEgcaBWlzdGlvCioKH3NlcnZpY2UuaXN0aW8uaW8vY2Fub25pY2FsLW5hbWUSBxoFc2xlZXAKLwojc2VydmljZS5pc3Rpby5pby9jYW5vbmljYWwtcmV2aXNpb24SCBoGbGF0ZXN0ChoKB01FU0hfSUQSDxoNY2x1c3Rlci5sb2NhbAogCgROQU1FEhgaFnNsZWVwLTc2NTZjZjg3OTQtcDg2ZjIKFgoJTkFNRVNQQUNFEgkaB2RlZmF1bHQKSQoFT1dORVISQBo+a3ViZXJuZXRlczovL2FwaXMvYXBwcy92MS9uYW1lc3BhY2VzL2RlZmF1bHQvZGVwbG95bWVudHMvc2xlZXAKGAoNV09SS0xPQURfTkFNRRIHGgVzbGVlcA==", 
    "X-Envoy-Peer-Metadata-Id": "sidecar~10.244.96.245~sleep-7656cf8794-p86f2.default~default.svc.cluster.local"
  }, 
  "origin": "46.232.50.12", 
  "url": "http://httpbin.org/delay/5"
}
100  1166  100  1166    0     0    208      0  0:00:05  0:00:05 --:--:--   324
```

检查 envoy 的日志:

```bash
kubectl logs "$SOURCE_POD" -c istio-proxy | tail
```

可以看到记录：

```bash
[2024-06-26T05:35:25.428Z] "GET /delay/5 HTTP/1.1" 200 - via_upstream - "-" 0 1166 5591 5591 "-" "curl/8.8.0" "41613cdb-2a2d-9e19-9a15-34b48de920c1" "httpbin.org" "44.206.219.79:80" outbound|80||httpbin.org 10.244.96.245:49362 44.206.219.79:80 10.244.96.245:49348 - default
```

### 增加超时设置


```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  http:
  - timeout: 3s
    route:
    - destination:
        host: httpbin.org
      weight: 100
EOF
```

继续访问这个会延迟5秒返回的URL：

```bash
kubectl exec "$SOURCE_POD" -c sleep -- curl http://httpbin.org/delay/5
```

输出为：

```bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    24  100    24    0     0      7      0  0:00:03  0:00:03 --:--:--    upstream request timeout 7
```

此时的日志是:

```bash
[2024-06-26T05:38:33.776Z] "GET /delay/5 HTTP/1.1" 504 UT response_timeout - "-" 0 24 3001 - "-" "curl/8.8.0" "6b6e7a5e-707b-9cb0-806b-5bca6edaa077" "httpbin.org" "44.195.190.188:80" outbound|80||httpbin.org 10.244.96.245:45852 44.206.219.79:80 10.244.96.245:60788 - -
```

用这个命令能看到服务器端返回的 http 状态码是 504

```bash
kubectl exec "$SOURCE_POD" -c sleep -- time curl -o /dev/null -sS -w "%{http_code}\n" http://httpbin.org/delay/5
```

```bash
504
real	0m 3.00s
user	0m 0.00s
sys	0m 0.00s
```

## 清理

删除上面创建的 service entry 和 virtual service：

```bash
k delete serviceentries.networking.istio.io -n default baidu-ext
k delete serviceentries.networking.istio.io -n default httpbin-ext
k delete virtualservices.networking.istio.io -n default httpbin-ext
```