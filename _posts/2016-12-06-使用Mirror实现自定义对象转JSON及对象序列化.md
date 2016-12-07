---
layout: post
title: 使用Mirror实现自定义对象转JSON及对象序列化
date: 2016-12-06
categories: blog
tags: [总结,知识管理]
description: 使用Mirror实现自定义对象转JSON及对象序列化
---


### 需求
实现一个基类Model，继承它的类不需要再写代码即可实现对象转json即对象序列化

### 实现效果如下
基类为JSONModel，为了测试其健壮性，下面的例子写了一些嵌套关系

创建一些类，继承JSONModel：

     //用户类
     class User : JSONModel {
         var name:String = ""  //姓名
         var nickname:String?  //昵称
         var age:Int = 0   //年龄
         var emails:[String]?  //邮件地址
     }

     class Student: User {
         var accountID : Int = 0 //学号
     }

     class SchoolStudent: Student {
         var schoolName : String? //学校名
         var schoolmates : [Student]? //校友
         var principal : User? //校长
     }
     
初始化：
    
     //创建一个schoolstudent实例对象
        let schoolstudent = SchoolStudent()
        schoolstudent.schoolName = "清华大学"
        schoolstudent.accountID = 1024
        schoolstudent.name = "martin"
        schoolstudent.age = 20
        
        let principal = User()
        principal.name = "校长"
        principal.age = 60
        principal.emails = ["zhang@hangge.com","xiao@hangge.com"]
        schoolstudent.principal = principal
        
        let student1 = Student()
        student1.accountID = 2009
        student1.name = "martin"
        student1.age = 25
        student1.emails = ["martin1@hangge.com","martin2@hangge.com"]

        
        let student2 = Student()
        student2.accountID = 2008
        student2.name = "james"
        student2.age = 26
        student2.emails = ["james1@hangge.com","james2@hangge.com"]
        //添加动画
        schoolstudent.schoolmates = [student1, student2] 
        
测试打印JSON：这里实现了对象可打印，打印出来的即为对象转化后的JSON字符串，也可以调用 toJSONString() 方法

     //输出json字符串
        print("school student = \(schoolstudent)")
输出结果：

     school student = {
       "name" : "martin",
       "age" : 20,
       "accountID" : 1024,
       "schoolName" : "清华大学",
       "schoolmates" : [
         {
           "name" : "martin",
           "age" : 25,
           "accountID" : 2009,
           "emails" : [
             "martin1@hangge.com",
             "martin2@hangge.com"
           ]
         },
         {
           "name" : "james",
           "age" : 26,
           "accountID" : 2008,
           "emails" : [
             "james1@hangge.com",
             "james2@hangge.com"
           ]
         }
       ],
       "principal" : {
         "name" : "校长",
         "age" : 60,
         "emails" : [
           "zhang@hangge.com",
           "xiao@hangge.com"
         ]
       }
     }
     
测试对象序列化：     
        
        let a = NSKeyedArchiver.archivedData(withRootObject: schoolstudent)
        let b = NSKeyedUnarchiver.unarchiveObject(with: a)
        print("unarchiveObject = \(b)")
输出结果：
     
     unarchiveObject = Optional({
       "name" : "martin",
       "age" : 20,
       "accountID" : 1024,
       "schoolName" : "清华大学",
       "schoolmates" : [
         {
           "name" : "martin",
           "age" : 25,
           "accountID" : 2009,
           "emails" : [
             "martin1@hangge.com",
             "martin2@hangge.com"
           ]
          },
         {
           "name" : "james",
           "age" : 26,
           "accountID" : 2008,
           "emails" : [
             "james1@hangge.com",
             "james2@hangge.com"
           ]
         }
       ],
       "principal" : {
         "name" : "校长",
         "age" : 60,
         "emails" : [
           "zhang@hangge.com",
           "xiao@hangge.com"
         ]
       }
     })        
        
## 完整项目代码:[JSONModelWithMirror](https://github.com/martinLilili/JSONModelWithMirror/tree/master)    

下面记录整个实现过程

主要参考资料：

[Swift使用反射将自定义对象数据序列化成JSON数据](http://www.111cn.net/sj/iOS/100918.htm)    

[REFLECTION 和 MIRROR](http://swifter.tips/reflect/)

[打造强大的BaseModel（4）：使用Swift反射](http://ios.jobbole.com/84760/)

### 首先我们实现将自定义的类转化为JSON字符串

#### 实现原理

将自定义对象转成JSON数据的实现原理

1. 首先我们使用反射（Reflection）对自定义类型的数据对象中所有的属性进行递归遍历，生成字典类型的数据并返回。
2. 接着使用NSJSONSerialization就可以把这个字典类型的数据转换成jSON数据了。

##### 定义JSON协议

     //自定义一个JSON协议
     protocol JSON {
    
         /// 如果是对象实现了JSON协议，这个方法返回对象属性及其value的dic，注意如果某个value不是基础数据类型，会一直向下解析直到基础数据类型为止，另外可选类型和数组需要单独处理
         /// 如果是基础数据类型实现了JSON协议，只返回他们自己
         /// - Returns: 对于对象返回dic，对于基础数据类型，返回他们自己
         func toJSONModel() -> AnyObject?
    
         /// 生成JSON字符串
         ///
         /// - Returns: 字符串
         func toJSONString() -> String
     }
     
##### 实现JSON协议

     //扩展协议方法
     extension JSON {

         /// 使用mirror遍历所有的属性，并保存与dic中，如果属性的value不是基础类型，则一直向下解析直到基础类型
         ///
         /// - Parameter mir: mirror
         /// - Returns: dic
         func getResultFromMirror(mir : Mirror) -> [String:AnyObject] {
             var result: [String:AnyObject] = [:]
             if let superMirror = mir.superclassMirror { //便利父类所有属性
                 result = getResultFromMirror(mir: superMirror)
             }
             if (mir.children.count) > 0  {
                 for case let (label?, value) in (mir.children) {
                     //属性：label   值：value
                     if let jsonValue = value as? JSON { //如果value实现了JSON，继续向下解析
                         result[label] = jsonValue.toJSONModel()
                     }
                 }
             }
             return result
         }
    
         /// 将数据转成可用的JSON模型
         func toJSONModel() -> AnyObject? {
             let mirror = Mirror(reflecting: self)
             if mirror.children.count > 0  {
                 let result = getResultFromMirror(mir: mirror)
                 return result as AnyObject?  //有属性的对象，返回result是一个dic
             }
             return self as AnyObject?  //基础数据类型，返回自己
         }
    
         //将数据转成JSON字符串
         func toJSONString() -> String {
        
             let jsonModel = self.toJSONModel()
             //利用OC的json库转换成OC的Data，
             let data : Data! = try? JSONSerialization.data(withJSONObject: jsonModel ?? [:] , options: .prettyPrinted)
             //data转换成String打印输出
             if let str = String(data: data, encoding: String.Encoding.utf8) {
                 return str
             } else {
                 return ""
             }
         }
     }     
     
##### 其他类型实现JSON协议，这里可选类型和数组需要单独处理       

     //扩展可选类型，使其遵循JSON协议
     extension Optional: JSON {
         //可选类型重写toJSONModel()方法
         func toJSONModel() -> AnyObject? {
             if let x = self {
                 if let value = x as? JSON {
                     return value.toJSONModel()
                 }
             }
             return nil
         }
     }

     //扩展Swift的基本数据类型，使其遵循JSON协议
     extension String: JSON { }
     extension Int: JSON { }
     extension Bool: JSON { }
     extension Dictionary: JSON { }
     extension Array: JSON {
         func toJSONModel() -> AnyObject? {
             let mirror = Mirror(reflecting: self)
             if mirror.children.count > 0  {
                 var arr:[Any] = []
                 for childer in Mirror(reflecting: self).children {
                     if let jsonValue = childer.value as? JSON {
                         if let jsonModel = jsonValue.toJSONModel() {
                             arr.append(jsonModel)
                         }
                     }
                 }
                 return arr as AnyObject?
             }
             return self as AnyObject?
         }
     }
     
##### 实现基类 JSONModel，实现JSON, 实现CustomStringConvertible使对象可打印

     //MARK: - JSONModel
     class JSONModel: JSON, CustomStringConvertible {
        
         public var description: String {
             return toJSONString()
         }
   
     }
     
##### 测试，实现几个类：

     //电话结构体
     struct Telephone {
         var title:String  //电话标题
         var number:String  //电话号码
     }

     extension Telephone: JSON { }
      //用户类
     class User : JSONModel {
         var name:String = ""  //姓名
         var nickname:String?  //昵称
         var age:Int = 0   //年龄
         var emails:[String]?  //邮件地址
         var tels:[Telephone]? //电话
     }

     class Student: User {
         var accountID : Int = 0
     }

     class SchoolStudent: JSONModel {
         var schoolName : String?
         var schoolmates : [Student]?
         var principal : User?
     }     
     
##### 测试代码：

        //创建一个schoolstudent实例对象
        let schoolstudent = SchoolStudent()
        schoolstudent.schoolName = "清华大学"
        
        let principal = User()
        principal.name = "校长"
        principal.age = 60
        principal.emails = ["zhang@hangge.com","xiao@hangge.com"]
        schoolstudent.principal = principal
        
        let student1 = Student()
        student1.accountID = 2009
        student1.name = "martin"
        student1.age = 25
        student1.emails = ["martin1@hangge.com","martin2@hangge.com"]
        //添加手机
        let tel1 = Telephone(title: "手机", number: "123456")
        let tel2 = Telephone(title: "公司座机", number: "001-0358")
        student1.tels = [tel1, tel2]
        
        let student2 = Student()
        student2.accountID = 2008
        student2.name = "james"
        student2.age = 26
        student2.emails = ["james1@hangge.com","james2@hangge.com"]
        //添加手机
        let tel3 = Telephone(title: "手机", number: "123456")
        let tel4 = Telephone(title: "公司座机", number: "001-0358")
        student2.tels = [tel3, tel4]
        
        schoolstudent.schoolmates = [student1, student2]
        
        //输出json字符串
        print("school student = \(schoolstudent)")   
        
##### 输出结果：

     school student = {
       "principal" : {
         "name" : "校长",
         "age" : 60,
         "emails" : [
           "zhang@hangge.com",
           "xiao@hangge.com"
         ]
       },
       "schoolName" : "清华大学",
       "schoolmates" : [
         {
           "name" : "martin",
           "age" : 25,
           "accountID" : 2009,
           "tels" : [
             {
               "number" : "123456",
               "title" : "手机"
             },
             {
               "number" : "001-0358",
               "title" : "公司座机"
             }
           ],
           "emails" : [
             "martin1@hangge.com",
             "martin2@hangge.com"
           ]
         },
         {
           "name" : "james",
           "age" : 26,
           "accountID" : 2008,
           "tels" : [
             {
               "number" : "123456",
               "title" : "手机"
             },
             {
               "number" : "001-0358",
               "title" : "公司座机"
             }
           ],
           "emails" : [
             "james1@hangge.com",
             "james2@hangge.com"
           ]
         }
       ]
     }
        
### 下面实现NSCoding协议，实现对象的序列化

为了实现NSCoding，我们需要使用KVC，所以不支持纯swift的属性，像结构体，或者Int?等基础数据类型的可选类型都是不行的

#####  修改JSONModel，继承NSObject，实现NSCoding协议

     class JSONModel: NSObject, JSON, NSCoding {
     		...
     }
     
##### 添加方法，将对象转化为dic
这里需要注意的是codablePropertieValues()这个方法返回的dic只有一层，即其中的value值可能是另一个自定义对象，和上面JSON协议的方法toJSONModel()并不相同

     func getPropertieValuesWithMirror(mir : Mirror) -> [String:AnyObject] {
        var result: [String:AnyObject] = [:]
        if let superMirror = mir.superclassMirror {
            result = getPropertieValuesWithMirror(mir: superMirror)
        }
        for case let (label?, value) in (mir.children) {
            result[label] = unwrap(any: value) as AnyObject?
        }
        return result
    }
    
    
    /// 得到所有的属性及其对应的value，注意value不一定全是基础数据类型，可能是其他自定义对象。同时，dic中存储的都是有值的属性，那些没有赋值的属性不会出现在dic中
    ///
    /// - Returns: dic
    func codablePropertieValues() -> [String:AnyObject] {
        var codableProperties = [String:AnyObject]()
        let mirror = Mirror(reflecting: self)
        codableProperties = getPropertieValuesWithMirror(mir: mirror)
        return codableProperties
    }  
    
    /// 将一个any类型的对象转化为可选类型
    ///
    /// - Parameter any: any 类型对象
    /// - Returns: 可选值   
    func unwrap(any: Any) -> Any? {
        let mirror = Mirror(reflecting: any)
        if mirror.displayStyle != .optional {
            return any
        }
        if mirror.children.count == 0 { return nil } // Optional.None
        for case let (_?, value) in (mirror.children) {
            return value
        }
        return nil
    }
    
这里需要讲解下unwrap(any: Any) -> Any?这个方法的作用

这个方法是把一个any类型的对象转化为可选类型，为什么需要这个方法？我们看下面的代码：

     var any: Any   //声明一个Any类型数据
     //any = nil  报错：Nil cannot be assigned to type 'Any'
     
     var op : String? //声明可选类型
     op = nil  //赋值为nil
        
     any = op   //赋值给any，不报错
     
     print(any)
     print(any == nil)
输出结果很奇妙：
     
     nil
     false
对于用Mirror得到的value，就会产生这种问题，例如上面User对象中的

var nickname:String?  //昵称，

在初始化时没有赋值，在使用Mirror遍历到这个属性时返回的就是这么一种Any类型的数据，我们没办法直接通过value == nil来判断它是不是可选值，直接使用也是不被KVC接受的数据类型，所以需要将其转换为确定的可选类型，通过mirror.displayStyle来判断其确定的类型来做相应的处理。

这里如果不使用unwrap转换，而直接使用value的话，在调用unarchiveObject时报错：-[NSNull length]: unrecognized selector sent to instance 0x10c702fb0，可见直接使用value是有问题的

##### 实现 public func encode(with aCoder: NSCoder) 方法

     public func encode(with aCoder: NSCoder) {
        let dic = codablePropertieValues()  //得到所有有值的属性及其value
        for (key, value) in dic {  //对于不同的类型需要调用不同的encode方法
            switch value {
            case let property as AnyObject:
                aCoder.encode(property, forKey: key)
            case let property as Int:
                aCoder.encodeCInt(Int32(property), forKey: key)
            case let property as Bool:
                aCoder.encode(property, forKey: key)
            default:
                print("Nil value for \(key)")
            }
        }
    }
    
##### 实现 public required init?(coder aDecoder: NSCoder) 方法
     
先实现方法获取所有的属性，包括父类的属性，并存储与数组中    
 
     func getPropertiesWithMirror(mir : Mirror) -> [String] {
        var result: [String] = []
        if let superMirror = mir.superclassMirror {
            result = getPropertiesWithMirror(mir: superMirror)
        }
        for case let (label?, _) in (mir.children) {
            result.append(label)
        }
        return result
    }
    
    //便利所有的属性列表，将所有的属性存储到数组中
    func codableProperties() -> [String] {
        var codableProperties = [String]()
        let mirror = Mirror(reflecting: self)
        codableProperties = getPropertiesWithMirror(mir: mirror)
        return codableProperties
    }    

再实现 public required init?(coder aDecoder: NSCoder) 方法
 
    public required init?(coder aDecoder: NSCoder) {
        super.init()     //先初始化
        let arr = codableProperties()  //得到所有的属性列表
        for key in arr {
            let object = aDecoder.decodeObject(forKey: key)
            self.setValue(object, forKey: key)
        }
    }
    
##### 测试，基于上面的示例测试 
代码：
    
        let a = NSKeyedArchiver.archivedData(withRootObject: schoolstudent)
        let b = NSKeyedUnarchiver.unarchiveObject(with: a)
        print("unarchiveObject = \(b)")  

在调用archivedData报错： -[_SwiftValue encodeWithCoder:]: unrecognized selector sent to instance 

原因是对象中有纯swift的结构，没办法只能去掉

注释掉结构：

     //用户类
     class User : JSONModel {
         var name:String = ""  //姓名
         var nickname:String?  //昵称
         var age:Int = 0   //年龄
         var emails:[String]?  //邮件地址
     //    var tels:[Telephone]? //电话
     }    

输出：
 
     unarchiveObject = Optional({
       "principal" : {
         "name" : "校长",
         "age" : 60,
         "emails" : [
           "zhang@hangge.com",
           "xiao@hangge.com"
         ]
       },
       "schoolName" : "清华大学",
       "schoolmates" : [
         {
           "name" : "martin",
           "age" : 25,
           "accountID" : 2009,
           "emails" : [
             "martin1@hangge.com",
             "martin2@hangge.com"
            ]
         },
         {
           "name" : "james",
           "age" : 26,
           "accountID" : 2008,
           "emails" : [
             "james1@hangge.com",
             "james2@hangge.com"
           ]
         }
       ]
     })   
     
     
## 以上就实现了对象的序列化

#### 下面我们做一些其他测试       

##### 将user类的age改成可选类型
     
     var age:Int? = 0   //年龄
    
在调用unarchiveObject时报错：

'[<JSONModel.Student 0x6000000d3080> setValue:forUndefinedKey:]: this class is not key value coding-compliant for the key age.' 

可见对象中不能有基础数据类型的可选值，原因应该是oc无法识别这种类型，但是String？却是可以的。

我们知道，swift中很多类型都被桥接到oc，如String->NSString等，Int也被桥接为NSNumber，但是有一定的条件，具体可看我另一片文章[swift 类型转换笔记](http://moonlspace.com/2016/11/swift-%E7%B1%BB%E5%9E%8B%E8%BD%AC%E6%8D%A2%E7%AC%94%E8%AE%B0/)，另外还有歪果仁写的：

KVO cannot function with pure Swift optionals because pure Swift optionals are not Objective-C objects. Swift forbids the use of dynamic or @objc with generic classes and structures because there is no valid Objective-C equivalent, and so the runtime is not setup to support KVO on instances of those kinds of objects. As for why it works with String?, that type is toll-free bridged to NSString, so it is semantically equivalent to NSString *, an Objective-C type that the runtime knows full-well how to deal with. But compare that to Int? whose semantic equivalent would be Optional<Int>, not UnsafePointer<Int> or NSNumber * like you might expect. For now, you'll need to convince the typechecker that it is safe to represent in Objective-C by using NSNumber!.

不管理解不理解，这里我们规定对象中不能出现基础类型的可选值即可。

### 总结：使用此JSONModel，如果只是实现对象转JSON，对于属性没有限制，如果想要实现对象序列化，需要确保对象中没有纯swift的类型数据，如Int？结构等等。

## 其他方案

根据喵神的说法，Mirror并不是为开发者准备的借口，功能也非常弱小，未来不确定性很大，可能突然被禁用，也可能加大支持，总之可能有很多变化，不建议使用。

那我们还有什么其他方案？

下面推荐一个库：[ObjectMapper](https://github.com/Hearst-DD/ObjectMapper)，可以实现对象转JSON
