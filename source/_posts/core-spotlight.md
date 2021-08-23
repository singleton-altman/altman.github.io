---
title: core-spotlight APP内搜索
tags:
  - spotlight
  - iOS
  - 搜索
date: 2021-08-23 11:33:47
---


**简介**
通过iOS的spotlight~~下拉搜索~~基于关键字快速查找, 点击item后打开APP, 代码内也可以支持自动跳转到对应页面. 关键字可以自设, 增加APP的曝光率.
<!-- more -->
**现状**
大部分的产品经理不懂iOS, 也不懂iPhone用户. 某些知名APP, 只做了itunes connect的下载搜索关键词覆盖, 竟然不做iOS特性分析...用户下载了以后就变成了一个僵尸APP放在手机里面, 然后被自动清理掉, 真的是太可惜了.
**使用**
- 此处用的是wwdc里面的swiftUI demo内的地标数据.
- 目标1: 让用户在搜索地标名字的时候显示 icon, name和description. 
- 目标2: 用户点击对应的item, APP接受到对应的事件和信息, 基于需求完成后续操作.
**主要代码**
- 生成关键字索引和被搜索到后的显示内容:
```
let itemAttributeSet: CSSearchableItemAttributeSet!
if #available(iOS 14.0, *) {
    itemAttributeSet = CSSearchableItemAttributeSet(contentType: .text)
} else {
    itemAttributeSet = CSSearchableItemAttributeSet(itemContentType: "text")
}
itemAttributeSet.identifier = "landmark_\(mark.id)"
itemAttributeSet.keywords = [mark.name, "landmark", "旅游", mark.park]
itemAttributeSet.title = mark.name
itemAttributeSet.contentDescription = mark.description
```
- 生成索引item
`CSSearchableItem(uniqueIdentifier: "com.hi.cage.CoreSpotlightDemo_\(mark.id)", domainIdentifier: "landmark", attributeSet: itemAttributeSet)`
    注意此处的 uniqueIdentifier, 需要 `唯一`
- 建立索引:
```
CSSearchableIndex.default().indexSearchableItems(items) { error in
    if let errMessage = error?.localizedDescription {
        debugPrint(errMessage)
    }
}
```
    items是一个数组, 所以我们一次可以建立`多个`索引.
- 接收用户在spotloght的点击事件
```
func application(_ application: UIApplication, continue userActivity: NSUserActivity, restorationHandler: @escaping ([UIUserActivityRestoring]?) -> Void) -> Bool {
    if (userActivity.activityType == CSSearchableItemActionType) {
        let id = userActivity.userInfo?[CSSearchableItemActivityIdentifier]
        debugPrint("userActivity id = \(id)")
        /// ...
    }
    return true
}
```
- 移除索引
```
CSSearchableIndex.default().deleteSearchableItems(withIdentifiers: ["com.hi.cage.CoreSpotlightDemo_\(mark.id)"]) { error in
    if let errMessage = error?.localizedDescription {
        debugPrint(errMessage)
    } else {
        debugPrint("deleteSearchableItems(withIdentifiers:) success")
    }
}
```
也可以基于 demain 删除
`/// CSSearchableIndex.default().deleteSearchableItems(withDomainIdentifiers: <#T##[String]#>, completionHandler: <#T##((Error?) -> Void)?##((Error?) -> Void)?##(Error?) -> Void#>)`
**结束**
功能很简单, 但是很实用, 接入 spotlight以后, APP活跃度可以有很明显的提升.
已经上传[gitthub](https://github.com/aioser/CoreSpotlightDemo.git), enjoy~