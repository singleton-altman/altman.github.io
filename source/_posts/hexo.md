---
title: Hexo搭建遇到的问题
---

1. 更换主题造成的乱码
    - 原因: hexo在5.0之后把swig给删除了
    - 解决: 项目内 `$ npm i hexo-renderer-swig`
    - 重编一下项目即可(CI的情况下直接提交代码)
2. 自定义配置时候.yml文件被覆盖问题
    - 原因: 因为我们的`NexT`主题是使用`npm`加载的, 所以资源都是统一被放在`node_modules`内, 如果做自定义配置而修改了内部文件的话, 在下次`$ npm install`的时候就会被覆盖.
    - 解决: `$ cp node_modules/hexo-theme-[theme name]/_config.yml _config.[theme name].yml`. `[theme name]` 需要替换成自己所用的主题名字, 比如`next`.
    - 在新拷贝出来的文件内做修改, `hexo`会自动选择.
3. 文章字数统计
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
4. `NexT`主题配置 `_config.next.yml`
    - 代码块: `codeblock`字段下按需配置.先[体验](https://theme-next.js.org/highlight/), 后选择.(🤔, 一玩一天).
    - favicon: `favicon` 字段下设置. 对应文件拖入项目内, 配置好路径即可~. 推荐一个[工具网站](https://realfavicongenerator.net)
    - 菜单配置: `menu` 字段下, 可以配置对应的标题和icon. 使用`$ hexo new page [name]`就可以生成对应的模版, `name`和`menu`下的配置匹配到就可以啦. easy~. `icon` 可以在[这里](https://fontawesome.com)找到.
    - 设置头像: `avatar`字段下, 配置文件地址. `rounded:` 支持自动切圆角.

    


    
 

