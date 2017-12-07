---
layout: post
title: 2017-12-07-LivePhoto转为视频，循环播放，来回播放
date: 2017-12-07
categories: blog
tags: [总结,知识管理]
description: 2017-12-07-LivePhoto转为视频，循环播放，来回播放
---
		

# 2017-12-07-LivePhoto转为视频，循环播放，来回播放


苹果的Live Photo功能推出后，并没有引起很大的反响，但我在使用中回看之前的照片，发现会不经意间拍到很多精彩的动态瞬间，加上后来苹果推出的回忆功能，来回播放及循环播放功能，使用后觉得效果很好，但这些功能都只能在iphone上使用，没法分享到朋友圈等常用平台，所以想尝试提取出Live Photo中的视频，自己实现来回播放，循环播放等功能，并以视频的形式存储与相册中

## Live Photo提取视频
Live Photo的存储形式是一张照片加3秒钟的视频文件，可以通过如下方式提取

    var movieURL = URL(fileURLWithPath: (NSTemporaryDirectory()).appending("tempVideo.mov"))
	let resources = PHAssetResource.assetResources(for: livePhotoAsset)
    for resource in resources {
        if resource.type == .pairedVideo {
            self.removeFileIfExists(fileURL: movieURL)
            PHAssetResourceManager.default().writeData(for: resource, toFile: movieURL as URL, options: nil) { (error) in
                if error != nil{
                    print("Could not write video file")
                } else {
                    print("Video has wrote to the file")        
                }
            }
            break
        }
    }
    
livePhotoAsset为PHAsset对象，从相册取出或拍照回掉的返回对象，如上述代码即可将视频写入到movieURL中，下面将其存储到相册

	func saveToAlbum(url : URL) {
        PHPhotoLibrary.shared().performChanges({
            PHAssetChangeRequest.creationRequestForAssetFromVideo(atFileURL: url)
        }, completionHandler: { (isSuccess, error) in
            if isSuccess {
                SVProgressHUD.showSuccess(withStatus: "保存成功，请到相册中查看")
            } else{
                SVProgressHUD.showError(withStatus: "保存失败：\(error!.localizedDescription)")
            }
        })
    }
    
    
## 视频倒放

Live Photo的视频已经提取出来了，想要实现循环播放，来回播放等功能，本质上都是对视频的操作，我们先尝试实现视频的倒叙播放

大体思路是通过AVAssetReader读取视频文件的每一帧，在通过AVAssetWriter倒叙合成新的视频文件

创建AVUtilities.swift类，添加如下代码：

	let videoQueue = DispatchQueue(label: "com.martinli.video")
	
	class AVUtilities {
	    
	    static func reverse(_ original: AVAsset, outputURL: URL, completion: @escaping (AVAsset) -> Void) {
	        
	        // Initialize the reader
	
	        var reader: AVAssetReader! = nil
	        do {
	            reader = try AVAssetReader(asset: original)
	        } catch {
	            print("could not initialize reader.")
	            return
	        }
	        
	        guard let videoTrack = original.tracks(withMediaType: AVMediaType.video).last else {
	            print("could not retrieve the video track.")
	            return
	        }
	        
	        let readerOutputSettings: [String: Any] = [kCVPixelBufferPixelFormatTypeKey as String : Int(kCVPixelFormatType_420YpCbCr8BiPlanarFullRange)]
	        let readerOutput = AVAssetReaderTrackOutput(track: videoTrack, outputSettings: readerOutputSettings)
	        reader.add(readerOutput)
	        
	        reader.startReading()
	        
	        // read in samples
	        
	        var samples: [CMSampleBuffer] = []
	        while let sample = readerOutput.copyNextSampleBuffer() {
	            samples.append(sample)
	        }
	        
	        // Initialize the writer
	        
	        let writer: AVAssetWriter
	        do {
	            writer = try AVAssetWriter(outputURL: outputURL, fileType: AVFileType.mov)
	        } catch let error {
	            fatalError(error.localizedDescription)
	        }
	        
	        let videoCompositionProps = [AVVideoAverageBitRateKey: videoTrack.estimatedDataRate]
	        let writerOutputSettings = [
	            AVVideoCodecKey: AVVideoCodecType.h264,
	            AVVideoWidthKey: videoTrack.naturalSize.width,
	            AVVideoHeightKey: videoTrack.naturalSize.height,
	            AVVideoCompressionPropertiesKey: videoCompositionProps
	            ] as [String : Any]
	        
	        let writerInput = AVAssetWriterInput(mediaType: AVMediaType.video, outputSettings: writerOutputSettings)
	        writerInput.expectsMediaDataInRealTime = false
	        writerInput.transform = videoTrack.preferredTransform
	        
	        let pixelBufferAdaptor = AVAssetWriterInputPixelBufferAdaptor(assetWriterInput: writerInput, sourcePixelBufferAttributes: nil)
	        
	        writer.add(writerInput)
	        writer.startWriting()
	        writer.startSession(atSourceTime: CMSampleBufferGetPresentationTimeStamp(samples.first!))
	        
	        videoQueue.async {
	            for (index, sample) in samples.enumerated() {
	                let presentationTime = CMSampleBufferGetPresentationTimeStamp(sample)
	                let imageBufferRef = CMSampleBufferGetImageBuffer(samples[samples.count - 1 - index])
	                while !writerInput.isReadyForMoreMediaData {
	                    Thread.sleep(forTimeInterval: 0.1)
	                }
	                pixelBufferAdaptor.append(imageBufferRef!, withPresentationTime: presentationTime)
	                
	            }
	            
	            writer.finishWriting {
	                DispatchQueue.main.async {
	                    completion(AVAsset(url: outputURL))
	                }
	            }
	        }
	        
	        
	    }
	}    
	
	
上述代码并没有什么难度，唯一要重视的是presentationTime这个参数，它表示视频每一帧显示的时间

	let presentationTime = CMSampleBufferGetPresentationTimeStamp(sample)

按正常顺序提取原视频每一帧的presentationTime

	let imageBufferRef = CMSampleBufferGetImageBuffer(samples[samples.count - 1 - index])
	
倒叙提取原视频每一帧的buffer

	pixelBufferAdaptor.append(imageBufferRef!, withPresentationTime: presentationTime)
	
将上述数据组合成新的视频帧

上述代码其实和使用多张图片组合成视频的原理是一样的。


## 循环播放，及来回播放

有了上述倒放的实践，循环播放及来回播放即不是问题了，只需要多几个循环即可

    static func loop(_ original: AVAsset, outputURL: URL, completion: @escaping (AVAsset) -> Void) {
        
        // Initialize the reader
        videoQueue.async {
        
            var reader: AVAssetReader! = nil
            do {
                reader = try AVAssetReader(asset: original)
            } catch {
                print("could not initialize reader.")
                return
            }
            
            guard let videoTrack = original.tracks(withMediaType: AVMediaType.video).last else {
                print("could not retrieve the video track.")
                return
            }
            
            let readerOutputSettings: [String: Any] = [kCVPixelBufferPixelFormatTypeKey as String : Int(kCVPixelFormatType_420YpCbCr8BiPlanarFullRange)]
            let readerOutput = AVAssetReaderTrackOutput(track: videoTrack, outputSettings: readerOutputSettings)
            reader.add(readerOutput)
            
            reader.startReading()
            
            // read in samples
            
            var samples: [CMSampleBuffer] = []
            while let sample = readerOutput.copyNextSampleBuffer() {
                samples.append(sample)
            }
            
            // Initialize the writer
            
            let writer: AVAssetWriter
            do {
                writer = try AVAssetWriter(outputURL: outputURL, fileType: AVFileType.mov)
            } catch let error {
                fatalError(error.localizedDescription)
            }
            
            let videoCompositionProps = [AVVideoAverageBitRateKey: videoTrack.estimatedDataRate]
            let writerOutputSettings = [
                AVVideoCodecKey: AVVideoCodecType.h264,
                AVVideoWidthKey: videoTrack.naturalSize.width,
                AVVideoHeightKey: videoTrack.naturalSize.height,
                AVVideoCompressionPropertiesKey: videoCompositionProps
                ] as [String : Any]
            
            let writerInput = AVAssetWriterInput(mediaType: AVMediaType.video, outputSettings: writerOutputSettings)
            writerInput.expectsMediaDataInRealTime = false
            writerInput.transform = videoTrack.preferredTransform
            
            let pixelBufferAdaptor = AVAssetWriterInputPixelBufferAdaptor(assetWriterInput: writerInput, sourcePixelBufferAttributes: nil)
            
            writer.add(writerInput)
            writer.startWriting()
            writer.startSession(atSourceTime: CMSampleBufferGetPresentationTimeStamp(samples.first!))
    
        
            for sample in samples {
                let presentationTime = CMSampleBufferGetPresentationTimeStamp(sample)
                let imageBufferRef = CMSampleBufferGetImageBuffer(sample)
                while !writerInput.isReadyForMoreMediaData {
                    Thread.sleep(forTimeInterval: 0.1)
                }
                pixelBufferAdaptor.append(imageBufferRef!, withPresentationTime: presentationTime)
            }
            var time = CMSampleBufferGetPresentationTimeStamp(samples.last!)
            time.value += 20
            for sample in samples {
                var presentationTime = time
                presentationTime.value += CMSampleBufferGetPresentationTimeStamp(sample).value
                let imageBufferRef = CMSampleBufferGetImageBuffer(sample)
                while !writerInput.isReadyForMoreMediaData {
                    Thread.sleep(forTimeInterval: 0.1)
                }
                pixelBufferAdaptor.append(imageBufferRef!, withPresentationTime: presentationTime)
            }
            
            writer.finishWriting {
                DispatchQueue.main.async {
                    completion(AVAsset(url: outputURL))
                }
            }
        }
    }
    
    static func playback(_ original: AVAsset, outputURL: URL, completion: @escaping (AVAsset) -> Void) {
        
        videoQueue.async {
            // Initialize the reader
            
            var reader: AVAssetReader! = nil
            do {
                reader = try AVAssetReader(asset: original)
            } catch {
                print("could not initialize reader.")
                return
            }
            
            guard let videoTrack = original.tracks(withMediaType: AVMediaType.video).last else {
                print("could not retrieve the video track.")
                return
            }
            
            let readerOutputSettings: [String: Any] = [kCVPixelBufferPixelFormatTypeKey as String : Int(kCVPixelFormatType_420YpCbCr8BiPlanarFullRange)]
            let readerOutput = AVAssetReaderTrackOutput(track: videoTrack, outputSettings: readerOutputSettings)
            reader.add(readerOutput)
            
            reader.startReading()
            
            // read in samples
            
            var samples: [CMSampleBuffer] = []
            while let sample = readerOutput.copyNextSampleBuffer() {
                samples.append(sample)
            }
            
            // Initialize the writer
            
            let writer: AVAssetWriter
            do {
                writer = try AVAssetWriter(outputURL: outputURL, fileType: AVFileType.mov)
            } catch let error {
                fatalError(error.localizedDescription)
            }
            
            let videoCompositionProps = [AVVideoAverageBitRateKey: videoTrack.estimatedDataRate]
            let writerOutputSettings = [
                AVVideoCodecKey: AVVideoCodecType.h264,
                AVVideoWidthKey: videoTrack.naturalSize.width,
                AVVideoHeightKey: videoTrack.naturalSize.height,
                AVVideoCompressionPropertiesKey: videoCompositionProps
                ] as [String : Any]
            
            let writerInput = AVAssetWriterInput(mediaType: AVMediaType.video, outputSettings: writerOutputSettings)
            writerInput.expectsMediaDataInRealTime = false
            writerInput.transform = videoTrack.preferredTransform
            
            let pixelBufferAdaptor = AVAssetWriterInputPixelBufferAdaptor(assetWriterInput: writerInput, sourcePixelBufferAttributes: nil)
            
            writer.add(writerInput)
            writer.startWriting()
            writer.startSession(atSourceTime: CMSampleBufferGetPresentationTimeStamp(samples.first!))
        
        
            for sample in samples {
                let presentationTime = CMSampleBufferGetPresentationTimeStamp(sample)
                let imageBufferRef = CMSampleBufferGetImageBuffer(sample)
                while !writerInput.isReadyForMoreMediaData {
                    Thread.sleep(forTimeInterval: 0.1)
                }
                pixelBufferAdaptor.append(imageBufferRef!, withPresentationTime: presentationTime)
            }
            var time = CMSampleBufferGetPresentationTimeStamp(samples.last!)
            time.value += 20
            for (index, sample) in samples.enumerated() {
                var presentationTime = time
                presentationTime.value += CMSampleBufferGetPresentationTimeStamp(sample).value
                let imageBufferRef = CMSampleBufferGetImageBuffer(samples[samples.count - 1 - index])
                while !writerInput.isReadyForMoreMediaData {
                    Thread.sleep(forTimeInterval: 0.1)
                }
                pixelBufferAdaptor.append(imageBufferRef!, withPresentationTime: presentationTime)
                
            }
            
            writer.finishWriting {
                DispatchQueue.main.async {
                    completion(AVAsset(url: outputURL))
                }
            }
        }
    }
    
循环播放即循环两次组合，或者也可以通过参数控制次数，来回播放即一次循环加一次倒放

需要注意的是两个循环衔接之处不能存在相同的presentationTime，否则会写不进去，所以我在循环之间presentationTime加20

	var time = CMSampleBufferGetPresentationTimeStamp(samples.last!)
    time.value += 20
这是通过打印的到的原视频的presentationTime时间间隔


[Demo地址](https://github.com/martinLilili/LYLivePhotos)    
	