---
date: 2018-09-29T21:00:00+08:00
title: check_cache源码
weight: 321
menu:
  main:
    parent: "mixercache"
keywords: 
- mixer cache
- referenced
- istio源码
description : "check_cache源码"
---


源文件路径为：

- `proxy/src/istio/mixerclient/check_cache.h`
- `proxy/src/istio/mixerclient/check_cache.cc`

## 类CheckCache.CheckResult

```c++
  // 每个请求的检查缓存结果。缓存使用方式：
  //   cache->Check(attributes, result);
  //   if (result->IsCacheHit()) return result->Status();
  // 缓存的保存方式，发起远程调用然后在收到response时：
  //   result->SetReponse(status, response);
  //   return result->Status();
  class CheckResult {
   public:
    CheckResult();

    bool IsCacheHit() const;

    ::google::protobuf::util::Status status() const { return status_; }

    void SetResponse(const ::google::protobuf::util::Status& status,
                     const ::istio::mixer::v1::Attributes& attributes,
                     const ::istio::mixer::v1::CheckResponse& response) {
      if (on_response_) {
        status_ = on_response_(status, attributes, response);
      }
    }

   private:
    friend class CheckCache;
    // 检查的status，这个是需要缓存的检查结果.
    ::google::protobuf::util::Status status_;

    // 设置检查response的方法
    using OnResponseFunc = std::function<::google::protobuf::util::Status(
        const ::google::protobuf::util::Status&,
        const ::istio::mixer::v1::Attributes& attributes,
        const ::istio::mixer::v1::CheckResponse&)>;
    OnResponseFunc on_response_;
  };
```

具体看实现代码，先看构造函数：

```c++
// 初始化status为UNAVAILABLE
CheckCache::CheckResult::CheckResult() : status_(Code::UNAVAILABLE, "") {}
```

IsCacheHit()方法的逻辑非常简单，只检查status是不是UNAVAILABLE，也就是说只要设置了其它任何值，都视为命中缓存：

```c++
bool CheckCache::CheckResult::IsCacheHit() const {
  return status_.error_code() != Code::UNAVAILABLE;
}
```

## 类CheckCache.CacheElem

每个缓存对象的定义：

```c++
  class CacheElem {
   public:
    CacheElem(const CheckCache& parent,
              const ::istio::mixer::v1::CheckResponse& response, Tick time)
        : parent_(parent) {
      SetResponse(response, time);
    }

    // 设置response
    void SetResponse(const ::istio::mixer::v1::CheckResponse& response,
                     Tick time_now);

    // 检查缓存项是否过期.
    bool IsExpired(Tick time_now);

    // getter方法，用来将response转化为protobuf的status.
    ::google::protobuf::util::Status status() const { return status_; }

   private:
    // 到关联的缓存父对象，构造函数传入.
    const CheckCache& parent_;
    // 最后检查请求的检查状态.
    ::google::protobuf::util::Status status_;
    // 过期时间，缓存项在过期之后就不能再使用了.
    std::chrono::time_point<std::chrono::system_clock> expire_time_;
    // 使用次数。
    // 如果为 -1, 则不检查使用次数。
    // 如果为 0, 缓存项不该再使用.
    // 每次请求，使用次数减一
    int use_count_;
  };
```

### SetResponse()方法

其中的关键方式是SetResponse()方法。这个SetResponse()方法是在Check方法被执行之后调用。

```protobuf
rpc Check(CheckRequest) returns (CheckResponse)
```

最好对照 mixer API 中CheckResponse的数据结构来看代码实现：

```c++
void CheckCache::CacheElem::CacheElem::SetResponse(
    const CheckResponse &response, Tick time_now) {
  // 检查response中是否含有前置检查的内容
  if (response.has_precondition()) {
    // 将response中的前置条件检查的status转换为protobuf的status
    // 这个protobuf的status才是真正要缓存的对象，保存起来
    status_ = parent_.ConvertRpcStatus(response.precondition().status());

    //  检查前置条件检查是否有有效期间的要求
    if (response.precondition().has_valid_duration()) {
      // 如果有设置有效期间，则据此设置缓存的过期时间
      expire_time_ = time_now + utils::ToMilliseonds(
                                    response.precondition().valid_duration());
    } else {
      // 如果没有设置有效期间，则永远不过期
      // 备注：如果设置了使用次数，则使用次数超过时还是会过期
      expire_time_ = time_point<system_clock>::max();
    }
    use_count_ = response.precondition().valid_use_count();
  } else {
    // 如果response没有前置检查的内容，则报错并设置当前缓存项为不可用＋过期
    status_ = Status(Code::INVALID_ARGUMENT,
                     "CheckResponse doesn't have PreconditionResult");
    use_count_ = 0;           // 0 表示这个缓存不能使用.
    expire_time_ = time_now;  // 立即过期.
  }
}
```

###IsExpired方法 

IsExpired方法用来检查缓存项是否过期，需要同时检查过期时间和使用次数：

```c++
bool CheckCache::CacheElem::CacheElem::IsExpired(Tick time_now) {
  // 检查过期时间和使用次数
  if (time_now > expire_time_ || use_count_ == 0) {
    // 注意use_count_为－1时不会进入这里
    return true;
  }
    
  // 如果没有过期，则使用次数减一
  if (use_count_ > 0) {
    --use_count_;
  }
  // 注意：如果use_count_一开始就设置为－1，则也会走到这里，返回false表示不过期
  // 也就是use_count_设置－1代表不做使用次数检查(就只看过期时间了)
  return false;
}
```

## 类CheckCache

```c++
// 缓存Mixer检查的调用结果.
// 这是接口是线程安全的。
class CheckCache {
 private:
  friend class CheckCacheTest;
  using Tick = std::chrono::time_point<std::chrono::system_clock>;

   // Key是属性的签名。Value是CacheElem对象.
  // 这是一个有最大数量的LRU缓存.
  // 当最大数量达到时，删除最老的项
  using CheckLRUCache = utils::SimpleLRUCache<std::string, CacheElem>;

  // 检查的配置项，从构造函数传入.
  CheckOptions options_;

  // Referenced map，以它们的hash为key
  std::unordered_map<std::string, Referenced> referenced_map_;

  // 用来守护cache_访问的Mutex;
  std::mutex cache_mutex_;

  // 缓存，将操作的签名映射到操作。
  // 不计算每个缓存项的准确定义的开销，简单的为每个配置项分配一个开销单元
  // 被mutex_守护, 除了和null比较.
  std::unique_ptr<CheckLRUCache> cache_;

  GOOGLE_DISALLOW_EVIL_CONSTRUCTORS(CheckCache);
```

### 构造函数

构造函数，根据传入的CheckOptions构造cache对象：

```c++
CheckCache::CheckCache(const CheckOptions &options) : options_(options) {
  if (options.num_entries > 0) {
    cache_.reset(new CheckLRUCache(options.num_entries));
  }
}

CheckCache::~CheckCache() {
  // 销毁时清理缓存
  FlushAll();
}
```

### Check()方法

看中关键的Check()方法：

```c++
  void Check(const ::istio::mixer::v1::Attributes& attributes,
               CheckResult* result);

  // 如果缓存不能处理这个检查，则返回NOT_FOUND,
  // 调用者将不得不发送请求给mixer.
  ::google::protobuf::util::Status Check(
      const ::istio::mixer::v1::Attributes& request, Tick time_now);
```

具体实现：

```c++
void CheckCache::Check(const Attributes &attributes, CheckResult *result) {
  // 先调用另外一个check方法，检查是否可以在缓存中直接拿到缓存的结果
  Status status = Check(attributes, system_clock::now());
  if (status.error_code() != Code::NOT_FOUND) {
    result->status_ = status;
  }

  result->on_response_ = [this](const Status &status,
                                const Attributes &attributes,
                                const CheckResponse &response) -> Status {
    if (!status.ok()) {
      if (options_.network_fail_open) {
        return Status::OK;
      } else {
        return status;
      }
    } else {
      return CacheResponse(attributes, response, system_clock::now());
    }
  };
}

Status CheckCache::Check(const Attributes &attributes, Tick time_now) {
  // 如果没有开启缓存
  if (!cache_) {
    // 通过返回NOT_FOUND, 调用者将发送请求到mixer服务器.
    return Status(Code::NOT_FOUND, "");
  }

  // 所有被缓存的引用属性都存放在referenced_map_中，
  // 由于请求请求众多，因此可能有很多中属性的组合被mixer处理然后结果缓存在这里
  // 检查时需要先计算签名，但是，签名的计算又必须和被保存的ReferencedAttributes关联才能计算
  // 因此这里的计算量会非常大，因为如果一个ReferencedAttributes没有匹配，就必须计算后续的
  // 直到匹配或者所有缓存的ReferencedAttributes都计算一遍完成
  for (const auto &it : referenced_map_) {
    // 根据attributes得到缓存的签名
    const Referenced &reference = it.second;
    std::string signature;
    if (!reference.Signature(attributes, "", &signature)) {
      continue;
    }

    std::lock_guard<std::mutex> lock(cache_mutex_);
    // 根据签名获取缓存的处理结果
    CheckLRUCache::ScopedLookup lookup(cache_.get(), signature);
    if (lookup.Found()) {
      // 如果找到了，还要继续检查缓存是否过期
      // 过期时间由response里面的数据定，mixer的response可以指定过期时间和使用次数
      CacheElem *elem = lookup.value();
      if (elem->IsExpired(time_now)) {
        // 如果发现已经过期，删除缓存返回NOT_FOUND要求发起对mixer的请求
        cache_->Remove(signature);
        return Status(Code::NOT_FOUND, "");
      }
      return elem->status();
    }
  }

  return Status(Code::NOT_FOUND, "");
}
```

总结说：

1. 缓存的key最终来自传入的Attributes
2. mixer通过response的设值可以指定缓存项的过期时间和使用次数

```c++
  // 缓存来自远程mixer调用的应答.
  // 返回从应答转换而来的status.
  ::google::protobuf::util::Status CacheResponse(
      const ::istio::mixer::v1::Attributes& attributes,
      const ::istio::mixer::v1::CheckResponse& response, Tick time_now);

  // 清除所有缓存的检查应答，清理所有缓存项
  // 通常在destructor中调用.
  ::google::protobuf::util::Status FlushAll();

  // 从 grpc status 转换到 protobuf status.
  ::google::protobuf::util::Status ConvertRpcStatus(
      const ::google::rpc::Status& status) const;
```

