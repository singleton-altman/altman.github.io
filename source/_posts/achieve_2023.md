---
title: 2023年终总结
tags:
  - iOS
date: 2024-04-10 11:33:47
security: true
---

# 2023 年
> 主要负责了狼人杀 iOS 端项目从 3.5.4 版本到目前的 3.8.0的功能开发和代码迭代优化。

### 主要功能包括:
1. 头像框加载优化、排行榜代码重构
2. 自定义埋点 SDK，休闲区列表、跟房、消息等列表数据埋点
3. APP 整体 UI 布局修改
3. 礼物面板新增金色礼物、冠名礼物、礼物背包等功能
4. 休闲区推荐引导、休闲区列表元素支持后端动态配置，并且列表内兼容未来配置的其他类型休闲区房间、个人主页背景图裁切显示
6. IAP支持连续包月
7. 优化现有的优惠券、APP评分、公告功能

### 自研功能:
1. fastlane 脚本打包
2. xcodegen 项目构建

### 具体实现

#### 资源加载
1. 资源下载框架 [LRSWebDownloader](http://ce.jiamiantech.com/p/LRSWebLoader/d/LRSWebLoader/git/blob/master/LRSWebLoader/Classes/LRSWebLoaderManager.h)
   > 目前狼人杀项目中的下载功能比较简陋, 使用的是 NSURLSession 的 download方法, 仅支持简单的资源下载
   - 支持同时下载的最大并发量
      <details>
       <summary>展开代码</summary>
       <pre><code>
       [_config addObserver:self forKeyPath:NSStringFromSelector(@selector(maxConcurrentDownloads)) options:0 context:nil];
       _downloadQueue = [NSOperationQueue new];
       _downloadQueue.maxConcurrentOperationCount = _config.maxConcurrentDownloads;
       </code></pre>
      </details>
   - 返回 token, 支持 cancel
      <details>
       <summary>展开代码</summary>
       <pre><code>
       NS_ASSUME_NONNULL_BEGIN
       @interface LRSWebLoaderManager : NSObject
       @property (nonatomic, copy, readonly, nonnull) LRSWebDownloaderConfig *config;
       + (nonnull instancetype)sharedDownloader;
       - (instancetype)initWithConfig:(nullable LRSWebDownloaderConfig *)config;
       - (nullable LRSWebDownLoaderloadToken *)downloadImageWithURL:(NSURL *)URL progress:(nullable LRSWebLoaderProgressBlock)progressBlock completed:(LRSWebLoaderCompletedBlock)completedBlock;
       @end
       NS_ASSUME_NONNULL_END
       </code></pre>
      </details>
   - 缓存使用同一个 URL 的下载进度和完成回调, 避免触发多次无效HTTP事件
     <details>
       <summary>展开代码</summary>
       <pre><code>
      	self.URLOperations[URL] = operation;
       downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
       [self.downloadQueue addOperation:operation];
       </code></pre>
     </details>
2. Lottie/图片/其他格式动画素材显示公共组件 [LRSWebCacheLoader](http://ce.jiamiantech.com/p/LRSWebCacheLoader/d/LRSWebCacheLoader/git/tree/master/LRSWebCacheLoader/Classes)
    - 依赖 SDWebImage/LRSWebDownloader下载资源
       <details>
        <summary>展开代码</summary>
        <pre><code>
        - (void)lrs_createOrUpdateWithURL:(NSURL *)URL complete:(void(^)(LOTComposition *scene, NSError *error))complete {
            if (URL.isFileURL) {
                [self _resetLotAnimationViewWithFilePath:[URL.absoluteString stringByReplacingOccurrencesOfString:URL.scheme withString:@""] complete:complete];
            } else {
                LRSWebDownLoaderloadToken *token = [LRSWebLottieLoader downloadFileWithURL:URL completed:^(NSString * _Nullable filePath, NSError * _Nullable error) {
                    [self lrs_associateValue:nil withKey:"kLOTDownLoadKey"];
                    if (error) {
                        [self stop];
                        if (complete) {
                            complete(nil, error);
                        }
                        NSLog(@"lrs_createOrUpdateWithURL: %@", error);
                        return;
                    }
                    [self _resetLotAnimationViewWithFilePath:filePath complete:complete];
                }];
                [self lrs_associateValue:token withKey:"kLOTDownLoadKey"];
            }
        }
       </code></pre>
       </details>
    - 使用对应的显示框架展示动画/素材
3. 头像框加载框架 [LRSAvatarFrameSource](http://ce.jiamiantech.com/p/LRSAvatarFrameSource/d/LRSAvatarFrameSource/git/tree/master/LRSAvatarFrameSource/Classes)
    - 对后端配置进行缓存
    - 构建一个[时间线对象](http://ce.jiamiantech.com/p/LRSAvatarFrameSource/d/LRSAvatarFrameSource/git/blob/master/LRSAvatarFrameSource/Classes/LRSAvatarFrameTimeLine.m), 存储列表内动画 View 以及下次重启的时间点, 支持指定间隔重复动画
    - 通过缓存数据和 UI 重用解决排行榜卡顿严重问题

#### 埋点框架
1. 埋点SDK基础逻辑 [LRSMobData](http://ce.jiamiantech.com/p/LRSMobData)
    - 支持单个点/多个数据同时埋点 [LRSMobClick](http://ce.jiamiantech.com/p/LRSMobData/d/LRSMobData/git/blob/master/LRSMobData/Classes/LRSMobClick.h)
    - 设置多套上报检查节点[API](http://ce.jiamiantech.com/p/LRSMobData/d/LRSMobData/git/blob/master/LRSMobData/Classes/LRSMobClick.m)
        - APP 状态切换
        - 写入数据后延迟 x 秒
        - 数据写入最大量强制上报
    - 专用数据库, 避免被其他数据污染
2. 埋点业务扩展逻辑 [LRSAPPEvent](http://ce.jiamiantech.com/p/langren-ios/d/langren-ios/git/tree/master/LRSAPPEvent), 用于列表数据上报和缓存控制
    - 抽离共同业务逻辑, 定义[页面/cell/数据](http://ce.jiamiantech.com/p/langren-ios/d/langren-ios/git/blob/master/LRSAPPEvent/LRSAPPEvent/LRSAppEventListProtocol.h)三个协议
    - 使用RAC hook 页面的出现消失以及其他满足需求的节点 [API](http://ce.jiamiantech.com/p/langren-ios/d/langren-ios/git/blob/master/LRSAPPEvent/LRSAPPEvent/LRSAPPEvent.m)
    - 使用样例
        - [跟房弹框](http://ce.jiamiantech.com/p/langren-ios/d/langren-ios/git/blob/master/LangRen/Business/ViewController/FollowView/FollowFriendRoomInfoViewController.m)
        - [列表cell](http://ce.jiamiantech.com/p/langren-ios/d/langren-ios/git/blob/master/LangRen/Business/View/GameRoomView/GameAlertCustomView/FollowFriendAlertTableViewCell.h)
        - [数据元](http://ce.jiamiantech.com/p/langren-ios/d/langren-ios/git/blob/master/LangRen/Business/Model/ResponseModel/RoomInfoResponse.h)

#### fastlane 脚本
> [fastlane](https://fastlane.tools/)是一个 iOS APP 自动化工具, 支持自动化打包, TF, APP Store提包等功能 

1. [fastlane](http://ce.jiamiantech.com/p/langren-ios/d/langren-ios/git/tree/master/fastlane) 特点
    - 简单易读
    - 自带 action 时间统计，自动生成 report.xml
    - 插件丰富 match、upload_dsym_to_bugly、 versioning、fir_cli等， 同时支持自定义 action
    - 支持 Env，用于变量配置
    - 支持多种 lane， 比如现在已有的 app version 设置和 dev 包发布 firim
2. [原有打包脚本](http://ce.jiamiantech.com/p/langren-ios/d/langren-ios/git/blob/master/output/autoarchive.sh)的缺陷： 
    - 直接使用`xcodebuild`, 环境变量、Scheme名字、Token等配置项全部在定义在脚本内， 业务逻辑和 log、Q&A杂糅， 可读性差。
    - 没有统计功能，无法查看打包速度
    - 没有插件使用， 无法扩展

#### xcodegen脚本
> [xcodegen](https://github.com/yonaskolb/XcodeGen) 是一个自动化构建 xcode 项目的工具
1. xcodegen 可以解决的问题
    - 主要：多人开发情况下的 .xcodeproj 冲突
    - 可选：生成单一的 target 用于开发，加速 xcode link
    - 快速构建项目，CI 可用
2. 现有问题
    - 增删文件导致的 xcodeproj 文件冲突，耗费大量精力解决
    - 多target共享 xcodeproj 文件，工程文件庞大，导致 link 缓慢甚至卡死
        ![xcodeproj](./xcodegen/xcodeproj.jpg)
    - 某些自定义 sdk 依赖， 比如 RNFramework引入需要配置[embed and sign](https://juejin.cn/post/7055596252519464990), 以及 RN 开发依赖的静态资源
    <div>
        <img src='https://cdn.jsdelivr.net/gh/singleton-altman/blog_picx@master/pics/RNFramework.4goofk26va00.webp' width="100%" alt="RN">
        <img src='https://cdn.jsdelivr.net/gh/singleton-altman/blog_picx@master/pics/RN依赖.5iuubmyyg780.webp' width="50%" alt="依赖">
    </div>
    - 多个 target 的 info.plist 相同配置需要逐个编写， 比如: 相机、相册、麦克风等权限申请描述文案
3. 目前开发状态
    私有库已经可用，如[狼人杀充值 UI](http://ce.jiamiantech.com/p/LRSRechargeUI/d/LRSRechargeUI/git/blob/master/Example/project.yml)
4. 后续会先在私有库模板工程内使用， 狼人杀主项目依赖整理完毕后可选使用

### 未来规划:
1. 从事狼人杀后续版本开发和维护
2. 使用 [CocoaAsyncSocket](https://github.com/robbiehanson/CocoaAsyncSocket) 或者 [SocketRocket](https://github.com/facebookincubator/SocketRocket) 作为 NSStream 实现 IM 功能的备选方案，排查解决 IM 断连的原因并尝试提供解决方案
   - CocoaAsyncSocket 已经考察可用， 数据通信正常。
   - SocketRocket 是 meta 出品，可能更加可靠。
3. 优化 UI 布局代码， 解决对 LRSDeviceKit 的缩放比例显式计算问题。

## 结束语
感谢观看
