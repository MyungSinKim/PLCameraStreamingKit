# PLCameraStreamingKit

PLCameraStreamingKit 是一个适用于 iOS 的 RTMP 直播推流 SDK，可高度定制化和二次开发。特色是支持 iOS Camera 画面捕获并进行 H.264 硬编码，以及支持 iOS 麦克风音频采样并进行 AAC 硬编码；同时，还根据移动网络环境的多变性，实现了一套可供开发者灵活选择的编码参数集合。借助 PLCameraStreamingKit，开发者可以快速构建一款类似 [Meerkat](https://meerkatapp.co/) 或 [Periscope](https://www.periscope.tv/) 的手机直播应用。

## 功能特性

- [x] 硬件编码
- [x] 多码率可选
- [x] H.264 视频编码
- [x] AAC 音频编码
- [x] 支持前后摄像头
- [x] 自动对焦支持
- [x] 手动调整对焦点支持
- [x] 闪光灯开关
- [x] 多分辨率编码支持
- [x] HeaderDoc 文档支持
- [x] 内置生成安全的 RTMP 推流地址
- [x] ARM64 支持
- [x] 支持 RTMP 协议直播推流


## 内容摘要

- [快速开始](#快速开始)
	- [配置工程](#配置工程)
	- [示例代码](#示例代码)
- [编码参数](#编码参数)
- [文档支持](#文档支持)
- [功能特性](#功能特性)
- [系统要求](#系统要求)
- [版本历史](#版本历史)

## 快速开始

先来看看 PLCameraStreamingKit 接入的步骤

### 配置工程

- 配置你的 Podfile 文件，添加如下配置信息

```shell
pod 'PLCameraStreamingKit'
```

- 安装 CocoaPods 依赖

```shell
pod install
```

或

```shell
pod update
```

- Done! 运行你工程的 workspace

### 示例代码

在需要的地方添加

```Objective-C
#import <PLCameraStreamingKit/PLCameraStreamingKit.h>
```

```PLCameraStreamingSession``` 是核心类，你只需要关注并使用这个类就可以完成通过摄像头推流、预览的工作

推流前务必要先检查摄像头 / 麦克风的授权，并记得设置预览界面，```StreamingSession``` 的创建需要 Stream 对象和 Publish host

```Objective-C
// streamJSON 是从服务端拿回的
//
// 从服务端拿回的 streamJSON 结构如下：
//    @{@"id": @"stream_id",
//      @"title": @"stream_title",
//      @"hub": @"hub_name",
//      @"publishKey": @"publish_key",
//      @"publishSecurity": @"dynamic", // or static
//      @"disabled": @(NO),
//      @"profiles": @[@"480p", @"720p"],    // or empty Array []
//      @"hosts": @{
//              @"publish": @{
//                      @"rtmp": @"rtmp_publish_host"
//                      },
//              @"play": @{
//                      @"rtmp": @"rtmp_play_host",
//                      @"hls": @"hls_play_host"
//                      }
//              }
//      }
NSDicationary *streamJSON;
PLStream *stream = [PLStream streamWithJSON:streamJSON];
// 授权后执行
void (^permissionBlock)(void) = ^{
        PLCameraStreamingConfiguration *configuration = [PLCameraStreamingConfiguration defaultConfiguration];
        self.session = [[PLCameraStreamingSession alloc] initWithConfiguration:configuration
                                                                        stream:stream
                                                              videoOrientation:AVCaptureVideoOrientationPortrait];
        self.session.delegate = self;
        self.session.previewView = self.view;
};

void (^noPermissionBlock)(void) = ^{ // 处理未授权情况 };
    
// 检查摄像头是否有授权
PLAuthorizationStatus status = [PLCameraStreamingSession cameraAuthorizationStatus];
   
if (PLAuthorizationStatusNotDetermined == status) {
    [PLCameraStreamingSession requestCameraAccessWithCompletionHandler:^(BOOL granted) {
    // 回调确保在主线程，可以安全对 UI 做操作
        granted ? permissionBlock() : noPermissionBlock();
    }];
} else if (PLAuthorizationStatusAuthorized == status) {
    permissionBlock();
} else {
	noPermissionBlock();
}
```

推流操作

```Objective-C
// 开始推流，无论 security policy 是 static 还是 dynamic，都无需再单独计算推流地址
[self.session startWithCompleted:^(BOOL success) {
	// 这里的代码在主线程运行，所以可以放心对 UI 控件做操作
	if (success) {
		// 连接成功后的处理
		// 成功后，在这里才可以读取 self.session.pushURL，start 失败和之前不能确保读取到正确的 URL
	} else {
    	// 连接失败后的处理
	}
}];

// 停止推流
[self.session stop];
```

销毁推流 session
```Objective-C
[self.session destroy];
```

## 编码参数

移动端因网络环境不稳定及用户电量宝贵等原因，并不建议直接使用最高码率和分辨率来做推流，以最佳编码参数来做设置可以带来更好的推流效果和用户体验。

你无需辛苦的一个个参数设置，```PLCameraStreamingKit``` 提供了一个编码配置的类来帮你快速完成配置。

```Objective-C
// 初始化编码配置类的实例需要的两个参数

// 视频横纵比及分辨率
typedef NS_ENUM(NSUInteger, PLStreamingDimension) {
    PLStreamingDimension_16_9__416x234,
    PLStreamingDimension_16_9__480x270,
    PLStreamingDimension_16_9__640x360,
    PLStreamingDimension_16_9__960x540,
    PLStreamingDimension_16_9__1280x720,
    PLStreamingDimension_16_9__1920x1080,
    PLStreamingDimension_4_3__400x300,
    PLStreamingDimension_4_3__480x360,
    PLStreamingDimension_4_3__640x480,
    PLStreamingDimension_4_3__960x720,
    PLStreamingDimension_4_3__1280x960,
    PLStreamingDimension_4_3__1920x1140,
    PLStreamingDimension_UserDefine,
    PLStreamingDimension_Default = PLStreamingDimension_4_3__640x480
};

// 推流质量
// fps: 12, profile level: baseline30, video bitrate: 150Kbps, audio sample rate: 44MHz, audio bitrate: 96Kbps
extern NSString *kPLStreamingQualityLow1;

// fps: 15, profile level: baseline30, video bitrate: 264Kbps, audio sample rate: 44MHz, audio bitrate: 96Kbps
extern NSString *kPLStreamingQualityLow2;

// fps: 15, profile level: baseline30, video bitrate: 350Kbps, audio sample rate: 44MHz, audio bitrate: 96Kbps
extern NSString *kPLStreamingQualityLow3;

// fps: 30, profile level: baseline31, video bitrate: 512Kbps, audio sample rate: 44MHz, audio bitrate: 96Kbps
extern NSString *kPLStreamingQualityMedium1;

// fps: 30, profile level: baseline31, video bitrate: 800Kbps, audio sample rate: 44MHz, audio bitrate: 96Kbps
extern NSString *kPLStreamingQualityMedium2;

// fps: 30, profile level: baseline31, video bitrate: 1000Kbps, audio sample rate: 44MHz, audio bitrate: 96Kbps
extern NSString *kPLStreamingQualityMedium3;

// fps: 30, profile level: main30, video bitrate: 1200Kbps, audio sample rate: 44MHz, audio bitrate: 128Kbps
extern NSString *kPLStreamingQualityHigh1;

// fps: 30, profile level: main30, video bitrate: 1500Kbps, audio sample rate: 44MHz, audio bitrate: 128Kbps
extern NSString *kPLStreamingQualityHigh2;

// fps: 30, profile level: main30, video bitrate: 2000Kbps, audio sample rate: 44MHz, audio bitrate: 128Kbps
extern NSString *kPLStreamingQualityHigh3;
```

你只需要明确以上两者，便可以直接获取到最佳编码配置。

```Objective-C
// 默认情况下，PLCameraStreamingKit 会使用 4:3 的 640x480 分辨率，及 kPLStreamingQualityMedium1 作为参数初始化编码配置类的实例.
PLCameraStreamingConfiguration *configuration = [PLCameraStreamingConfiguration defaultConfiguration];


// 当然你也可以自己指定，比如你希望输出直播视频是 16:9 的 960x540 的分辨率，并且你已经明确你需要的视频质量为 High1，你可以这样来设置编码配置
PLCameraStreamingConfiguration *configuration = [PLCameraStreamingConfiguration configurationWithDimension:PLStreamingDimension_16_9__960x540 quality:kPLStreamingQualityHigh1];

// 当已有的分辨率无法满足你的需求时，你可以自己定义视频的大小
PLCameraStreamingConfiguration *configuration = [PLCameraStreamingConfiguration configurationWithUserDefineDimension:CGSizeMake(width, height) quality:kPLStreamingQualityHigh1];
```

在创建好编码配置对象后，就可以用它来初始化 ```PLCameraStreamingSession``` 了。

### Quality 具体参数

| Quality | FPS | ProfileLevel | Video BitRate(Kbps) | Audio Samplerate(MHz)) | Audio BitRate(Kbps) |
|---|---|---|---|---|---|
|kPLStreamingQualityLow1|12|Baseline 30|150|44|96|
|kPLStreamingQualityLow2|15|Baseline 30|264|44|96|
|kPLStreamingQualityLow3|15|Baseline 30|350|44|96|
|kPLStreamingQualityMedium1|30|Baseline 31|512|44|96|
|kPLStreamingQualityMedium2|30|Baseline 31|800|44|96|
|kPLStreamingQualityMedium3|30|Baseline 31|1000|44|96|
|kPLStreamingQualityHigh1|30|Main 30|1200|44|128|
|kPLStreamingQualityHigh2|30|Main 30|1500|44|128|
|kPLStreamingQualityHigh2|30|Main 30|2000|44|128|

## 文档支持

PLCameraStreamingKit 使用 HeaderDoc 注释来做文档支持。
开发者无需单独查阅文档，直接通过 Xcode 就可以查看接口和类的相关信息，减少不必要的麻烦。

![Encode 推荐](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/header-doc.png?raw=true)

## 系统要求

- iOS Target : >= iOS 7

## 版本历史

- 1.2.7 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.2.7.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.2.7.md))
    - 修复断网后停止流时导致的 UI 卡死
    - 修复停止推流时概率出现的 crash 问题
    - 添加 `- (void)destroy` 方法到 `PLCameraStreamingSession`
    - 暂且去除 iOS 8 编码，待进一步优化后再做开放
    - 修复反复进出持有 streamingSession 的 viewController 出现 IO 错误或者 crash 的问题
- 1.2.6 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.2.6.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.2.6.md))
    - 修复 iOS 8 以下推流时，编码导致的崩溃问题
- 1.2.5 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.2.5.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.2.5.md))
    - 更新了 Quality 配置
- 1.2.4 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.2.4.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.2.4.md))
    - 添加全新的 iOS 8 硬编码器，减少编码延时
- 1.2.3 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.2.3.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.2.3.md))
    - 更新了 `PLStream` 类结构
    - 添加新的 session 创建接口
    - 移除 publishHost，`PLStream` 对象支持自动生成推流地址，请搭配服务端 SDK 至少 1.2.0 或以上版本使用
- 1.2.2 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.2.2.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.2.2.md))
    - 更新麦克风资源调用方式，添加进入后台后自动停用麦克风功能
- 1.2.1 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.2.1.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.2.1.md))
	- 修复初次授权摄像头后预览界面为黑屏的问题
- 1.2.0 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.2.0.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.2.0.md))
	- 添加了 `PLStream` 类，支持 `Coding` 协议便于打包存储
	- 更新 `StreamingSession` 创建方法，借助传递 `PLStream` 对象再无需推流时等待服务端生成推流地址
	- 完善 HeaderDoc 文档
- 1.1.8 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.1.8.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.1.8.md))
	- 添加摄像头开启和关闭的操作，便于开发者控制 AVCaptureSession
- 1.1.7 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.1.7.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.1.7.md))
	- 添加推流质量字段，方便开发者指定推流质量
	- 移除原配置中的网络选项
	- 默认情况下关闭了 PLLog
	- 添加摄像头开始运行的通知
	- 添加麦克风开始运行的通知
- 1.1.6 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.1.6.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.1.6.md))
	- 尝试修复音频变声问题
- 1.1.5 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.1.5.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.1.5.md))
	- 修复了重连后无法播放的问题
- 1.1.4 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.1.4.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.1.4.md))
	- 更新了码率和采样率。现在统一使用 128Kbps 和 44100KHZ，用以解决噪音的问题
	- 更新了网络 socket。对 socket 在调试情况下出现的 SIGPIPE 进行了处理，不会再出现
	- 更新了网络状态的返回情况。当不是主动调用 close 接口导致的断开都将以 error 状态通知 delegate
- 1.1.3 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.1.3.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.1.3.md))
	- 更新了获取视频截图的方法
	- 添加 HeaderDoc 格式注释文档
- 1.1.2 ([Release Notes](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/ReleaseNotes/release-notes-1.1.2.md) && [API Diffs](https://github.com/pili-engineering/PLCameraStreamingKit/blob/master/APIDiffs/api-diffs-1.1.2.md))
	- 添加自定视频大小的编码配置
	- 添加进入后台后自动停止推流的支持
	- 添加保持常亮的开关
	- 添加获取当前录制截屏的功能
	- 修复进入后台导致崩溃的问题
	- 修复切换前置摄像头失效的问题
	- 添加 Release Notes 说明每个版本变动
	- 添加 API Diffs 说明具体修改细节
- 1.1.1
	- 修复 release 版本无法推流的 bug
	- 内存优化，比 1.1.0 减少 6% 左右内存消耗
- 1.1.0
	- 重构接口
	- 优化编码参数
	- 提供不同网络和分辨率下的多种配置可选
	- 添加消息通知，便于监听
	- 兼顾 arc 及非 arc 的工程
- 1.0.0
	- 发布 CocoaPods 版本