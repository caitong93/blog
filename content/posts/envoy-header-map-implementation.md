---
title: "Envoy HeaderMap Implementation in 1.17"
date: 2021-04-01T18:48:18+08:00
draft: true
---

## Intro

在将内部 Envoy 工程适配上游 1.17 的过程中，我发现原来使用 `HeaderMap` 的地方都换成了 `RequestHeaderMap`, `ResponseHeaderMap`。`HeaderMap` 仍存在，成为了后者的 base class。Envoy 对标准或者常用的 header 做了优化，定义了一个 inline header 集合，对这些 header 的操作是 O(1) 开销，内存开销也更少。在 1.17 中，优化更进一步，`RequestHeaderMap` 和 `ResponseHeaderMap` 维护独立的 inline header 数据结构。

`HeaderMap` 的主要实现仍然在`HeaderMapImpl`上，`RequestHeaderMap` 和 `ResponseHeaderMap` 只负责声明各自的 inline headers。

```cpp

class HeaderMapImpl {
public:
  // ...

  virtual HeaderEntryImpl** inlineHeaders() PURE;
}

class class RequestHeaderMapImpl final: public TypedHeaderMapImpl<RequestHeaderMap>, public InlineStorage {
public:
  // ...
protected:
  HeaderEntryImpl** inlineHeaders() override { return inline_headers_; }

private:
  HeaderEntryImpl* inline_headers_[];
}
```

`HeaderMapImpl` 通过虚函数 `inlineHeaders` 在运行时获取到实际的 inline headers。

从 `inline_headers_` 的定义，已经可以猜到每个 `HeaderMapImpl` 对象只存储 inline headers 的 value，而 key 被映射成了 `inline_headers_` 的下标。

那么如何调用方法和 inline header 交互 ?`HeaderMap`内部又是如何处理 inline header 的呢 ? 这里以一个具体的调用来看一下。

## Inline headers

```cpp
// 将 inline header :path 设置为 /v1/apples
request.headers().setPath("/v1/apples");
```

`RequestHeaderMap` 的 `setPath` 方法实现如下:

```cpp

class RequestHeaderMapImpl {
public:
  void setPath(absl::string_view value) override { setInline(HeaderHandles::get().Path, value); }

private:
  struct HeaderHandleValues {
    INLINE_RESP_HEADERS(DEFINE_HEADER_HANDLE)
    INLINE_REQ_RESP_HEADERS(DEFINE_HEADER_HANDLE)
    INLINE_RESP_HEADERS_TRAILERS(DEFINE_HEADER_HANDLE)
  };

  using HeaderHandles = ConstSingleton<HeaderHandleValues>;
}
```

`HeaderHandles` 是一个单例，也就是说全局只存储一份，它存储的内容是 `HeaderHandleValues`。而 `HeaderHandleValues` 包含了 inline headers 的 handle, handle 中
包含 header key 对应的 index, 即 `inline_headers_` 的下标。
对于 `:path` 来说，有一个全局的 `static HeaderHandleValues`，其中包含 `Handle<RequestHeaderMapImpl> Path` 成员。
所以 `HeaderHandles::get().Path` 就是拿到这个 handle。`setInline` 要做的事情也很简单，就是调用 `inlineHeaders` 获取到 `inline_headers_`, 然后通过 handle 中的下标拿到对应的 entry 并设置 value。

```cpp
#define DEFINE_HEADER_HANDLE(name)                                                                 \
  Handle name =                                                                                    \
      CustomInlineHeaderRegistry::getInlineHeader<header_map_type>(Headers::get().name).value();

```

`Handle<RequestHeaderMapImpl> Path` 是被上面的宏所定义的。`CustomInlineHeaderRegistry` 和 `HeaderHandles` 一样，也是一个 Singleton, 通过模版变量可以将 `RequestHeaderMapImpl` 和
`ResponseHeaderMapImpl` 的 registry 区分开。

```cpp
class CustomInlineHeaderRegistry {
public:
  enum class Type { RequestHeaders, RequestTrailers, ResponseHeaders, ResponseTrailers };
  using RegistrationMap = std::map<LowerCaseString, size_t>;
};
```

`CustomInlineHeaderRegistry` 中用于存储 inline header 的结构为 `std::map<LowerCaseString, size_t>`, key 为 inline header 的 lower case 表示, value 为整型 index。

因为 `CustomInlineHeaderRegistry` 定义在 `header_map.h` 中，所以只要引用这个模块，插件代码也可以注册自定义的 inline header。

```cpp
/**
 * These are definitions of all of the inline header access functions described inside header_map.h
 */
#define DEFINE_INLINE_HEADER_FUNCS(name)                                                           \
public:                                                                                            \
  const HeaderEntry* name() const override { return getInline(HeaderHandles::get().name); }        \
  void append##name(absl::string_view data, absl::string_view delimiter) override {                \
    appendInline(HeaderHandles::get().name, data, delimiter);                                      \
  }                                                                                                \
  void setReference##name(absl::string_view value) override {                                      \
    setReferenceInline(HeaderHandles::get().name, value);                                          \
  }                                                                                                \
  void set##name(absl::string_view value) override {                                               \
    setInline(HeaderHandles::get().name, value);                                                   \
  }                                                                                                \
  void set##name(uint64_t value) override { setInline(HeaderHandles::get().name, value); }         \
  size_t remove##name() override { return removeInline(HeaderHandles::get().name); }               \
  absl::string_view get##name##Value() const override {                                            \
    return getInlineValue(HeaderHandles::get().name);                                              \
  }
```

然后回过头来看 `DEFINE_INLINE_HEADER_FUNCS` 这个宏, 它的作用是声明 inline headers 的相关方法。

但更多的场景下，header 是动态获取的，比如通过 header 做路由匹配。

```cpp
HeaderMap::GetResult HeaderMapImpl::get(const LowerCaseString& key) const
```

例如，当需要动态获取一个 header 时，可以用上面的方法。

```cpp
HeaderMap::NonConstGetResult HeaderMapImpl::getExisting(const LowerCaseString& key) {
  HeaderMap::NonConstGetResult ret;
  auto lookup = staticLookup(key.get());
  if (lookup.has_value()) {
    if (*lookup.value().entry_ != nullptr) {
      ret.push_back(*lookup.value().entry_);
    }
    return ret;
  }

  if (headers_.maybeMakeMap()) {
    HeaderList::HeaderLazyMap::iterator iter = headers_.mapFind(key.get());
    if (iter != headers_.mapEnd()) {
      const HeaderList::HeaderNodeVector& v = iter->second;
      for (const auto& values_it : v) {
        // Convert the iterated value to a HeaderEntry*.
        ret.push_back(&(*values_it));
      }
    }
    return ret;
  }

  for (HeaderEntryImpl& header : headers_) {
    if (header.key() == key.get().c_str()) {
      ret.push_back(&header);
    }
  }

  return ret;
}

```

`getExisting` 按顺序查找 header:
- 首先查找 inline headers
- 然后在 lazy_map 中找 
- 最后在 header list 中找

中间这以及通常是没有的，除非显示配置了 `envoy.http.headermap.lazy_map_min_size` runtime feature, 它的作用是当 header list 的大小超过 lazy_map_min_size 时，将 list 中
的 header 插入到 map 中, 用内存换取查找效率。

## TODO

`HeaderMap` 出于性能考虑，还使用了很多特殊的数据结构，例如 `HeaderString`, `StaticLookupTable`, 有时间再写