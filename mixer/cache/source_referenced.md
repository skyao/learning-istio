# referenced源码

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

  // 可能缺席的key.
  std::vector<AttributeRef> absence_keys_;

  // 应该精确匹配的key.
  std::vector<AttributeRef> exact_keys_;
};
```

定义了一个结构体AttributeRef，然后有两个vector分别保存可能缺席的key和应该精确匹配的key。

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

### 方法Fill()

```c++
  // 用来自Check应答的protobuf填充对象.
  // 如果有属性名无法从客户端全局字典中解码，则返回false
  bool Fill(const ::istio::mixer::v1::Attributes &attributes,
            const ::istio::mixer::v1::ReferencedAttributes &reference);
```

代码具体实现：

```c++
bool Referenced::Fill(const Attributes &attributes,
                      const ReferencedAttributes &reference) {
  // 获取全局字典，具体实现见后面章节
  const std::vector<std::string> &global_words = GetGlobalWords();
  const auto &attributes_map = attributes.attributes();

  for (const auto &match : reference.attribute_matches()) {
    AttributeRef ar;
    // 解码出属性的name
    if (!Decode(match.name(), global_words, reference, &ar.name)) {
      return false;
    }

    // 再尝试解码属性的map_key
    const auto it = attributes_map.find(ar.name);
    if (it != attributes_map.end()) {
      const Attributes_AttributeValue &value = it->second;
      if (value.value_case() == Attributes_AttributeValue::kStringMapValue) {
        if (!Decode(match.map_key(), global_words, reference, &ar.map_key)) {
          return false;
        }
      }
    }

    if (match.condition() == ReferencedAttributes::ABSENCE) {
      absence_keys_.push_back(ar);
    } else if (match.condition() == ReferencedAttributes::EXACT) {
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
// Decode dereferences index into str using global and local word lists.
// Decode returns false if it is unable to Decode.
bool Decode(int idx, const std::vector<std::string> &global_words,
            const ReferencedAttributes &reference, std::string *str) {
  if (idx >= 0) {
    if ((unsigned int)idx >= global_words.size()) {
      GOOGLE_LOG(ERROR) << "Global word index is too big: " << idx
                        << " >= " << global_words.size();
      return false;
    }
    *str = global_words[idx];
  } else {
    // per-message index is negative, its format is:
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

TBD: 需要把mixer v1 API的细节读一遍。

## 方法Signature

```c++
  // 计算属性的缓存签名.
  // 如果属性不匹配则返回false，比如 "absence" 属性出现了
  // 或者 "exact" 匹配属性没有出现.
  bool Signature(const ::istio::mixer::v1::Attributes &attributes,
                 const std::string &extra_key, std::string *signature) const;
```

Signature方法的具体实现：

```c++
bool Referenced::Signature(const Attributes &attributes,
                           const std::string &extra_key,
                           std::string *signature) const {
  // 要检查的属性，以map格式出现
  const auto &attributes_map = attributes.attributes();

  // 逐个检查absence key是否在属性中出现
  for (std::size_t i = 0; i < absence_keys_.size(); ++i) {
    const auto &key = absence_keys_[i];
    const auto it = attributes_map.find(key.name);
    if (it == attributes_map.end()) {
      continue;
    }

    const Attributes_AttributeValue &value = it->second;
    // 如果"absence"的key存在，而且这个属性还不适stringMap，返回false表示没有匹配到
    if (value.value_case() != Attributes_AttributeValue::kStringMapValue) {
      return false;
    }

    std::string map_key = key.map_key;
    const auto &smap = value.string_map_value().entries();
    // Since absence_keys_ are sorted by key.name,
    // continue processing stringMaps until a new name is found.
    do {
      // if subkey is found, it is a violation of "absence" constrain.
      if (smap.find(map_key) != smap.end()) {
        return false;
      }
      // break loop if at the end or keyname changes.
      if (i + 1 == absence_keys_.size() ||
          absence_keys_[i + 1].name != key.name) {
        break;
      }

      map_key = absence_keys_[++i].map_key;

    } while (true);
  }

  utils::MD5 hasher;

  // 逐个检查exact key是否在属性中出现，并更新hash值
  for (std::size_t i = 0; i < exact_keys_.size(); ++i) {
    const auto &key = exact_keys_[i];
    const auto it = attributes_map.find(key.name);
    // 如果有"exact" 属性没有出现，返回false表示匹配失败.
    if (it == attributes_map.end()) {
      return false;
    }

    hasher.Update(it->first);
    hasher.Update(kDelimiter, kDelimiterLength);

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

  // 最后计算签名
  *signature = hasher.Digest();
  return true;
}
```

