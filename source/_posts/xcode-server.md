---
title: Xcode Server/macOS Server 保姆级教程
tags:
  - CI
  - xcode
  - iOS
  - xcode server
  - 持续集成
date: 2021-08-09 18:06:53
---

- #### 简介
    `Xcode Server` 是苹果官方提供的[持续集成](https://baike.baidu.com/item/持续集成/6250744?fr=aladdin)方案. 
    - 优点: `xcode` 自带工具, 部署简单, ~~有手就能玩~~ 甚至支持 `swift` 脚本.
    - 缺点: ~~iOS 完美无缺~~ 只支持 apple 体系.
<!-- more -->

- #### 前提
    - 准备一个 iOS 的项目
        - 需要设置了 `remote URL` 的项目, 否则无法使用.
        - `$ git remote add origin [URL]`
    - 一个<bold>付费</bold>的 apple 开发者账号. 
    - 去这个[macOS APP store](https://apps.apple.com/cn/app/macos-server/id883878097?mt=12)下载一个 `macOS server` 软件, 安装, 打开启动

- #### 步骤
    以下步骤全部在 `xcode` 内进行
    - `command ,` 打开偏好设置, 
    - 选择` Accounts`, `+` 号, 选择 `xcode server`, 选中在 `macOS server` 内相同主机名字的哪个栏目, `next ->`,
    - 输入 `macOS server` 内的可用 用户(可以自己创建)账户密码. 然后 add 进去
    - 偏好设置内选择 `server&bot`, 启动. 
    - 项目的操作菜单内, `Product`, `create bot`:
        - `server` 选择刚才创建的, name随意写. `next ->`
        - `verify SSH`, 项目的 `git` 地址, 公钥验证. `next ->`
        - `configuration`, 选择 `scheme`, `export` 可选, 如果你是企业级项目进行分发的话, 需要选择 `plist`, 填写所需配置. `next ->`
        - `integrate`, 一般选择 `on commit`, 在收到 `commit` 的时候进行编译; 也可以选择 `manually`, 自己主动选择. `clean`, 我选的 `once a weak`, 自己按需选择. `next ->`
        - `build for`, 按需选择. `next ->`
        - 配置证书和描述文件, 选择配置项目打包证书 `add to server`. `next ->` 
        - 环境变量, 按需填写. ~~我没有, 我不写~~
        - `configure bot triggers`, 配置 编译前后脚本和邮件事件. 
            - `pre-integration script`
                - 打包编译前的脚本
                    - 一般来说, 我们的项目是 [cocoapods](https://cocoapods.org) 管理的
                    ```
                        export LANG=en_US.UTF-8
                        cd ${XCS_PRIMARY_REPO_DIR}
                        pwd
                        rm -f Podfile.lock
                        /usr/local/bin/pod install
                    ```
            - `post-intefration script`
                - 打包编译后的脚本
                    - 上报[蒲公英](https://www.pgyer.com/doc/api#uploadApp)
                    ```
                    curl -F "file=@$XCS_PRODUCT" \
                    -F "uKey=you uKey" \
                    -F "_api_key=you _api_key" \
                    https://qiniu-storage.pgyer.com/apiv1/app/upload
                    ```
                    - 提交 [app store](https://help.apple.com/itc/apploader/#/apdATD1E53-D1E1A1303-D1E53A1126)
                    ```
                    #!/bin/sh
                    altoolPath="/Applications/Xcode.app/Contents/Applications/Application Loader.app/Contents/Frameworks/ITunesSoftwareService.framework/Versions/A/Support/altool"
                    # 填入你的Apple ID
                    USERNAME="你的AppleID"
                    #  需要去Apple ID账户生成 App 专用密码
                    PASSWORD="App 专用密码"

                    "$altoolPath" --validate-app -f ${XCS_PRODUCT} -u ${USERNAME} -p ${PASSWORD} -t ios
                    "$altoolPath" --upload-app -f ${XCS_PRODUCT} -u ${USERNAME} -p ${PASSWORD} -t ios

                    ```
            - `issure email`
                - 按需配置
            - `Periodic Email Report`
                - 按需配置

- #### 遇到的问题
    - `macOS server` 付费: 自行解决, [苹果开发者中心的beta](https://developer.apple.com)版本不要钱.
    - `product` 下 没有 `create bot` 选项: `accounts` 里面添加 `xcode server` 了吗?

- #### `xcode server RESTful API!`
    - [官方文档](https://developer.apple.com/library/archive/documentation/Xcode/Conceptual/XcodeServerAPIReference)
    - 也有大佬提供了一个 [ xcode server SDK ](https://github.com/buildasaurs/XcodeServerSDK) 可以用
    - 推荐体验[xcodeserverapidocs](https://xcodeserverapidocs.docs.apiary.io), 参照各种配置, 便于上手. 
    - 本地的 server地址: [base URL](https://127.0.0.1:20343/api/bots)

#### enjoy~🐸
        
    


