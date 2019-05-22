---
title: Swift Codable体验
date: 2019-03-21 15:38:38
tags: [iOS, Swift]
---

>“Hey bro！Json数据转模型，用啥？”
“emmm... 最常用的 MJExtension , YYModel呗”
“What about struct Type Model in Swift?”
“额，OC里都是依赖Runtime的转的，这值类型...”
“Swift 自带的Codable. Take a look”

<!-- more -->

Swift 由于类型安全的特性，对于像 JSON 这类弱类型的数据处理一直是一个比较头疼的问题，`ObjectMapper` `SwiftyJSON` 和 `HandyJSON` 都做了这样的努力，但都还是有所缺陷。 所以 Codable 协议的推出，拯救了这样的局面，也会对这些三方库提供了些新的思路吧，毕竟是苹果自己造的，拥有从底层改动的权限。

Codable 其实是一个组合协议，由 Decodable 和 Encodable 两个协议组成：

``` swift
/// A type that can convert itself into and out of an external representation.
public typealias Codable = Decodable & Encodable

/// A type that can encode itself to an external representation.
public protocol Encodable {
    public func encode(to encoder: Encoder) throws
}

/// A type that can decode itself from an external representation.
public protocol Decodable {
    public init(from decoder: Decoder) throws
}
```

Encodable 和 Decodable 分别定义了 encode(to:) 和 init(from:) 两个协议函数，分别用来实现数据模型的归档和外部数据的解析和实例化。

__现在我们用codable来解决工作重常遇到的情况__

### 处理服务端返回字段和本地字段名称不一致情况

就举一个将服务端下划线命名的字段赋值给客户端驼峰命名的字段的例子。

OC中的实现(使用`MJExtension`)：

``` objectivec
@interface JakeyObj : NSObject
@property (nonatomic, copy) NSString *countNum;
@end

@implementation JakeyObj
+ (NSDictionary *)mj_replacedKeyFromPropertyName {
    return @{@"countNum" : @"count_num"};
}
@end
```

使用Swift的Codable来实现，得先了解下Codable的内部实现原理。

关于Codable内部的具体实现 摘自参考[[1]](#ref)：
>通过研究底层的 C++ 源代码可以发现，Codable 通过巧（kai）妙（guà）的方式，在编译代码时根据类型的属性，自动生成了一个 `CodingKeys` 的枚举类型定义，这是一个以 String 类型作为原始值的枚举类型，对应每一个属性的名称。然后再给每一个声明实现 Codable 协议的类型自动生成 `init(from:)` 和 `encode(to:)` 两个函数的具体实现，最终完成了整个协议的实现。
>

了解Codable的内部实现之后，我们可以自己实现`CodingKeys`的类型定义， 并且给属性指定不同的原始值，实现自定义字段的解析，这样编译器不会给我们重新生成一个默认的`CodingKeys`

``` swift
struct JakeyObj: Codable {
    var name: String
    var countNum: Int
    
    enum CodingKeys: String, CodingKey {
        case name
        case countNum = "count_num"
    }
}
```

需要注意的是，即使属性名称与 JSON 中的字段名称一致，如果自定义了 CodingKeys，这些属性也是无法省略的【可选值类型属性可以省略】，否则会得到一个 Type 'JakeyObj' does not conform to protocol 'Codable' 的编译错误，这一点还是有点坑的。


Swift 4.1中对此进行了改进，只需要对JSONDecoder对象进行设置指定`keyDecodingStrategy`属性

``` Swift
let jsonData = Data(jsonString.utf8)

let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
```



### 处理服务端字段返回为空、不返回字段情况

将字段对应的属性标记为可选值类型即可。

``` swift
struct JakeyObj: Codable {
    var name: String?
}
```

如果定义非可选类型，服务端没有给值的话会Crash哦。
这样的话，模型不是都得使用可选类型才合理了，取值的时候会有点麻烦

### Date类型的转换
当然时间的处理可以通过字符串接收下来之后在使用前进行处理，但这难免会重复执行增加了CPU的额外负担。

看看如何在模型中一次性处理完成吧

`JSONDecoder`提供了一个枚举类型来处理日期格式

``` Swift
public enum DateDecodingStrategy {

    /// Defer to `Date` for decoding. This is the default strategy.
    case deferredToDate

    /// Decode the `Date` as a UNIX timestamp from a JSON number.
    case secondsSince1970

    /// Decode the `Date` as UNIX millisecond timestamp from a JSON number.
    case millisecondsSince1970

    /// Decode the `Date` as an ISO-8601-formatted string (in RFC 3339 format).
    case iso8601

    /// Decode the `Date` as a string parsed by the given formatter.
    case formatted(DateFormatter)

    /// Decode the `Date` as a custom value decoded by the given closure.
    case custom((Decoder) throws -> Date)
}
```

注释写得很清楚，每个类型的用途，我们来尝试下最后两个`.formatted(DateFormatter)`和`.custom((Decoder) throws -> Date)`的使用。

``` swift
let decoder = JSONDecoder()
let formatter = DateFormatter()
formatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
formatter.timeZone = TimeZone(abbreviation: "UTC")
decoder.dateDecodingStrategy = .formatted(formatter)
```

使用`.formatted(DateFormatter)`已经能满足基本上的需求的，特殊的实现就可以指定如何解析。

例子：

``` swift
let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .custom({ (decoder) -> Date in
    let container = try decoder.singleValueContainer()
    if let str = try? container.decode(String.self) {
        // 如有必要,这里还可以判断字符串是否为时间戳,最终转换成Date
        let formatter = DateFormatter()
        formatter.dateFormat = "yyyy-MM-dd HH:mm:ss"
        formatter.timeZone = TimeZone(abbreviation: "UTC")
        if let date = formatter.date(from: str) {
            return date
        }
    }
    if let double = try? container.decode(Double.self) {
        // 可根据服务器返回的时间戳是相对于1970.1.1 00:00:00还是2001.1.1 00:00:00进行相应的转换
        return Date(timeIntervalSinceReferenceDate: double)
    }
    throw DecodingError.dataCorruptedError(in: container, debugDescription: "Cannot decode date.")
})
```

更多对于`DateDecodingStrategy`的优化使用见参考[[4]](#ref)

### <span class="ref">参考</span>

[1]. [Swift 4 踩坑之 Codable 协议](https://juejin.im/post/5a3869bb5188257d167a4ebd#heading-12)
[2]. [谈谈MVX中的Model](https://juejin.im/entry/59603e696fb9a06ba6464d32)
[3]. [Swift 项目中涉及到 JSONDecoder...](https://ming1016.github.io/2018/04/02/record-and-think-about-swift-project-jsondecoder-networking-and-pop/)
[4]. [Swift4之Codable协议](http://www.0daybug.com/2018/03/26/swift4-Codable/)