---
layout: post
title: Blockly在iOS上的实践
date: 2017-11-24
categories: blog
tags: [总结,知识管理]
description: Blockly在iOS上的实践
---
		

# Blockly在iOS上的实践


好久没写东西了，惭愧

身在一个机器人公司，才知道Blockly，惭愧

## 什么是Blockly？

Blockly是一个用于Web、Android、IOS的可视化代码编辑器库。Blockly使用了相互关联的积木来表示表达代码中变量、逻辑表达式、循环等。它让用户能够了解编程，而不用面对命令行上让人恐惧和枯燥的代码和语法。

官方地址：

[overview](https://developers.google.com/blockly/guides/overview)            
[iosgithup](https://github.com/google/blockly-ios)

大概的形式是这样的：
![pic1](http://oh36yj5vw.bkt.clouddn.com/Simulator%20Screen%20Shot%20-%20iPhone%208%20-%202017-11-24%20at%2011.22.22.png)



个人理解：Blockly是一套编程语言，我们可以通过拖拽的方式组织逻辑，每一个可拖拽的模块都是一段代码块，Blockly将我们组织好的逻辑转化为js代码，iOS通过JavaScriptCore调用这些js代码实现我们想要的功能。


下面参考[官方Demo BlocklyCodeLab](https://codelabs.developers.google.com/codelabs/blockly-ios/#0)实现一个简单如上面图所示的Demo

设想这样一种需求，我们需要给机器人编辑一套行为，行为是由机器人的表情，具体动作，及语音播报组合而成，如让机器人介绍自己的行为，是由开心表情，和抬头动作及一段语音"你好，我是瓦利"组成。下面我们就实现如上功能



## 初始化APP

创建BlocklyDemo工程

通过cocoapods引入Blockly

    cd "path-to-your-project"

    pod init
    
编辑podfile

	source 'https://github.com/CocoaPods/Specs.git'
	
	target 'YourProjectAppTarget' do
	  use_frameworks!
	
	  pod 'Blockly'
	end    
		
执行：

	pod install
	pod update
	
	
#### 初始化Blockly workbench（工作台）	

引用官方文档：
WorkbenchViewController is a view controller that contains a Blockly workspace, toolbox, trash can, and undo/redo controls. A workspace is the UI component that contains the "active" blocks to be dragged around, while the toolbox contains the list of blocks that can be added to the workspace.

![pic2](http://oh36yj5vw.bkt.clouddn.com/af1300cf1866d3e1.png)

代码如下：

    //style 设置工具条是在左边还是下边
    let workbenchViewController = WorkbenchViewController(style: .alternate)     
    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
       
        // Load its block factory with default blocks。加载默认内置的代码块，后面我们需要用它来加载自定义的代码块
        let blockFactory = workbenchViewController.blockFactory
        blockFactory.load(fromDefaultFiles: .allDefault)
        
        
        // Add editor to this view controller
        addChildViewController(workbenchViewController)
        view.addSubview(workbenchViewController.view)
        workbenchViewController.view.frame = view.bounds
        workbenchViewController.view.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        workbenchViewController.didMove(toParentViewController: self)
    }		


这时我们会得到一个空的工作台


## 设置toolbox工具条


我们创建一个toolbox.xml文件，将它添加到项目中

文件中代码如下：

	<xml>
	  <category name="Loops" colour="120">
	    <block type="controls_repeat_ext">
	      <value name="TIMES">
	        <shadow type="math_number">
	          <field name="NUM">5</field>
	        </shadow>
	      </value>
	    </block>
	  </category>
	</xml>
	
这个xml文件定义了一个工作条，它上面有一个循环的代码块叫Loops

我们将这个xml文件加载进去：

    override func viewDidLoad() {
        super.viewDidLoad()
        
        ...
        
        // Load toolbox
        do {
            let toolboxPath = "toolbox.xml"
            if let bundlePath = Bundle.main.path(forResource: toolboxPath, ofType: nil) {
                let xmlString = try String(contentsOfFile: bundlePath, encoding: String.Encoding.utf8)
                let toolbox = try Toolbox.makeToolbox(xmlString: xmlString, factory: blockFactory)
                try workbenchViewController.loadToolbox(toolbox)
            } else {
                print("Could not load toolbox XML from '\(toolboxPath)'")
            }
        } catch let error {
            print("An error occurred loading the toolbox: \(error)")
        }
        
        // Add editor to this view controller
        ...
    }	


运行APP 得到如下：

![pic3](http://oh36yj5vw.bkt.clouddn.com/3ef5d1e445445b85.png)

工具条中多了一个叫Loops的选项

## 自定义代码块

系统默认的代码块显然无法满足我们的需求，我们需要自定义

我们创建一个custom_blocks.json文件并加入到项目中

首先我们需要一个形如![pic4](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%882.48.41.png)的代码块，它允许用户输入一个标题，并将其他表情，行为等组合起来。custom_blocks.json中代码如
下：

	[
	 {
	 "type": "title_input_block",
	 "message0": "标题: %1",
	 "args0": [
	           {
	           "type": "field_input",
	           "name": "Input",
	           "text": ""
	           }
	           ],
	 "message1": "添加 %1",
	 "args1": [
	           {
	           "type": "input_statement",
	           "name": "DO"
	           }
	           ],
	 "previousStatement": null,
	 "nextStatement": null,
	 "colour": 20
	 }
	]
	
type是这块代码的唯一标识，后面要用到

message即显示的文字	

args0中的type是具体控件的类型，如这里是系统默认提供的输入框

args0中的name是这个控件的唯一标识，后面取它的值时要用到

args1与args0类似，注意input_statement这个类型表示可以拼接其他类型的控件

previousStatement，nextStatement，colour等在[官方文档Define Blocks](https://developers.google.com/blockly/guides/create-custom-blocks/define-blocks)中都有详细的介绍。这里表示可以向前关联及向后关联，及颜色

同理我们还需要![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%882.48.48.png)![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%882.48.55.png)![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%882.49.01.png)这3中代码块，实现如下：

	[
	 {
	 "type": "title_input_block",
	 "message0": "标题: %1",
	 "args0": [
	           {
	           "type": "field_input",
	           "name": "Input",
	           "text": ""
	           }
	           ],
	 "message1": "添加 %1",
	 "args1": [
	           {
	           "type": "input_statement",
	           "name": "DO"
	           }
	           ],
	 "previousStatement": null,
	 "nextStatement": null,
	 "colour": 20
	 },
	  {
	    "type": "expression",
	    "message0": "表情 %1",
	    "args0": [{
	      "type": "field_dropdown",
	      "name": "VALUE",
	      "options": [
	        ["开心", "happy"],
	        ["难过", "sad"]
	      ]
	    }],
	    "previousStatement": null,
	    "nextStatement": null,
	    "colour": 60
	  },
	     {
	     "type": "action",
	     "message0": "动作 %1",
	     "args0": [{
	               "type": "field_dropdown",
	               "name": "VALUE",
	               "options": [
	                           ["抬头", "head"],
	                           ["握手", "hand"]
	                           ]
	               }],
	     "previousStatement": null,
	     "nextStatement": null,
	     "colour": 100
	     },
	 {
	 "type": "text_input_block",
	 "message0": "文本内容: %1",
	 "args0": [
	           {
	           "type": "field_input",
	           "name": "Input",
	           "text": ""
	           }
	 ],
	 "message1": "延时 %1",
	 "args1": [
	           {
	           "type": "field_number",
	           "name": "number",
	           "min": 0
	           }
	           ],
	 "previousStatement": null,
	 "nextStatement": null,
	 "colour": 140
	 }
	]
	
这部分可能初学者最难的部分，需要好好查看文档及官方示例，但也不是很难	

##### 下一步将json加载到项目中

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view, typically from a nib.
       
        // Load its block factory with default blocks
        ...
        
        do {
            try blockFactory.load(fromJSONPaths: ["custom_blocks.json"])
        } catch let error {
            print("An error occurred loading the sound blocks: \(error)")
        }
        
        // Load toolbox
        ...
        
        // Add editor to this view controller
        ...
        
    }
    
##### 下面将上述代码块加入到工具条中，修改toolbox.xml文件

	<xml>
	    <category name="标题" colour="20">
	        <block type="title_input_block"></block>
	    </category>
	  <category name="表情" colour="60">
	    <block type="expression"></block>
	  </category>
	  <category name="动作" colour="100">
	      <block type="action"></block>
	  </category>
	  <category name="文本内容" colour="140">
	      <block type="text_input_block"></block>
	  </category>
	</xml>    

##### 运行APP可以看到

![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%883.13.41.png)![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%883.13.48.png)![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%883.13.54.png)![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%883.14.00.png)


## 保存/加载 workspace

当我们使用如上的代码块编写一段逻辑后我们需要将其保存下来，以便转化为js代码得以运行

添加一个保存按钮，实现如下：

    @IBAction func saveBtnClicked(_ sender: UIButton) {
        print("saveBtnClicked")
        // Save the workspace to disk
        if let workspace = workbenchViewController.workspace {
            do {
                let xml = try workspace.toXML()
                FileHelper.saveContents(xml, to: "workspace.xml") //FileHelper是一个文件工具类
                
            } catch let error {
                print("Couldn't save workspace to disk: \(error)")
            }
        }
    }


当我们已经编辑过一段代码块后，下次回来也可以重新加载进来

	do {
	    // Create fresh workspace
	    let workspace = Workspace()
	
	    // Load blocks into this workspace from a saved file (if it exists).
	    if let xml = FileHelper.loadContents(of: "workspace.xml") {
	      try workspace.loadBlocks(
	        fromXMLString: xml, factory: workbenchViewController.blockFactory)
	    }
	
	    // Load the workspace into the workbench
	    try workbenchViewController.loadWorkspace(workspace)
	  } catch let error {
	    print("Couldn't load workspace from disk: \(error)")
	  }


## 实现JavaScript代码

从上面可知我们存贮的是xml格式的代码，想让它运行在iOS设备上，需要将其转化为JavaScript代码

#### 为custom_blocks.json添加js实现

创建custom_generators.js文件并添加到项目中

代码如下：

	'use strict';
	
	// Generators for blocks defined in `custom_blocks.json`.
	Blockly.JavaScript['expression'] = function(block) {
    	//通过'VALUE'字段获得表情的标识
	  var value = '\'' + block.getFieldValue('VALUE') + '\''; 
	  //将	expression实现为ActionMaker.setExpression('value'); value及选中的表情标识
	  return 'ActionMaker.setExpression(' + value + ');\n';
	};
	
	Blockly.JavaScript['action'] = function(block) {
	//通过'VALUE'字段获得动作的标识
	    var value = '\'' + block.getFieldValue('VALUE') + '\'';
	    //将	action实现为ActionMaker.setAction('value');value及选中的动作标识
	    return 'ActionMaker.setAction(' + value + ');\n';
	};
	
	Blockly.JavaScript['text_input_block'] = function(block) {
	//通过'Input'字段获得输入的文本
	    var text = '\'' + block.getFieldValue('Input') + '\'';
	    //通过'number'字段获得选择的数字
	    var delay = '\'' + block.getFieldValue('number') + '\'';
	    //将	text_input_block实现为'ActionMaker.setText('text','delay');
	    return 'ActionMaker.setText(' + text + ',' + delay + ');\n';
	};
	
	
	Blockly.JavaScript['title_input_block'] = function(block) {
	//通过'Input'字段获得输入的文本
	    var title = '\'' + block.getFieldValue('Input') + '\'';
	    
	    //通过'DO'字段获其他代码块转化的js代码
	    var func = Blockly.JavaScript.statementToCode(block,"DO");
	    
	    //将	title_input_block实现为：其他代码块的代码 + ActionMaker.setText('title');
	    return func + 'ActionMaker.setTitle(' + title + ');\n';
	};
	
从上述代码我们可以看到通过json中的name字段取值的方法，及随后生成的代码基本规则。这里如何取得我们想要的值也是需要好好参考文档及Demo的地方

#### 创建代码生成服务

创建CodeManager.swift文件，并添加代码：

	
	import Blockly
	import Foundation
	
	/**
	 Manages JS code in the app. It generates JS code from workspace XML and saves it in-memory for
	 future use.
	 */
	class CodeManager {
	    /// Stores JS code for a unique key (ie. a button ID).
	    private var savedCode = String()
	    
	    /// Service used for converting workspace XML into JS code.
	    private var codeGeneratorService: CodeGeneratorService = {
	        let service = CodeGeneratorService(
	            jsCoreDependencies: [
	                // The JS file containing the Blockly engine
	                "blockly_web/blockly_compressed.js",
	                // The JS file containing a list of internationalized messages
	                "blockly_web/msg/js/en.js"
	            ])
	        
	        let requestBuilder = CodeGeneratorServiceRequestBuilder(
	            // This is the name of the JS object that will generate JavaScript code
	            jsGeneratorObject: "Blockly.JavaScript")
	        // Load the block definitions for all default blocks
	        requestBuilder.addJSONBlockDefinitionFiles(fromDefaultFiles: .allDefault)
	        // 加载自定义的json
	        requestBuilder.addJSONBlockDefinitionFiles(["custom_blocks.json"])
	        requestBuilder.addJSBlockGeneratorFiles([
	            // Use JavaScript code generators for the default blocks
	            "blockly_web/javascript_compressed.js",
	            // 加载自定义的js实现
	            "custom_generators.js"])
	        
	        // Assign the request builder to the service and cache it so subsequent code generation
	        // runs are immediate.
	        service.setRequestBuilder(requestBuilder, shouldCache: true)
	        
	        return service
	    }()
	    
	    deinit {
	        codeGeneratorService.cancelAllRequests()
	    }
	    
	}

注意上面的blockly_web/blockly_compressed.js是系统默认自带的代码块的js实现，pod时并没有加到项目中，需要单独添加，在官方Demo中有blockly_web文件夹，将其添加到项目中，注意选择Create folder reference	.


#### 实现代码

在CodeManager.swift中添加：

    /// Stores JS code 
    private var savedCode = String()
    
    /**
     Generates code 
     */
    func generateCode(workspaceXML: String, savedCode : @escaping (String) -> Void) {
        let errorHandler = { (error: String) -> () in
            print("An error occurred generating code - \(error)\n" + "workspaceXML: \(workspaceXML)\n")
        }
        
        do {
            // Clear the code for this key as we generate the new code.
            self.savedCode = ""
            
            let _ = try codeGeneratorService.generateCode(
                forWorkspaceXML: workspaceXML,
                onCompletion: { requestUUID, code in
                    // Code generated successfully. Save it for future use.
                    self.savedCode = code
                    savedCode(code)
            },
                onError: { requestUUID, error in
                    errorHandler(error)
            })
        } catch let error {
            errorHandler(error.localizedDescription)
        }
    }
    
如上将xml转为js代码，并通过savedCode block返回转化js的代码

#### 使用CodeManager

在ViewController.swift中

	private var codeManager = CodeManager()
	
    func generateCode() {
        // If a saved workspace file exists for this button, generate the code for it.
        if let workspaceXML = FileHelper.loadContents(of: "workspace.xml") {
            codeManager.generateCode(workspaceXML: workspaceXML, savedCode: { code in
                print("Generate Code  :\n \(code)")
            })
        }
    }	
    
在saveBtn中调用

    @IBAction func saveBtnClicked(_ sender: UIButton) {
        print("saveBtnClicked")
        // Save the workspace to disk
        if let workspace = workbenchViewController.workspace {
            do {
                let xml = try workspace.toXML()
                FileHelper.saveContents(xml, to: "workspace.xml")
                
                //调用
                generateCode()
                
            } catch let error {
                print("Couldn't save workspace to disk: \(error)")
            }
        }
    } 
    
形如![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%883.56.14.png)打印结果如下：

	Generate Code   :
	   ActionMaker.setExpression('happy');
	  ActionMaker.setAction('head');
	  ActionMaker.setText('我是瓦利','1');
	ActionMaker.setTitle('介绍自己');
    
    
## 通过js调用iOS代码

从上面的代码我们知道，我们需要一个ActionMaker类并为其实现几个方法

创建ActionMaker.swift文件并添加代码如下：

	import Foundation
	import JavaScriptCore
	
	
	/**
	 Protocol declaring all methods and properties that should be exposed to a JS context.
	 */
	@objc protocol ActionMakerJSExports: JSExport {
	    static func setExpression(_ name: String)
	    static func setAction(_ name: String)
	    static func setText(_ text: String, _ delay : String)
	    static func setTitle(_ title: String)
	}
	
	/**
	 Class exposed to a JS context.
	 */
	@objcMembers class ActionMaker: NSObject, ActionMakerJSExports {
	    static func setText(_ text: String, _ delay: String) {
	        print("text = \(text), delay = \(delay)")
	    }
	    
	    static func setTitle(_ title: String) {
	        print("title = \(title)")
	    }
	 
	    static func setExpression(_ name: String) {
	        print("Expression = \(name)")
	    }
	    
	    static func setAction(_ name: String) {
	        print("Action = \(name)")
	    }
	}
	
如上我们声明了一系列方法，并做了简单打印实现，以确认其确实被调用

## 运行js代码

创建CodeRunner.swift文件，并添加代码：
	
	import JavaScriptCore
	
	
	/**
	 Runs JavaScript code.
	 */
	class CodeRunner {
	    /// Use a JSContext object, which contains a JavaScript virtual machine.
	    private var context: JSContext?
	    
	    /// Create a background thread, so the main thread isn't blocked when executing
	    /// JS code.
	    private let jsThread = DispatchQueue(label: "jsContext")
	    
	    init() {
	        // Initialize the JSContext object on the background thread since that's where
	        // code execution will occur.
	        jsThread.async {
	            self.context = JSContext()
	            self.context?.exceptionHandler = { context, exception in
	                let error = exception?.description ?? "unknown error"
	                print("JS Error: \(error)")
	            }
	            
	            // Register ActionMaker class with the JSContext object. This tells JSContext to
	            // route any JavaScript calls to `ActionMaker` back to iOS code.
	            self.context?.setObject(
	                ActionMaker.self, forKeyedSubscript: "ActionMaker" as NSString)
	        }
	    }
	    
	    /**
	     Runs Javascript code on a background thread.
	     
	     - parameter code: The Javascript code.
	     - parameter completion: Closure that is called on the main thread when
	     the code has finished executing.
	     */
	    func runJavascriptCode(_ code: String, completion: @escaping () -> ()) {
	        jsThread.async {
	            // Evaluate the JavaScript code asynchronously on the background thread.
	            _ = self.context?.evaluateScript(code)
	            
	            // When it finishes, call the completion closure on the main thread.
	            DispatchQueue.main.async {
	                completion()
	            }
	        }
	    }
	}	
	
在ViewController中的generateCode中调用

    func generateCode() {
        // If a saved workspace file exists for this button, generate the code for it.
        if let workspaceXML = FileHelper.loadContents(of: "workspace.xml") {
            codeManager.generateCode(workspaceXML: workspaceXML, savedCode: { code in
                print("Generate Code  :\n \(code)")
                
                let codeRunner = CodeRunner()
                codeRunner.runJavascriptCode(code, completion: {
                    print("code runned")
                })
            })
        }
    }	
    
    
运行APP实现![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-11-24%20%E4%B8%8B%E5%8D%883.56.14.png)
点击save，打印结果如下：

	Generate Code  :
	   ActionMaker.setExpression('happy');
	  ActionMaker.setAction('head');
	  ActionMaker.setText('我是瓦利','1');
	ActionMaker.setTitle('介绍自己');
	
	Expression = happy
	Action = head
	text = 我是瓦利, delay = 1
	title = 介绍自己
	code runned
    
可见ActionMaker中的方法都被调用了，后面就是根据具体需求去实现逻辑了。


## 总结    
本文只是从最简单的地方入手，实现了整体流程，对于初学者来说，如何自定义代码块，及编写代码块的js实现才是重点需要深入研究的地方。

[本文Demo代码地址](https://github.com/martinLilili/BlocklyDemo)

[官方overview](https://developers.google.com/blockly/guides/overview)            

[官方iosgithup](https://github.com/google/blockly-ios)

[官方Demo BlocklyCodeLab](https://codelabs.developers.google.com/codelabs/blockly-ios/#0)