---
date: 2018-09-29T21:00:00+08:00
title: referenced源码
weight: 823
menu:
  main:
    parent: "mixercache"
description : "介绍Istio中的全局字典"
---


源文件路径为：

- `proxy/src/istio/mixerclient/referenced.h`
- `proxy/src/istio/mixerclient/referenced.cc`

## 类Referenced

类Referenced用于存储mixer服务器使用的引用属性的对象：

```c++
// Mixer客户端缓存仅能在它的缓存中使用引用属性(包括Check缓存和Quota缓存).
class Referenced {
 private:
  struct AttributeRef {......};

  // 不应该出现的key.
  std::vector<AttributeRef> absence_keys_;

  // 应该出现并且值要精确匹配的key.
  std::vector<AttributeRef> exact_keys_;
};
```

定义了一个结构体AttributeRef，然后有两个vector分别保存可能缺席的key和应该精确匹配的key。

这个类可以认为是和Mixer API中定义的ReferencedAttributes类型，但是实现方式稍有不同。ReferencedAttributes是通过attributeMatches属性（类型是ReferencedAttributes.AttributeMatch数组）来详细描述每个属性的匹配情况。而Referenced做了大幅简化，不支持正则表达式，然后存储结构也简化为上述的两个vector，分别存储不应该出现的key和需要精确匹配的key。

## 结构体AttributeRef

```c++
// 持有一个属性和一个可能的map key的引用
  struct AttributeRef {
    // 属性名
    std::string name;
    // 属性的key，仅当属性是stringMap时使用
    std::string map_key;

    // operator让 vector<AttributeRef> 可以排序
    bool operator<(const AttributeRef &b) const {
      int cmp = name.compare(b.name);
      if (cmp == 0) {
        return map_key.compare(b.map_key) < 0;
      }

      return cmp < 0;
    };
  };
```

## 主要方法

### Fill()方法

```c++
  // 用来自Check应答的protobuf填充对象.
  // 如果有属性名无法从客户端全局字典中解码，则返回false
  bool Fill(const ::istio::mixer::v1::Attributes &attributes,
            const ::istio::mixer::v1::ReferencedAttributes &reference);
```

其中参数reference是来自mixer返回的Check调用的应答，对应mixer API中的类型ReferencedAttributes。表示这些属性是在mixer产生应答的处理过程中涉及到的属性列表，构建客户端的缓存时，判断依据就是这些引用属性。也就是说，只要请求的属性匹配这些引用属性，就可以从缓存中获取结果从而跳过mixer的调用。

参数attributes是当前请求的所有属性的列表，从Check／CacheResponse方法中一路传递进来。

代码具体实现：

```c++
bool Referenced::Fill(const Attributes &attributes,
                      const ReferencedAttributes &reference) {
  // 获取全局字典，具体实现见后面章节
  const std::vector<std::string> &global_words = GetGlobalWords();
  const auto &attributes_map = attributes.attributes();

  // attributeMatches描述了这些引用属性中每个属性的具体匹配情况
  // 这里游历attributeMatches，根据每个属性的匹配情况进行处理
  for (const auto &match : reference.attribute_matches()) {
    AttributeRef ar;
    // 解码出属性的name，必须解码成功，否则视为整个fill操作失败
    // 注意这里match的name属性是一个int32值，不是string，因为这里使用的是索引
    if (!Decode(match.name(), global_words, reference, &ar.name)) {
      return false;
    }

    // 注意mixer返回的ReferencedAttributes信息中对于属性只给出了name(还是以索引形式)
    // 因此要得到这个属性的具体情况(如类型)，还需要在列表中根据名字查找一遍
    const auto it = attributes_map.find(ar.name);
    if (it != attributes_map.end()) {
      // 如果找到了这个属性，我们来继续看它的value
      const Attributes_AttributeValue &value = it->second;
      // 因为如果这个属性的值类型是stringMap，则需要额外的处理
      if (value.value_case() == Attributes_AttributeValue::kStringMapValue) {
        // 这里解码一下mapkey的值，必须解码成功，否则视为整个fill操作失败
        if (!Decode(match.map_key(), global_words, reference, &ar.map_key)) {
          return false;
        }
      }
    }

    // 然后根据mixer返回的具体匹配条件来处理
    if (match.condition() == ReferencedAttributes::ABSENCE) {
      // ABSENCE表示属性不存在时匹配，因此应该存放在absence_keys_中
      absence_keys_.push_back(ar);
    } else if (match.condition() == ReferencedAttributes::EXACT) {
      //EXACT表示属性存在并且值精确匹配时匹配，因此应该存放在exact_keys_中
      exact_keys_.push_back(ar);
    } else if (match.condition() == ReferencedAttributes::REGEX) {
      // Don't support REGEX yet, return false to no caching the response.
      GOOGLE_LOG(ERROR) << "Received REGEX in ReferencedAttributes for "
                        << ar.name;
      return false;
    }
  }

  std::sort(absence_keys_.begin(), absence_keys_.end());
  std::sort(exact_keys_.begin(), exact_keys_.end());

  return true;
}
```

其中调用的Decode方法的实现：

```c++
// Decode方法将index转为对应的字符串，需要使用全局和本地词语列表
// 如果解码失败则返回false
// 如果解码成功，则str参数将携带回解码得到的属性词汇的字符串表示
bool Decode(int idx, const std::vector<std::string> &global_words,
            const ReferencedAttributes &reference, std::string *str) {
  // 参数idx表示名称在字典中的索引。
  // 如果为正整数，表示索引到全局部署级字典
  if (idx >= 0) {
    if ((unsigned int)idx >= global_words.size()) {
      GOOGLE_LOG(ERROR) << "Global word index is too big: " << idx
                        << " >= " << global_words.size();
      return false;
    }
    *str = global_words[idx];
  } else {
    // 如果为负数，表示索引到消息级别的字典。
    // 消息级别的字典由ReferencedAttributes的words字段在Check调用的应答中带回来
    // 计算方式为（即－1为第一个值，对应到下标0，－2对应到下标1）:
    //    per_message_idx = -(array_idx + 1)
    idx = -idx - 1;
    if (idx >= reference.words_size()) {
      GOOGLE_LOG(ERROR) << "Per message word index is too big: " << idx
                        << " >= " << reference.words_size();
      return false;
    }
    *str = reference.words(idx);
  }

  return true;
}
```

> 注意：要看懂这段代码，务必把 [mixer v1 API](http://istio.doczh.cn/docs/reference/api/istio.mixer.v1.html) 的细节读一遍，了解属性，字典，压缩属性等概念和具体的请求／应答消息格式。

## Signature方法

```c++
  // 计算属性的缓存签名.
  // 如果属性不匹配则返回false，比如 "absence" 属性出现了
  // 或者 "exact" 匹配属性没有出现.
  bool Signature(const ::istio::mixer::v1::Attributes &attributes,
                 const std::string &extra_key, std::string *signature) const;
```

参数attributes是本地请求的属性列表，signature用来传递在匹配成功时计算好的签名。

Signature方法的具体实现：

```c++
bool Referenced::Signature(const Attributes &attributes,
                           const std::string &extra_key,
                           std::string *signature) const {
  // 要检查的属性，以map格式出现
  const auto &attributes_map = attributes.attributes();

  // 逐个检查absence key是否在本地请求的属性列表中出现
  for (std::size_t i = 0; i < absence_keys_.size(); ++i) {
    const auto &key = absence_keys_[i];
    const auto it = attributes_map.find(key.name);
    if (it == attributes_map.end()) {
      // 没出现就跳过，继续下一个absence key的检查
      continue;
    }

    // 如果属性出现了，那么就需要判断一下value，因为value是stringMap时还有匹配的可能
    const Attributes_AttributeValue &value = it->second;
    // 如果"absence"的key存在，而且这个属性还不是stringMap，返回false表示没有匹配到
    if (value.value_case() != Attributes_AttributeValue::kStringMapValue) {
      return false;
    }

    // value是stringMap，此时要匹配，要求map_key对应的值在这个stringMap中不存在
    // 这里特别注意，如果一个属性的值是stringMap，而这个stringMap中的多个key被应用
    // 则回出现多个absence key。它们有相同的name，不同的mapKey。
    // 举例：如果有属性label＝{"a":"1", "b":2, "c":3}，其中的a和b被引用了
    // 则absence_keys_中会保存两个key:
    // 1. {name: "label", mapkey:"a"} 
    // 2. {name: "label", mapkey:"b"}
    std::string map_key = key.map_key;
    const auto &smap = value.string_map_value().entries();
    // 因为 absence_keys_ are 是按照key.name排序的,
    // 所以可以继续处理下一个key的stringMaps，知道发现新的name
    // 即处理完{name: "label", mapkey:"a"} 之后，可以直接看下一个absence key
    // 即 {name: "label", mapkey:"b"}。 
    do {
      // 如果发现stringMap里面有mapkey对应的值，则违反了 "absence" 的约束.
      if (smap.find(map_key) != smap.end()) {
        return false;
      }
      // 直接跳到下一个mapkey，如果它还是当前这个name
      // 这算是特别的性能优化了，这是代码很晦涩，如果不是对此处的数据结构足够了解，很难看懂
      // break loop if at the end or keyname changes.
      if (i + 1 == absence_keys_.size() ||
          absence_keys_[i + 1].name != key.name) {
        break;
      }
	  // 更新mapkey为下一个absence key的mapkey
      map_key = absence_keys_[++i].map_key;

    } while (true);
  }
  // absence key的检查到此接触，代码能走到这里，说明absence key的约束没有违反

  utils::MD5 hasher;
  // 继续extac key的检查，边检查边计算hash值
  // 逐个检查exact key是否在属性中出现，并更新hash值
  for (std::size_t i = 0; i < exact_keys_.size(); ++i) {
    const auto &key = exact_keys_[i];
    const auto it = attributes_map.find(key.name);
    // 如果"exact" key在当前属性中没有出现，返回false表示匹配失败.
    if (it == attributes_map.end()) {
      return false;
    }

    hasher.Update(it->first);
    hasher.Update(kDelimiter, kDelimiterLength);

    // 走到这里说明要求的exact key在当前属性列表中出现了，此时就需要取出这个属性的value
    // 然后根据value的类型进行hash的处理
    const Attributes_AttributeValue &value = it->second;
    switch (value.value_case()) {
      case Attributes_AttributeValue::kStringValue:
        hasher.Update(value.string_value());
        break;
      case Attributes_AttributeValue::kBytesValue:
        hasher.Update(value.bytes_value());
        break;
      case Attributes_AttributeValue::kInt64Value: {
        auto data = value.int64_value();
        hasher.Update(&data, sizeof(data));
      } break;
      case Attributes_AttributeValue::kDoubleValue: {
        auto data = value.double_value();
        hasher.Update(&data, sizeof(data));
      } break;
      case Attributes_AttributeValue::kBoolValue: {
        auto data = value.bool_value();
        hasher.Update(&data, sizeof(data));
      } break;
      case Attributes_AttributeValue::kTimestampValue: {
        auto seconds = value.timestamp_value().seconds();
        auto nanos = value.timestamp_value().nanos();
        hasher.Update(&seconds, sizeof(seconds));
        hasher.Update(kDelimiter, kDelimiterLength);
        hasher.Update(&nanos, sizeof(nanos));
      } break;
      case Attributes_AttributeValue::kDurationValue: {
        auto seconds = value.duration_value().seconds();
        auto nanos = value.duration_value().nanos();
        hasher.Update(&seconds, sizeof(seconds));
        hasher.Update(kDelimiter, kDelimiterLength);
        hasher.Update(&nanos, sizeof(nanos));
      } break;
      case Attributes_AttributeValue::kStringMapValue: {
        // 如果属性的值为stringMap格式，依然要麻烦一些，但是原理和上面类似
        std::string map_key = key.map_key;
        const auto &smap = value.string_map_value().entries();
        // Since exact_keys_ are sorted by key.name,
        // continue processing stringMaps until a new name is found.
        do {
          const auto sub_it = smap.find(map_key);
          // exact match of map_key is missing
          if (sub_it == smap.end()) {
            return false;
          }

          hasher.Update(sub_it->first);
          hasher.Update(kDelimiter, kDelimiterLength);
          hasher.Update(sub_it->second);
          hasher.Update(kDelimiter, kDelimiterLength);

          // break loop if at the end or keyname changes.
          if (i + 1 == exact_keys_.size() ||
              exact_keys_[i + 1].name != key.name) {
            break;
          }

          map_key = exact_keys_[++i].map_key;
        } while (true);
      } break;
      case Attributes_AttributeValue::VALUE_NOT_SET:
        break;
    }
    hasher.Update(kDelimiter, kDelimiterLength);
  }
  hasher.Update(extra_key);
    
  // 代码能走到这里，说明匹配的两个约束都满足了
  // 1. absence key要求的不能出现的key都没有出现
  // 2. extace key要求的必须出现的key都出现了
  
  // 同时hasher也收集到了足够的数据，最后计算签名
  *signature = hasher.Digest();
  return true;
}
```

### Hash()方法

hash()方法用来返回一个hash值，用来表示当前这个referenced实例

```c++
std::string Hash() const;
```

实现比较简单，referenced实例的状态就只在absenc key和exact key两个vector，计算一下hash就好了：

```c++
std::string Referenced::Hash() const {
  utils::MD5 hasher;

  // keys are sorted during Fill
  UpdateHash(absence_keys_, &hasher);
  hasher.Update(kWordDelimiter);ß
  UpdateHash(exact_keys_, &hasher);

  return hasher.Digest();
}
```

