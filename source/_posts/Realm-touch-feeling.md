---
title: Realm 触感记录
date: 2018-05-26 23:36:24
tags: [iOS]
---

简单地看了Realm的demo，感觉这个数据库在移动端给人的感觉是写着原生的代码，编写一个个对象就可以毫无感知地创建一个个数据库表。这里记录Realm数据库的创建和删除。


[Realm文档地址](https://realm.io/docs/swift/latest/#realms)

### 查看本地的数据库

打印地址：

``` swift
print(Realm.Configuration.defaultConfiguration.fileURL!)
```
<!--more-->

或者在断点的时候：

``` lldb
Objective-C:
(lldb) po [RLMRealmConfiguration defaultConfiguration].fileURL

Swift using Realm Objective-C:
(lldb) po RLMRealmConfiguration.defaultConfiguration().fileURL

Swift using Realm Swift:
(lldb) po Realm.Configuration.defaultConfiguration.fileURL

Or if you have an RLMRealm instance at hand, you can use:
(lldb) po myRealm.configuration.fileURL
```


[stackoverflow: How to find my realm file?](https://stackoverflow.com/questions/28465706/how-to-find-my-realm-file/28465803#28465803)


### 创建多个数据库

App登录不同的用户可以创建一个每个用户独立的数据库文件。


``` swift
//create default Realm.

let realm = try! Realm()


//create configured Realm

let config = Realm.Configuration(
    // Get the URL to the bundled file
    fileURL: Bundle.main.url(forResource: "MyBundledData", withExtension: "realm"),
    // Open the file in read-only mode as application bundles are not writeable
    readOnly: true)

// Open the Realm with the configuration
let realm = try! Realm(configuration: config)
```

创建的地址默认都是在`Documents`文件夹下，初始化的时候指定了`fileUrl`需要保证能有写的权限，像上面例子中在`Bundle`中的数据库，它将是只读的，需要在打开的时候带上`readOnly`，如果需要写的话就复制到沙盒中使用。

### 创建存储在内存中的数据库

``` swift
let realm = try! Realm(configuration: Realm.Configuration(inMemoryIdentifier: "MyInMemoryRealm"))
```

### 创建远程同步的数据库

``` swift
// Create the configuration
let syncServerURL = URL(string: "realm://localhost:9080/~/userRealm")!
let config = Realm.Configuration(syncConfiguration: SyncConfiguration(user: user, realmURL: syncServerURL))

// Open the remote Realm
let realm = try! Realm(configuration: config)
// Any changes made to this Realm will be synced across all devices!
```

如果打开一个数据库需要伴随着耗时的操作，或者打开远程数据库是只读属性的，那么需要使用`asyncOpen`API

``` swift
let config = Realm.Configuration(schemaVersion: 1, migrationBlock: { migration, oldSchemaVersion in
    // potentially lengthy data migration
})
Realm.asyncOpen(configuration: config) { realm, error in
    if let realm = realm {
        // Realm successfully opened, with migration applied on background thread
    } else if let error = error {
        // Handle error that occurred while opening the Realm
    }
}
```

``` swift
let config = Realm.Configuration(syncConfiguration: SyncConfiguration(user: user, realmURL: realmURL))
Realm.asyncOpen(configuration: config) { realm, error in
    if let realm = realm {
        // Realm successfully opened, with all remote data available
    } else if let error = error {
        // Handle error that occurred while opening or downloading the contents of the Realm
    }
}

```

### 创建只能存贮指定类型数据的数据库

下面这个数据库中就能存储`MyClass`和`MyOtherClass`

``` swift
let config = Realm.Configuration(objectTypes: [MyClass.self, MyOtherClass.self])
let realm = try! Realm(configuration: config)
```

### 彻底删除一个本地数据库

再删除数据库之前，需要保证所有被Realm持有的对象都被释放了，包括所有从Realm数据库中读取的和添加到数据库中的对象。在删除realm文件的时候也需要删除其他的附属文件。

``` swift
autoreleasepool {
    // all Realm usage here
}
let realmURL = Realm.Configuration.defaultConfiguration.fileURL!
let realmURLs = [
    realmURL,
    realmURL.appendingPathExtension("lock"),
    realmURL.appendingPathExtension("note"),
    realmURL.appendingPathExtension("management")
]
for URL in realmURLs {
    do {
        try FileManager.default.removeItem(at: URL)
    } catch {
        // handle error
    }
}
```



