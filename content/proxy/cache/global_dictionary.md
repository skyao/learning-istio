---
date: 2018-09-29T21:00:00+08:00
title: 全局字典
weight: 823
menu:
  main:
    parent: "mixercache"
description : "介绍Istio中的全局字典"
---

## 全局字典的生成方式

### global_dictionary.h

文件路径：`proxy/src/istio/mixerclient/global_dictionary.h`

定义了方法GetGlobalWords：

```c++
// 获取自动生成的全局词语（global words）.
const std::vector<std::string>& GetGlobalWords();
```

这个方法会在referenced.cc中调用以得到全局字典。我们来看看这个全局字典的内容是如何得来。

### create_global_dictionary.py

文件路径： `proxy/src/istio/mixerclient/create_global_dictionary.py`

这个python脚本可以生成全局字典：

```python
/*
 * 从
 * https://github.com/istio/api/blob/master/mixer/v1/global_dictionary.yaml
 * 自动生成全局字典。
 * 需要运行:
 *  ./create_global_dictionary.py \
 *    bazel-mixerclient/external/mixerapi_git/mixer/v1/global_dictionary.yaml \
 *      > src/global_dictionary.cc
 */

const std::vector<std::string> kGlobalWords{
"""

BOTTOM = r"""};

}  // namespace

const std::vector<std::string>& GetGlobalWords() { return kGlobalWords; }

}  // namespace mixerclient
}  // namespace istio"""

all_words = ''
with open(sys.argv[1]) as src_file:
    for line in src_file:
        if line.startswith("-"):
            all_words += "    \"" + line[1:].strip() + "\",\n"

print TOP + all_words + BOTTOM
```

我们模拟一下，手工试试生成全局字典的过程：

```bash
cd proxy/src/istio/mixerclient
## 把global_dictionary.yaml文件内容拿下来
## github地址为：https://github.com/istio/api/blob/master/mixer/v1/global_dictionary.yaml
wget https://raw.githubusercontent.com/istio/api/master/mixer/v1/global_dictionary.yaml
## 或者从下载下来的api仓库里面直接获取这个文件
cp ../../../../api/mixer/v1/global_dictionary.yaml .
## 执行脚本生成global_dictionary.cc文件
./create_global_dictionary.py global_dictionary.yaml > global_dictionary.cc
```

### global_dictionary.cc

`global_dictionary.cc`文件的内容：

```c++
namespace {
const std::vector<std::string> kGlobalWords{
    "source.ip",
    "source.port",
    "source.name",
    "source.uid",
    "source.namespace",
    "source.labels",
    "source.user",
    "target.ip",
    "target.port",
    "target.service",
    "target.name",
    "target.uid",
    "target.namespace",
    "target.labels",
    "target.user",
    "request.headers",
    "request.id",
    "request.path",
    "request.host",
    "request.method",
    "request.reason",
    "request.referer",
    "request.scheme",
    "request.size",
    "request.time",
    "request.useragent",
    "response.headers",
    "response.size",
    "response.time",
    "response.duration",
    "response.code",
    ":authority",
    ":method",
    ":path",
    ":scheme",
    ":status",
    "access-control-allow-origin",
    "access-control-allow-methods",
    "access-control-allow-headers",
    "access-control-max-age",
    "access-control-request-method",
    "access-control-request-headers",
    "accept-charset",
    "accept-encoding",
    "accept-language",
    "accept-ranges",
    "accept",
    "access-control-allow",
    "age",
    "allow",
    "authorization",
    "cache-control",
    "content-disposition",
    "content-encoding",
    "content-language",
    "content-length",
    "content-location",
    "content-range",
    "content-type",
    "cookie",
    "date",
    "etag",
    "expect",
    "expires",
    "from",
    "host",
    "if-match",
    "if-modified-since",
    "if-none-match",
    "if-range",
    "if-unmodified-since",
    "keep-alive",
    "last-modified",
    "link",
    "location",
    "max-forwards",
    "proxy-authenticate",
    "proxy-authorization",
    "range",
    "referer",
    "refresh",
    "retry-after",
    "server",
    "set-cookie",
    "strict-transport-sec",
    "transfer-encoding",
    "user-agent",
    "vary",
    "via",
    "www-authenticate",
    "GET",
    "POST",
    "http",
    "envoy",
    "'200'",
    "Keep-Alive",
    "chunked",
    "x-envoy-service-time",
    "x-forwarded-for",
    "x-forwarded-host",
    "x-forwarded-proto",
    "x-http-method-override",
    "x-request-id",
    "x-requested-with",
    "application/json",
    "application/xml",
    "gzip",
    "text/html",
    "text/html; charset=utf-8",
    "text/plain",
    "text/plain; charset=utf-8",
    "'0'",
    "'1'",
    "true",
    "false",
    "gzip, deflate",
    "max-age=0",
    "x-envoy-upstream-service-time",
    "x-envoy-internal",
    "x-envoy-expected-rq-timeout-ms",
    "x-ot-span-context",
    "x-b3-traceid",
    "x-b3-sampled",
    "x-b3-spanid",
    "tcp",
    "connection.id",
    "connection.received.bytes",
    "connection.received.bytes_total",
    "connection.sent.bytes",
    "connection.sent.bytes_total",
    "connection.duration",
    "context.protocol",
    "context.timestamp",
    "context.time",
    "0",
    "1",
    "200",
    "302",
    "400",
    "401",
    "403",
    "404",
    "409",
    "429",
    "499",
    "500",
    "501",
    "502",
    "503",
    "504",
    "destination.ip",
    "destination.port",
    "destination.service",
    "destination.name",
    "destination.uid",
    "destination.namespace",
    "destination.labels",
    "destination.user",
    "source.service",
    "api.service",
    "api.version",
    "api.operation",
    "api.protocol",
    "request.auth.principal",
    "request.auth.audiences",
    "request.auth.presenter",
    "request.api_key",
    "check.error_code",
    "check.error_message",
};
}  // namespace

const std::vector<std::string>& GetGlobalWords() { return kGlobalWords; }
```

## 全局字典的数据来源

这里的kGlobalWords的内容完全来自前面的global_dictionary.yaml文件，它的内容类似如下：

```yaml
# Section 1: Standard Istio attribute vocabulary
# https://istio.io/docs/reference/config/mixer/attribute-vocabulary.html

- source.ip
- source.port
- source.name
- source.uid
- source.namespace
- source.labels
- source.user
......
```

详细内容可以查看文件：

https://github.com/istio/api/blob/master/mixer/v1/global_dictionary.yaml





