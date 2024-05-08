---
title: Hexo搭建遇到的问题
categories: [Hexo, Next]
tags: [Hexo, Next]
date: 2021-03-10 10:33:47
---

### 部署后css文件加载错误：
>`_config.yml` 中的 url错误，需要填写为自己的`域名或者git page`地址。

### 隐藏文章
 - 安装插件 `yarn add hexo-hide-posts`
 - `_config.yml` 中配置 
    ```
    hide_posts:
    # 可以改成其他你喜欢的名字
        filter: hidden
        # 指定你想要传递隐藏文章的位置，比如让所有隐藏文章在存档页面可见
        # 常见的位置有：index, tag, category, archive, sitemap, feed, etc.
        # 留空则默认全部隐藏
        public_generators: []
        # 为隐藏的文章添加 noindex meta 标签，阻止搜索引擎收录
        noindex: true 
    ```
 - 在文章的属性中定义 hidden: true 即可隐藏文章。

### zsh: command not found: hexo 
``` *.sh
$ echo \ export PATH=~/.npm-global/bin:\$PATH >> ~/.zshrc
$ source ~/.zshrc
```
### 更换主题造成的乱码
 - 原因: hexo 在5.0之后把 swig 删除了
 - 解决: 项目内 `$ npm i hexo-renderer-swig`
 - 重编一下项目即可(CI的情况下直接提交代码)
<!-- more -->
### 自定义配置时候.yml文件被覆盖问题
 - 原因: 因为我们的`NexT`主题是使用`npm`加载的, 所以资源都是统一被放在`node_modules`内, 如果做自定义配置而修改了内部文件的话, 在下次`$ npm install`的时候就会被覆盖.
 - 解决: `$ cp node_modules/hexo-theme-[theme name]/_config.yml _config.[theme name].yml`. `[theme name]` 需要替换成自己所用的主题名字, 比如`next`.
 - 在新拷贝出来的文件内做修改, `hexo`会自动选择.
### 文章字数统计
 - 安装: `$ npm install hexo-symbols-count-time --save`
 - `_config.yml`设置:
 ```
symbols_count_time: 
    symbols: true # 文章字数统计  
    time: true # 文章阅读时长  
    total_symbols: true # 站点总字数统计  
    total_time: false # 站点总阅读时长  
    exclude_codeblock: false # 排除代码字数统计
 ```
 - 可选: `_config.yml`中`language`设置为`zh_CN`
### 设置自定义域名
 - 先去申请一个域名吧, 比如 `loser.work` [阿里云](https://wanwang.aliyun.com/?spm=5176.19720258.J_8058803260.55.c9a82c4aVT0pTf), 便宜的一年几块钱.
 - `github` 对应的 `blog` 项目内, 打开 `settings` 下的 `pages`, 找到 `Custom domain`, 输入 `loser.work`, 然后 `save`. 
 - 打开阿里云[阿里云](https://wanwang.aliyun.com/?spm=5176.19720258.J_8058803260.55.c9a82c4aVT0pTf)`控制台`, 找到`域名`, `解析`, `添加记录`, `记录类型` 选择 `cname`, `主机记录` 输入 `www`, `记录值`输入 `git page` 地址. `确认
 - 使用 [vercel](https://vercel.com/) 自动部署, 并把域名切换为自己申请的即可
 - 打开浏览器, 键入 `loktar.com.cn`, 就可以看到你的博客了.
 - 感谢 `github`. 
  
--- 
> Next主题已经不在使用
##### `NexT`主题配置 `_config.next.yml`
  - 代码块: `codeblock`字段下按需配置.
  - 先[体验](https://theme-next.js.org/highlight/), 后选择.
  - favicon: `favicon` 字段下设置. 对应文件拖入项目内, 配置好路径即可~.
  - 推荐一个[工具网站](https://realfavicongenerator.net)
  - 菜单配置: `menu` 字段下, 可以配置对应的标题和icon. 使用`$ hexo new page [name]`就可以生成对应的模版, `name`和`menu`下的配置匹配到就可以啦. 
  - `icon` 可以在[这里](https://fontawesome.com)找到
  - <bold>注意</bold>: 设置`tags`, `NexT`会自动匹配`page`, 只需要给`tag page`下的`index.md`中的`type`设置为`tags`即可.
```
---
title: tags
date: 2021-08-03 13:44:18
type: tags
tags: [技术, iOS]
---
```
 - 设置头像: `avatar`字段下, 配置文件地址. 
 - `rounded:` 支持自动切圆角.
 - 支持`gif`

    


    
 

