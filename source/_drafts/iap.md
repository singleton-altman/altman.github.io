---
title: iap
tags: [iOS, in-app-purchase, StoreKit, StoreKitTest]
---

#### 什么是iap
    - 概念
        - 基础
        - note
            - 
    - 分类
        - 一次性消耗品
        - 非一次性消耗品
        - 订阅
        - 自动续订
#### StoreKit实践
    - 通用
        - 基础
            - 逻辑
            - note
                - applicationUsername 作用
                - 4个status
                - `finishQueue`
        - sandbox环境
        - 验证逻辑
            - 客户端验证
                - note:
                    - 修改本地时间
            - 服务端验证
                - 回执信息存储
                - 轮询
    - 问题:
        - 订阅问题
        - 退款问题
#### server-to-server notification
    - 订阅类型状态变更
    - wwdc2020 新增退款推送
#### StoreKitTest
    - 配置

