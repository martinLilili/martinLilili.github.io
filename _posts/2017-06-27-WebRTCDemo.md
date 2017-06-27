---
layout: post
title: WebRTCDemo
date: 2016-12-27
categories: blog
tags: [总结,知识管理]
description: WebRTCDemo
---
		

## WebRTCDemo

最近公司有需求做局域网视频通讯，于是调研了webrtc并实现一个简单demo，本文主要记录demo的实现逻辑，以便以后使用时尽快上手。

本文主要参考 ：

[iOS下音视频通信-基于WebRTC](http://www.jianshu.com/p/c49da1d93df4)

[iOS下WebRTC音视频通话（二）-局域网内音视频通话](http://www.jianshu.com/p/aa9802c4296f)

		
		
一些基本概念这里不讲，关于什么是ice，sdp参考上面的文章，直接讲思路及代码

### 思路

要建立p2p连接，我们需要：发起端A，接受端B，信令服务。

这里信令服务的作用主要是在A和B建立连接前做一些信息的交换，当A和B真正建立连接后就不需要信令服务了，通用做法是建立一个第三方服务器，A和B通过socket连接到服务器做中转，我这里因为只做局域网，为了简单，用GCDAsyncSocket将A和B直接连接起来。

步骤：

1. A和B创建connection，初始化本地视频流，设置ice server（STUN Server）获取ice candidate，（局域网不需要ice穿墙，这里ice server可以设置为空）
2. B开始监听端口，A通过socket直连B
3. A创建offer，并将自己的sdp（session description）通过socket发送给B
4. B收到socket发来的offer，创建answer并将自己的sdp通过socket回复给A
5. A将自己的ice candidate通过socket发送给B，B将自己的ice candidate通过socket发送给A （局域网视频通信也需要这一步骤）
6. 此时A和B就已经建立了p2p连接，其他的webrtc都为我们做好了

### 代码

#### 发起端A

##### 初始化

    let factory = RTCPeerConnectionFactory()
    
    var localStream : RTCMediaStream?
    
    var connection : RTCPeerConnection?
    var tcpSocket : GCDAsyncSocket?
    var accepttcpSocket : GCDAsyncSocket?
    
    var remoteVideoTrack : RTCVideoTrack?
    
    var isoffering = false
    
    var tempStr = ""
    
    var setLocal = false
    var setRemote = false
    var sentICE = false
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        //如果你需要安全一点，用到SSL验证，那就加上这句话。还没有仔细研究，先加上
        RTCPeerConnectionFactory.initializeSSL()
        
        //初始化socket
        tcpSocket = GCDAsyncSocket(delegate: self, delegateQueue: DispatchQueue.main)

        //创建本地视频流，并显示到页面上
        createLocalStream()

        let ICEServers : [RTCICEServer] = [RTCICEServer]()
	//        ICEServers.append(defaultSTUNServer(url: "stun:stun.l.google.com:19302"))
	//        ICEServers.append(defaultSTUNServer(url: "turn:numb.viagenie.ca"))
        
        let constraints = RTCMediaConstraints(mandatoryConstraints: nil, optionalConstraints: [RTCPair(key: "DtlsSrtpKeyAgreement", value: "true")])
        connection = factory.peerConnection(withICEServers: ICEServers, constraints: constraints, delegate: self)
        
        //加入本地视频流
        connection?.add(localStream)
    
    }

    func createLocalStream() {
        localStream = factory.mediaStream(withLabel: "ARDAMS")
        let audioTrack = factory.audioTrack(withID: "ARDAMSa0")
        localStream?.addAudioTrack(audioTrack)
        
        let deviceArray = AVCaptureDevice.devices(withMediaType: AVMediaTypeVideo)
        let device = deviceArray?.last as? AVCaptureDevice
        let authStatus = AVCaptureDevice.authorizationStatus(forMediaType: AVMediaTypeVideo)
        if authStatus == .restricted || authStatus == .denied {
            
        } else {
            if (device != nil) {
                let capturer : RTCVideoCapturer = RTCVideoCapturer(deviceName: (device as AnyObject).localizedName)
                let videoSource = factory.videoSource(with: capturer, constraints: localVideoConstraints())
                let videoTrack = factory.videoTrack(withID: "ARDAMSv0", source: videoSource)
                localStream?.addVideoTrack(videoTrack)
                
                
                let localVideoView = RTCEAGLVideoView(frame: CGRect(x:0, y:0, width: 300, height: 300 * 640 /
                    480))
                localVideoView.transform = CGAffineTransform(scaleX: -1, y: 1)
                videoTrack?.add(localVideoView)
                self.view.addSubview(localVideoView)
                
            }
        }
        
    }

这段代码主要是声明一些属性，设置ice Server， 创建connection，因为是做局域网，所已ICEServers里面是空的，这些代码写完，在界面上就可以看到自己的画面了

##### 发送offer 

    @IBAction func offerBtnClicked(_ sender: UIButton) {
        connection?.createOffer(with: self, constraints: offerOranswerConstraint())
        
    }
    
通过按钮发起offer。connection调用createOffer方法，之后会会掉RTCSessionDescriptionDelegate的方法didCreateSessionDescription：

    func peerConnection(_ peerConnection: RTCPeerConnection!, didCreateSessionDescription sdp: RTCSessionDescription!, error: Error!) {
        print("didCreateSessionDescription")
        
        if !setLocal {
            peerConnection.setLocalDescriptionWith(self, sessionDescription: sdp)
            setLocal = true
        }
        
    }
		
在这个方法里我们设置自己的sdp：setLocalDescription，之后会回调RTCSessionDescriptionDelegate的方法didSetSessionDescriptionWithError 如下：

    func peerConnection(_ peerConnection: RTCPeerConnection!, didSetSessionDescriptionWithError error: Error!) {
        print("didSetSessionDescriptionWithError")
        
        if peerConnection.signalingState == RTCSignalingHaveLocalOffer {

            print("send offer")
            var dic = [String:String]()
            dic["event"] = "offer"
            dic["sdp"] = peerConnection.localDescription.description
            
            let data : Data? = try? JSONSerialization.data(withJSONObject: dic)
            do {
                try tcpSocket!.connect(toHost: SocketModel.targetHost, onPort: SocketModel.targetPort, withTimeout: -1)
            } catch  {
                print("Error connect:\(error)")
            }
            
            var str = String(data: data!, encoding: String.Encoding.utf8)!
            str += "|"
            tcpSocket?.write(str.data(using: .utf8)!, withTimeout: -1, tag: 0)
            
            isoffering = true
            
        }         
    }
    
此时peerConnection.signalingState为RTCSignalingHaveLocalOffer，在这里我们使用socket将自己的sdp发送给B如上：

我们先组织我们的数据：

	var dic = [String:String]()
	dic["event"] = "offer"
	dic["sdp"] = peerConnection.localDescription.description

建立连接：SocketModel.targetHost为B的IP，SocketModel.targetPort为B监听的端口号

            do {
                try tcpSocket!.connect(toHost: SocketModel.targetHost, onPort: SocketModel.targetPort, withTimeout: -1)
            } catch  {
                print("Error connect:\(error)")
            }
	
发送数据：

            var str = String(data: data!, encoding: String.Encoding.utf8)!
            str += "|"
            tcpSocket?.write(str.data(using: .utf8)!, withTimeout: -1, tag: 0)
            
注意：为了防止分包和连包问题，我们在数据结尾加了一个"|"

##### 收到B回复的answer            	
	
    func socket(_ sock: GCDAsyncSocket, didRead data: Data, withTag tag: Int) {
        
        tcpSocket?.readData(withTimeout: -1, tag: 0)
        accepttcpSocket?.readData(withTimeout: -1, tag: 0)
        
        if let str = String(data: data, encoding: .utf8) {
            print("Message didReceiveData : \(str)")
            
            for c in str.characters {
                if c != "|" {
                    tempStr += String(c)
                    
                } else {
                    parseDic(msg: tempStr)
                    tempStr = ""
                }
                
            }
            
        }
       
    }
    
    func parseDic(msg : String)  {
        
        var parsedJSON: Any?
        do {
            parsedJSON = try JSONSerialization.jsonObject(with: msg.data(using: .utf8)!, options: JSONSerialization.ReadingOptions.mutableLeaves)
        } catch let error {
            print(error)
        }
        if let dic = parsedJSON as? [String : String] {
            if dic["event"] == "offer" {
					...
                
            } else if dic["event"] == "answer" {
                if !isoffering {
                    return
                }
                let remoteSdp = RTCSessionDescription(type: "answer", sdp: dic["sdp"])
                connection?.setRemoteDescriptionWith(self, sessionDescription: remoteSdp)
            } else if dic["event"] == "candidate" {
               ...
            }
        }
    }
	
我们在GCDAsyncSocket的didRead回调方法中处理接收到的数据.

首先通过"|"分包，然后交给parseDic方法处理，主要看下parseDic里面的代码：当解析event为answer时，我们即收到了B的sdp，并调用setRemoteDescriptionWith设置B的sdp：

	let remoteSdp = RTCSessionDescription(type: "answer", sdp: dic["sdp"])
	                connection?.setRemoteDescriptionWith(self, sessionDescription: remoteSdp)		
	                
##### 发送ice candidate及接收ice candidate

当我们调用setLocalDescriptionWith后还会触发RTCPeerConnectionDelegate的代理方法gotICECandidate ： 

    func peerConnection(_ peerConnection: RTCPeerConnection!, gotICECandidate candidate: RTCICECandidate!) {
        print("gotICECandidate = \(candidate)")
        if sentICE {
            return
        }
        
        var dic = [String:String]()
        dic["event"] = "candidate"
        dic["sdp"] = candidate.sdp
        dic["label"] = String(candidate.sdpMLineIndex)
        dic["id"] = candidate.sdpMid
        
        let data : Data? = try? JSONSerialization.data(withJSONObject: dic)
        var str = String(data: data!, encoding: String.Encoding.utf8)!
        str += "|"
        
        if isoffering {
            tcpSocket?.write(str.data(using: .utf8)!, withTimeout: -1, tag: 0)
        } else {
            accepttcpSocket?.write(str.data(using: .utf8)!, withTimeout: -1, tag: 0)
        }
        
        sentICE = true
    }	                

在这里我们得到自己的ice candidate并发送给对方如上

同样，当B获得了它的ice candidate信息后也会发送给A，处理逻辑如下：

    func parseDic(msg : String)  {
        
        var parsedJSON: Any?
        do {
            parsedJSON = try JSONSerialization.jsonObject(with: msg.data(using: .utf8)!, options: JSONSerialization.ReadingOptions.mutableLeaves)
        } catch let error {
            print(error)
        }
        if let dic = parsedJSON as? [String : String] {
            if dic["event"] == "offer" {
               ...   
            } else if dic["event"] == "answer" {
                ...
            } else if dic["event"] == "candidate" {
                let candidate = RTCICECandidate(mid: dic["id"], index: Int(dic["label"]!)!, sdp: dic["sdp"])
                connection?.add(candidate)
            }
        }
    }

当解析到event为candidate时，创建candidate并添加到connection中，如上

##### 显示远端视频流 

当上面我们调用setRemoteDescriptionWith后，就会调用RTCPeerConnectionDelegate的方法addedStream来让我们处理远程视频流，当A和B完成了上述的sdp交换和ice candidate交换后，视频流就可以显示出来了：

    func peerConnection(_ peerConnection: RTCPeerConnection!, addedStream stream: RTCMediaStream!) {
        print("addedStream")
        DispatchQueue.main.async {
            if stream.videoTracks.count > 0{
                self.remoteVideoTrack = stream.videoTracks.last as? RTCVideoTrack
                
                let remoteVideoView = RTCEAGLVideoView(frame: CGRect(x:0, y:0, width: 100, height: 100 * 640 /
                    480))
                remoteVideoView.backgroundColor = UIColor.red
                remoteVideoView.transform = CGAffineTransform(scaleX: -1, y: 1)
                self.remoteVideoTrack?.add(remoteVideoView)
                
                self.view.addSubview(remoteVideoView)
            }
        }
    }
    
#### 接收端B

##### 初始化  

与发起端相同

##### 监听端口

    @IBAction func acceptBtn(_ sender: UIButton) {
        
        do {
            try tcpSocket?.accept(onPort: SocketModel.targetPort)
        } catch  {
            
        }
    }  
这里用一个按钮开启监听

##### 收到offer

    func parseDic(msg : String)  {
        
        var parsedJSON: Any?
        do {
            parsedJSON = try JSONSerialization.jsonObject(with: msg.data(using: .utf8)!, options: JSONSerialization.ReadingOptions.mutableLeaves)
        } catch let error {
            print(error)
        }
        if let dic = parsedJSON as? [String : String] {
            if dic["event"] == "offer" {
                if isoffering {
                    return
                }
                let remoteSdp = RTCSessionDescription(type: "offer", sdp: dic["sdp"])
                connection?.setRemoteDescriptionWith(self, sessionDescription: remoteSdp)
                connection?.createAnswer(with: self, constraints: offerOranswerConstraint())
                
                
            } else if dic["event"] == "answer" {
                ...
            } else if dic["event"] == "candidate" {
                ...
            }
        }
    }  
    
socke的处理逻辑和A差不多，都是在parseDic中解析数据，当event为offer时，我们即收到了A发来的sdp，这里首先设置RemoteDescription  

	let remoteSdp = RTCSessionDescription(type: "offer", sdp: dic["sdp"])           
	connection?.setRemoteDescriptionWith(self, sessionDescription: remoteSdp) 
	
然后创建answer ：

	 connection?.createAnswer(with: self, constraints: offerOranswerConstraint())
	 
setRemoteDescriptionWith会回调RTCSessionDescriptionDelegate的方法didSetSessionDescriptionWithError，此时peerConnection.signalingState为RTCSignalingHaveRemoteOffer，我们不做任何事情。

调用createAnswer后，会回调RTCSessionDescriptionDelegate的didCreateSessionDescription方法，这里我们与发起端一样调用setLocalDescriptionWith设置自己的sdp，这时又会回调didSetSessionDescriptionWithError方法。peerConnection.signalingState为RTCSignalingStable，在此我们向A回复answer及sdp：

    func peerConnection(_ peerConnection: RTCPeerConnection!, didSetSessionDescriptionWithError error: Error!) {
        print("didSetSessionDescriptionWithError")
        
        if peerConnection.signalingState == RTCSignalingHaveLocalOffer {
			...
        } else if peerConnection.signalingState == RTCSignalingStable {
            if !isoffering {
                var dic = [String:String]()
                dic["event"] = "answer"
                dic["sdp"] = peerConnection.localDescription.description
                let data : Data? = try? JSONSerialization.data(withJSONObject: dic)
                var str = String(data: data!, encoding: String.Encoding.utf8)!
                str += "|"
                accepttcpSocket?.write(str.data(using: .utf8)!, withTimeout: -1, tag: 0)
            }
        }
        
    }
}


##### 发送ice candidate及接收ice candidate

与发起端一样

##### 显示远端视频流 

与发起端一样


如上就可以实现视频通信了。




## demo

地址：[WebrtcDemo](https://github.com/martinLilili/WebrtcDemo)

使用方法：

准备两台手机A和B，连接相同局域网

修改Demo的IP和端口号，并分别运行于A和B

	struct SocketModel {
	    static let targetHost = B的IP
	    static let targetPort : UInt16 = B监听的端口号
	}

B点击开启监听

A点击发起offer

如上即可实现A和B通信


