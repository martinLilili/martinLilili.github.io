---
layout: post
title: A Peek at 3D Touch(2016wwdc笔记)
date: 2016-12-14
categories: blog
tags: [总结,知识管理]
description: A Peek at 3D Touch(2016wwdc笔记)
---
		

苹果在iOS9引入了3D Touch，使APP可以根据用户手指按压屏幕的力度来做出响应，实现新的用户体验。

iOS9中提供了Home Screen Quick Actions 和 Peek and Pop 两种应用方法，iOS10又加入了UIPreviewInteraction API，使开发者可以更多的应用此功能。

下面分别简单介绍上面3种功能：

本文例子取自session中的Demo: AppChat，可以在xcode的document中找到：

![1](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-01-05%20%E4%B8%8B%E5%8D%885.22.23.png)

## Home Screen Quick Actions
即主屏幕快捷键，手指按压APP图标，可以弹出快捷键使用户可以直接开启常用功能。

![2](http://oh36yj5vw.bkt.clouddn.com/3dtouchgif1.gif)

快捷键最多为4个。

快捷键分为两种：静态的（如上图 new chat 按钮）和动态的（上图中其他3个按钮）
下面分别介绍如何实现两种快捷键：

#### 静态快捷键
静态快捷键即是标题或icon不会变化的快捷键，只需要在info.plist声明即可，如图：
![3](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-01-05%20%E4%B8%8B%E5%8D%885.33.26.png)

1. UIApplicationShortcutItemIconType 表示快捷键的图标，这里只能使用系统提供的一系列图标，通过枚举 UIApplicationShortcutIconType 定义
2. UIApplicationShortcutItemTitle 表示快捷键的标题，这里是支持国际化的，即你可以在InfoPlist.strings中声明，如上图中的例子在InfoPlist.strings中声明

		"SHORTCUT_TITLE_NEWCHAT" = "New Chat";
		
3. UIApplicationShortcutItemType 表示快捷键的唯一标识，用于在接收到快捷键点击事件时区分，其定义方式类似于Bundle id。

#### 动态快捷键
动态快捷键即快捷键的图标标题等可能会有变化，如本文Demo中的后3个快捷键，本文Demo是一款聊天软件，后三个快捷键是最常聊天的3个人的快捷聊天入口，所以需要根据具体情况变化。动态快捷键的实现方式如下：

        var shortcutItems = [UIApplicationShortcutItem]() //创建UIApplicationShortcutItem数组
        
        let top3Friends = ChatItemManager.sharedInstance.topFriends.prefix(3) //查找3个最常联系人
        for friend in top3Friends {
            let type = ShortcutItemType.sendChatTo   //声明type
            let title = friend.name   //声明标题
            let subtitle = NSLocalizedString("Send a chat", comment: "Send a chat to a specific friend")   //声明子标题
            //声明icon，如果能够得到联系人的头像则设置为具体的联系人头像，如果不能取到，则默认设置为message图标
            var icon = UIApplicationShortcutIcon(type: .message) 
            if grantedAccessToContacts() {
                let predicate = CNContact.predicateForContacts(matchingName: friend.name)
                let contacts = try? CNContactStore().unifiedContacts(matching: predicate, keysToFetch: [])
                if let contact = contacts?.first {
                    icon = UIApplicationShortcutIcon(contact: contact)
                }
            }
            
            let userInfo = ShortcutItemUserInfo(friendIdentifier: friend.identifier)
            //创建UIApplicationShortcutItem
            let shortcutItem = UIApplicationShortcutItem(type: type.prefixedString, localizedTitle: title, localizedSubtitle: subtitle, icon: icon, userInfo:userInfo.dictionaryRepresentation)  
            shortcutItems.append(shortcutItem) //加入数组
        }
        
        application.shortcutItems = shortcutItems	
上面最重要的就是创建UIApplicationShortcutItem

		public init(type: String, localizedTitle: String, localizedSubtitle: String?, icon: UIApplicationShortcutIcon?, userInfo: [AnyHashable : Any]? = nil)

它要求传入title，subtitle，icon用于显示，还需要传入type，userInfo用于响应点击时间，进行我们想要的操作。

### 响应快捷键
当用户点击了快捷键后，我们需要监听并做出相应的处理。

当APP在后台运行时，通过系统回调处理

    func application(_ application: UIApplication, performActionFor shortcutItem: UIApplicationShortcutItem, completionHandler: @escaping (Bool) -> Void) {
        // Handle ShortcutItem
    }

当APP没在后台运行时，在didFinishLaunch中处理

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
        ...
       
        if let shortcutItem = launchOptions?[.shortcutItem] as? UIApplicationShortcutItem {
            
            // Handle ShortcutItem
        }
        
        ...
        
        return performAdditionalHandling
    }



## Peek and Pop
Peek and Pop是一个很有限制的功能，它只能结合viewcontroller实现对viewcontroller的预览，例如我们有个列表，点击一个cell进入一个viewcontroller，点击返回再回到列表，完成这一系列操作需要3步，但使用Peek and Pop只需要按压cell即可预览viewcontroller，此时用户可以通过控制手指的力度选择进入这个viewcontroller或者返回，或者通过快捷键进行一些常用操作，如图：
![4](http://oh36yj5vw.bkt.clouddn.com/3dtouchgif2.gif)

总结Peek and Pop的主要功能

1. 用户可以通过按压cell预览子viewcontroller
2. 用户可以放手，返回列表
3. 用户可以继续按压，进入viewcontroller
4. 用户可以向上滑动预览页面，使用快捷键进行操作

下面讲下如何实现

#### 注册
使列表页面实现

	extension ChatTableViewController: UIViewControllerPreviewingDelegate 
	
Peek and Pop的页面关系如下：
![5](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-01-05%20%E4%B8%8B%E5%8D%886.32.51.png)	
注册

    override func viewDidLoad() {
        super.viewDidLoad()

        registerForPreviewing(with: self, sourceView: tableView) //注册tableView为sourceView
    }

#### 响应按压

Peek and Pop分为两段式按压，即Peak（Preview）和Pop（Commit），分别对应两个回调方法

当用户轻轻按压cell时会先进入Priview回调，实现如下：

    func previewingContext(_ previewingContext: UIViewControllerPreviewing, viewControllerForLocation location: CGPoint) -> UIViewController? {
        // 得到点击cell的index
        guard let indexPath = tableView.indexPathForRow(at: location) else { return nil }
        
        // 初始化将要显示的viewcontroller
        let storyboard = UIStoryboard(name: "Main", bundle: nil)
        let viewController = storyboard.instantiateViewController(withIdentifier: ChatDetailViewController.identifier)
        guard let chatDetailViewController = viewController as? ChatDetailViewController else { return nil }
        chatDetailViewController.chatItem = chatItem(at: indexPath)
        chatDetailViewController.isReplyButtonHidden = true
        
        //设置点击时显示的高亮区域，当按压cell时cell会有一个高亮的效果
        let cellRect = tableView.rectForRow(at: indexPath)
        previewingContext.sourceRect = previewingContext.sourceView.convert(cellRect, from: tableView)

		//返回viewcontroller        
        return chatDetailViewController
    }


当用户继续加力按压时会进入Commit回调，实现如下：

    func previewingContext(_ previewingContext: UIViewControllerPreviewing, commit viewControllerToCommit: UIViewController) {
        ...
        //show方法即push，这里也可以使用present
        show(viewControllerToCommit, sender: self)
    }

#### 实现快捷键

当进入Preview状态时，向上滑动页面可以现实快捷键如：
![](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-01-05%20%E4%B8%8B%E5%8D%886.47.04.png)

它需要子viewcontroller重写previewActionItems方法

如：
	override func previewActionItems() -> [UIPreviewActionItem] {       let heart = UIPreviewAction(title:"❤", style: .default) { (action, viewController) in let heart = UIPreviewAction(title: "        // 处理点击事件       }       return [heart]    }

也可以为快捷键添加子健：

	let replyActions = [UIPreviewAction(title: "❤️", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "😄", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "👍", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "😯", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "😢", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "😈", style: .default, handler: replyActionHandler)]    let sendReply = UIPreviewActionGroup(title: "Send Reply...",                                     style: .default,                                     actions: replyActions)
                                     
下面看下Demo中完整的实现：

    override var previewActionItems: [UIPreviewActionItem] {
       
        let replyActionHandler = {[unowned self] (action: UIPreviewAction, viewController: UIViewController) -> Void in
            // 处理reply事件
        }
        let replyActions = [UIPreviewAction(title: "❤️", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "😄", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "👍", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "😯", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "😢", style: .default, handler: replyActionHandler),
                            UIPreviewAction(title: "😈", style: .default, handler: replyActionHandler)]
        let sendReply = UIPreviewActionGroup(title: NSLocalizedString("Send Reply…", comment: "Send reply action group title"), style: .default, actions: replyActions)
        
        let save = UIPreviewAction(title: savePreviewActionTitle(for: chatItem), style: savePreviewActionStyle(for: chatItem)) {[unowned self] (action, viewController) in
            // 处理save事件
        }
        
        let block = UIPreviewAction(title: NSLocalizedString("Block", comment: "Block the user action item"), style: .destructive) {[unowned self] (action, viewController) in
            // 处理block事件
        }
        
        return [sendReply, save, block]
    }
                                     
值得说明的是style参数，不同的style会有不同的效果:
![6](http://oh36yj5vw.bkt.clouddn.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202017-01-05%20%E4%B8%8B%E5%8D%887.11.55.png)


## UIPreviewInteraction

从上面的例子可以看出Peek and Pop的局限性，所以苹果在iOS10加入了新的API，可以让我们自定义实现更多功能。

#### 创建UIPreviewInteraction

	fileprivate var replyPreviewInteraction: UIPreviewInteraction!
	
	override func viewDidLoad() {
        super.viewDidLoad()
        
        ...
        
        replyPreviewInteraction = UIPreviewInteraction(view: view)
        replyPreviewInteraction.delegate = self
     }
	
	
#### 实现协议	

	extension ChatDetailViewController: UIPreviewInteractionDelegate {
		    ...
		}  
		
UIPreviewInteraction和Peek and Pop很像，也是有两段按压回调Preview和Commit

当手指按下时，调用：

	func previewInteractionShouldBegin(_ previewInteraction: UIPreviewInteraction) -> Bool {
	        return !replyViewControllerIsPresented
    }
    
之后连续调用：

    func previewInteraction(_ previewInteraction: UIPreviewInteraction, didUpdatePreviewTransition transitionProgress: CGFloat, ended: Bool) {
        var sourcePoint = previewInteraction.location(in: view) //得到当前按压的位置
        //transitionProgress 取值范围0到1，表示按压强度
        if ended {
            //do end
        }
    }    

手指加力继续按压，连续调用：

    func previewInteraction(_ previewInteraction: UIPreviewInteraction, didUpdateCommitTransition transitionProgress: CGFloat, ended: Bool) {
        if ended {
            //do end
        }
    }    
    
上面两个方法有相类似的参数，

可以通过previewInteraction.location(in: view)得到手指的位置，

可以通过transitionProgress得到按压的力度，取值0到1， 

可以通过ended判断是否已经结束按压

当有异常情况打断按压的话，如接到电话，会调用

    func previewInteractionDidCancel(_ previewInteraction: UIPreviewInteraction) {
        //do cancel
    }
    
完成如上述代码即可实现自定义的UIPreviewInteraction功能，本文中例子实现效果如下：
![7](http://oh36yj5vw.bkt.clouddn.com/3dtouchgif3.gif)
具体可参照Demo代码。
    