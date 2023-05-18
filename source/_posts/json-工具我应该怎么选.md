---
title: json 工具我应该怎么选？
date: 2023-05-18 23:25:00
categories:
- tools
tags:
- tools
---

编程多年，其实适合项目/自己/团队才是合适的。JSON 在 B/S 应用下，作为轻量级的数据交换方式。也应运而生不少序列化反序列化的 JSON 工具包。
比如，json-lib、fastjson、gson、jackson 等。我用过的，主要是这几个就说说我的选择场景及依据。

### json-lib

这是我**被迫**使用的 JSON 工具包。不是主动选择，也是因为确实觉得很坑。**首先**，序列化和反序列化耗时很慢。**其次**，`JSONObject.getXXX(key)` JSONObject
的 get 系方法都必须要求 key 必须在 JSON 串中且非空。也就是被使用的 key 都必须是非空值，所以就逼迫设置默认值。但这样可能会导致业务被"逼不得已"的默认值覆盖。**第三**，`JSONObject.put(key,value)` 方法会针对 value[ 类型为 String 且符合 JSON 格式时，会将 value 反序列化。这个特性看似很好，其实在特定场景下，给人误导，
以为 value 是 JSONObject 结果发现是 String。Oh my god ！！总结，坑比较多，不建议使用。]()

### fastjson

阿里巴巴出品，使用是很简单，基于类属性 get/set 来序列化和非序列化。包括也支持很多选项 `FastJsonConfig` 、 `SerializerFeature`。对研发来说，序列化反序列化使用
起来也很简单，性能也偏好。以为应该是很受欢迎，但 github 上 issue 很多。传说中源码写的不好，也没什么注释。所以，不是最受欢迎，在中国很有地位。

### jackson

被 spring-web 用作默认的 `application/json` 序列化和反序列化 JSON 选型。依赖的 jar 包少，提供扩展选项/高级功能很多。对我而言，相比较 fastjson 而言，使用
不方便。对于 String 转 Date 时，必须指定 `@JsonProperty(format='xxxx')` 格式化字段。

### gson

google 推出 JSON 工具包，符合 JSON 格式定义，是依据属性进行序列化和反序列化，包含 public、private、protected 修饰字段。转 List、Set 集合类，相比 fastjson 支持
的很好。但是性能相比而言会弱一些。

### **如果需要功能完善，易上手建议 fastjson/gson。jackson 就结合 spring-web 使用。**

