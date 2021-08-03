---
title: iOS @功能始末
date: 2021-08-03 11:44:17
categories: [技术, iOS, 艾特, YYText]
tags: [技术, iOS, 艾特, YYText]
---

- ## 前言
    前段时间产品给了一个在`@`好友的功能
    - 用户在单独输入`@`符号的时候, 触发好友列表页面弹出, 然后在用户点击好友A的时候在输入框内自动设置文案: `@A `; 
    - 在这段文案后输入`删除`的时候, 整段文案一起被移除.
    - 用户发出这段文本以后, 其他用户可以点击前往用户A的个人主页
- ## 分析
    - 在 UITextfield 的 delegate 内监听关键字 `@` 和` `, 分别实现弹出好友列表和移除文案操作, 这个很简单.
    - 在用户选择好友后, 文案拼接拼装到 UITextfield 的 text 内; 在监听到 ` `输入的时候, 通过`字符串匹配`或者`正则表达式`, 移除光标前一段符合规则的 `@A `文本.
    - 在发出文本的时候, 基于字符串查找逻辑, 将特殊文本和用户id做关联, 将整个数据包发送给后端. 然后进行转发.
    - 以上, 这个功能就已经实现了. 
- ## 问题
    上面的方案虽然可以解决基础的问题, 但是(PS1: 凡事总有一个但是; PS2: 大家都讨厌这个但是...)
    - 如果好友A的昵称在存在空格 ` ` (文本: `@A B `)的情况下, 以上方案的删除就会失效. 我们也不能无脑暴力的按照 `@` 和 ` ` 进行删除文本, 这样 `BUG🐛` 会更加严重.
    - 因为需要考虑数据的跨端传输, 而因为用户昵称可能存在 emoji 字符或者其他特殊字体, 所以使用位置计算告知后端index进行匹配的话, 可能会有不同语言的index计算错误问题. 比如 [swift](https://swift.org)~~谁会不爱 `swift` 那~~ 内的 `😊.count` 为 1, 而 `objective-C` 和 `java `是 2. 所以这个方案怎么看都不那么合理...
    这时候, 多么希望有方案可以给某一段文案设置(标记)成一个整体的方案啊.
- ## 解决
    原生的控件已经满足不了或者需要大量的定制化才能我们的需求了, 目光只能看向开源轮子. 这时候 `YYText` 引入了眼帘.
    `YYText`的数据流`demo`中, 使用了 `YYTextBackedString` 实现 `@":)` 替换 😊 制作表情菜单. 这就给了鶸需求实现的灵感.
    - 首先自定义一个 `YYTextView` 的 `textParser`. 重写 `- (BOOL)parseText:(NSMutableAttributedString *)text selectedRange:(NSRangePointer)selectedRange` 方法
    - 定义一段特殊字符串 `BackedString` , 里面包含了用户昵称 `A` 和 他的用户 `id`, 然后对用户对可见文本 `FrontString` 为 `@A `. 配合 `NSMutableAttributedString` 扩展方法 `- yy_setTextBackedString: range:`, 对 `YYTextView` 的可见内容进行替换. 这样我们就把所需要的全部信息藏入了富文本内. 
    - 使用 `- yy_setTextBinding: range: `将这段可见文本设置成一个整体, 从而实现整体删除.
    - 使用 `- yy_setTextHighlight: range: `设置局部文本高亮.
    - ... 其他的完全可以基于自己的需求自定义.

#### 回到 `BackedString`.
    - 这一段自定义文本尽量稍微复杂一点, 不然被黑产破解的话, 线上就会出现一些奇奇怪怪的现象了...
    - 这里面还有一个问题, 使用以上方案的时候, 如果用户点击这段文本 `copy` 的时候, 会把密文拿出来. 所以这时候就需要重写 `YYText` 内的 `copy` 方法, 把 `FrontString` 拿出来.
    ```
    /// Save current selected attributed text to pasteboard.
    - (void)_copySelectedTextToPasteboard {
        if (_allowsCopyAttributedString) {
            NSAttributedString *text = [_innerText attributedSubstringFromRange:_selectedTextRange.asRange];
            if (text.length) {
                [UIPasteboard generalPasteboard].yy_AttributedString = text;
            }
        } else {
            NSString *string = [_innerText attributedSubstringFromRange:_selectedTextRange.asRange].string;
            if (string.length) {
                [UIPasteboard generalPasteboard].string = string;
            }
        }
    }
    ```
#### 关于 YYText
    目前iOS 15已经来了, `YYText`年久失修, 在某些情况下会出一些奇怪的BUG, 能不用就不用吧. 