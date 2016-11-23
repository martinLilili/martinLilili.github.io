---
layout: post
title: swift开发tips
date: 2016-11-23
categories: blog
tags: [总结,知识管理]
description: swift开发tips
---


##swift开发tips

###打log

在 Xcode 的 Build Setting 中，在 Other Swift flags 的 Debug 栏中加入 -D DEBUG 即可加入一个编译标识。

![Other Swift flags](http://oh36yj5vw.bkt.clouddn.com/debug-flag.png)

添加方法：

        /// 打log方法，release不会输出
	     ///
        /// - Parameters:
        ///   - message: log 文本
        ///   - fileName: 文件名，有默认值
        ///   - methodName: 方法名，有默认值
        ///   - lineNumber: 行数，有默认值
        public func DebugLog<T>(item: T, fileName: String = #file, methodName: String =  #function, lineNumber: Int = #line)
       {
            #if DEBUG
            let str : String = ((fileName as NSString).pathComponents.last! as NSString).replacingOccurrences(of: "swift", with: "")
            print("\(str)\(methodName)[\(lineNumber)]:\(item)")
            #endif
       }
       
测试，在didFinishLaunchingWithOptions打印各种情况：

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        // Override point for customization after application launch.

        DebugLog(item: "输出字符串")
        DebugLog(item: ["arr1", "arr2"])
        DebugLog(item: ["key" : "value"])
        DebugLog(item: testLog())
        return true
    }
    
    func testLog() -> String {
        DebugLog(message: "test")
        return "testResult"
    }       
       
输出结果：

    AppDelegate.application(_:didFinishLaunchingWithOptions:)[20]:输出字符串
    AppDelegate.application(_:didFinishLaunchingWithOptions:)[21]:["arr1", "arr2"]
    AppDelegate.application(_:didFinishLaunchingWithOptions:)[22]:["key": "value"]
    AppDelegate.testLog()[28]:test
    AppDelegate.application(_:didFinishLaunchingWithOptions:)[23]:testResult   
    
注意这里如果你传入的 items 是一个表达式而不是直接的变量的话，这个表达式还是会被先执行求值的。如果这对性能也产生了可测的影响的话，我们最好用 @autoclosure 修饰参数来重新包装 print。这可以将求值运行推迟到方法内部，这样在 Release 时这个求值也会被一并去掉： 
 
    func dPrint(@autoclosure item: () -> Any) {
        #if DEBUG
        print(item())
        #endif
    }

    dPrint(resultFromHeavyWork())
    // Release 版本中 resultFromHeavyWork() 不会被执行     
    
    
    
###添加注释快捷键

![zhushi](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-11-23%20%E4%B8%8B%E5%8D%885.16.37.png)
       