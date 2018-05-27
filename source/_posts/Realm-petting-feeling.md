---
title: Realm 质感记录
date: 2018-05-27 11:49:56
tags: [iOS]
---

先来段代码欣赏下

``` swift
let myDog = Dog()
myDog.name = "Fido"
myDog.age = 1

try! realm.write {
    realm.add(myDog)
}

let myPuppy = realm.objects(Dog.self).filter("age == 1").first
try! realm.write {
    myPuppy!.age = 2
}

print("age of my dog: \(myDog.age)") // => 2

```
<!--more-->

### Models（数据库中的表）

#### 属性

>Realm supports the following property types: Bool, Int, Int8, Int16, Int32, Int64, Double, Float, String, Date, and Data.
>

__`CGFloat` properties are discouraged, as the type is not platform independent.__

>String, Date and Data properties can be optional. Object properties must be optional. Storing optional numbers is done using RealmOptional.
>

在Swift中存储对象必须使用optional，存储optional numbers使用RealmOptional

String, Data, Date 这三种类型的属性可是可选类型可也以不是

``` Swift
class Person: Object {
    // Optional string property, defaulting to nil
    @objc dynamic var name: String? = nil

    // Optional int property, defaulting to nil
    // RealmOptional properties should always be declared with `let`,
    // as assigning to them directly will not work as desired
    let age = RealmOptional<Int>()
}

let realm = try! Realm()
try! realm.write() {
    var person = realm.create(Person.self, value: ["Jane", 27])
    // Reading from or modifying a `RealmOptional` is done via the `value` property
    person.age.value = 28
}

```

#### 主键

主键一旦被添加了就不能被修改

``` swift
class Person: Object {
    @objc dynamic var id = 0
    @objc dynamic var name = ""

    override static func primaryKey() -> String? {
        return "id"
    }
}
```

#### 索引

非必要的时候不建议添加索引，这会增加数据库的大小。

``` swift
class Book: Object {
    @objc dynamic var price = 0
    @objc dynamic var title = ""

    override static func indexedProperties() -> [String] {
        return ["title"]
    }
}
```

索引支持 `string`, `integer`, `boolean`, `Date`类型的属性


#### Ignoring properties

不想要存储到Realm中的属性可以添加标记为`ignoredProperties`

``` swift
class Person: Object {
    @objc dynamic var tmpID = 0
    var name: String { // read-only properties are automatically ignored
        return "\(firstName) \(lastName)"
    }
    @objc dynamic var firstName = ""
    @objc dynamic var lastName = ""

    override static func ignoredProperties() -> [String] {
        return ["tmpID"]
    }
}
```

#### 属性修饰词


|Type|	Non-optional|	Optional|
|:-:|:-:|:-:|
|Bool	|@objc dynamic var value = false	|let value = RealmOptional<Bool>()|
|Int	|@objc dynamic var value = 0	|let value = RealmOptional<Int>()|
|Float	|@objc dynamic var value: Float = 0.0	|let value = RealmOptional<Float>()|
|Double|	@objc dynamic var value: Double = 0.0	|let value = RealmOptional<Double>()|
|String|	@objc dynamic var value = ""	|@objc dynamic var value: String? = nil|
|Data	|@objc dynamic var value = Data()	|@objc dynamic var value: Data? = nil|
|Date	|@objc dynamic var value = Date()	|@objc dynamic var value: Date? = nil|
|Object|	n/a: must be optional	|@objc dynamic var value: Class?=nil
|List	|let value = List<Type>()	|n/a: must be non-optional|
|LinkingObjects	|let value = LinkingObjects(fromType: Class.self, property: "property")	|n/a: must be non-optional|



---

### Relationships关系

连接数据库的两个对象（表），只需要通过`Object`和`List`这个两个属性就可以了。

`List`非常像`Array`，`List`中只能存在单种类型的对象。

#### Many-to-One 多对一

创建一对一或者多对一的关系时，只需要在模型中创建一个将要关联对象类型的对象。

``` swift
class Dog: Object {
    @objc dynamic var name = ""
}

class Dog: Object {
    // ... other property declarations
    @objc dynamic var owner: Person? // to-one relationships must be optional
}
```

Usage

``` swift
let jim = Person()
let rex = Dog()
rex.owner = jim
```

#### Many-to-Many 多对多

使用`List`属性创建关系

``` swift
class Person: Object {
    // ... other property declarations
    let dogs = List<Dog>()
}

let someDogs = realm.objects(Dog.self).filter("name contains 'Fido'")
jim.dogs.append(objectsIn: someDogs)
jim.dogs.append(rex)

```

#### Inverse relationships 逆关系

从Person可以找到Dog对象，但是不能够从Dog找到它的主人对象。除非是使用一对一的关系，但是这个关系是独立的，给Person对象添加一个新的Dog，并不会讲这个Dog对象中的owner属性设置到正确的Person。因此引入了逆关系

``` swift
class Dog: Object {
    @objc dynamic var name = ""
    @objc dynamic var age = 0
    let owners = LinkingObjects(fromType: Person.self, property: "dogs")
}
```

上面的Dog对象中的owners属性中将包含所有拥有这个Dog对象的Person对象。


---
### Notifications

#### Realm Notification

通知返回的是这个通知的token，一旦token被释放了，就意味着这个通知也被注销了。

一旦有相关的写操作被提交，无论在什么线程上的写操作，通知将会异步地被触发，此时可以进行UI的刷新。

``` swift
// Observe Realm Notifications
let token = realm.observe { notification, realm in
    viewController.updateUI()
}

// later
token.invalidate()

```

在通知的block中进行数据库的写操作将会抛出异常，可以使用`Realm.isInWriteTransaction`来区分当前是否处于写的事务中。

注意通知所在的线程将于注册通知时所处的线程相同。如果不是在主线程注册的通知，则需要给该线程启动一个run loop。因为通知的传递需要依靠run loop，当一个run loop中有其他的任务在执行，那么realm的通知可能会被延迟，因此会有多个写操作事务合并到同一个通知中。



#### Collection Notification

Collection的通知在objects添加、删除、修改的时候都会触发，但不会接收整个realm，只会接收改变的部分数据。通过Block的参数`RealmCollectionChange`可以获取到改变的值，包括删除、插入、修改影响到的索引位置。

一个对象的属性发生变化、属性List中的对象发生变化，不管是对多还是对一的关系，都会触发通知。

``` swift
class Dog: Object {
    @objc dynamic var name = ""
    @objc dynamic var age = 0
}

class Person: Object {
    @objc dynamic var name = ""
    let dogs = List<Dog>()
}
```

``` swift
class ViewController: UITableViewController {
    var notificationToken: NotificationToken? = nil

    override func viewDidLoad() {
        super.viewDidLoad()
        let realm = try! Realm()
        let results = realm.objects(Person.self).filter("age > 5")

        // Observe Results Notifications
        notificationToken = results.observe { [weak self] (changes: RealmCollectionChange) in
            guard let tableView = self?.tableView else { return }
            switch changes {
            case .initial:
                // Results are now populated and can be accessed without blocking the UI
                tableView.reloadData()
            case .update(_, let deletions, let insertions, let modifications):
                // Query results have changed, so apply them to the UITableView
                tableView.beginUpdates()
                tableView.insertRows(at: insertions.map({ IndexPath(row: $0, section: 0) }),
                                     with: .automatic)
                tableView.deleteRows(at: deletions.map({ IndexPath(row: $0, section: 0)}),
                                     with: .automatic)
                tableView.reloadRows(at: modifications.map({ IndexPath(row: $0, section: 0) }),
                                     with: .automatic)
                tableView.endUpdates()
            case .error(let error):
                // An error occurred while opening the Realm file on the background worker thread
                fatalError("\(error)")
            }
        }
    }

    deinit {
        notificationToken?.invalidate()
    }
}
```
如此就可以根据具体需要刷新的位置来局部刷新列表，不需要整个重新地刷新。

#### Object Notifications

可以监听对象，当对象被删除或者对象的内容改变的时候会触发通知。修改对象属性的时候触发通知的参数会返回对象的属性数组，包含每个属性的旧值和新值

``` swift
class StepCounter: Object {
    @objc dynamic var steps = 0
}

let stepCounter = StepCounter()
let realm = try! Realm()
try! realm.write {
    realm.add(stepCounter)
}
var token : NotificationToken?
token = stepCounter.observe { change in
    switch change {
    case .change(let properties):
        for property in properties {
            if property.name == "steps" && property.newValue as! Int > 1000 {
                print("Congratulations, you've exceeded 1000 steps.")
                token = nil
            }
        }
    case .error(let error):
        print("An error occurred: \(error)")
    case .deleted:
        print("The object was deleted.")
    }
}
```

#### Interface-driven writes

在添加数据之后需要同步刷新UI的操作，就不需要再出发一个异步的通知去刷新TableView了，以免引起crash。

在提交写操作的时候只需要调用`Realm.commitWrite(withoutNotifying:)`

``` swift
// Add fine-grained notification block
token = collection.observe { changes in
    switch changes {
    case .initial:
        tableView.reloadData()
    case .update(_, let deletions, let insertions, let modifications):
        // Query results have changed, so apply them to the UITableView
        tableView.beginUpdates()
        tableView.insertRows(at: insertions.map({ IndexPath(row: $0, section: 0) }),
                             with: .automatic)
        tableView.deleteRows(at: deletions.map({ IndexPath(row: $0, section: 0)}),
                             with: .automatic)
        tableView.reloadRows(at: modifications.map({ IndexPath(row: $0, section: 0) }),
                             with: .automatic)
        tableView.endUpdates()
    case .error(let error):
        // handle error
        ()
    }
}

func insertItem() throws {
     // Perform an interface-driven write on the main thread:
     collection.realm!.beginWrite()
     collection.insert(Item(), at: 0)
     // And mirror it instantly in the UI
     tableView.insertRows(at: [IndexPath(row: 0, section: 0)], with: .automatic)
     // Making sure the change notification doesn't apply the change a second time
     try collection.realm!.commitWrite(withoutNotifying: [token])
}
```



