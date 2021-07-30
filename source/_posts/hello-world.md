---
title: Hexo搭建遇到的问题
---

1. 更换主题造成的乱码
    - 原因: hexo在5.0之后把swig给删除了
    - 解决: 项目内 `$ npm i hexo-renderer-swig`
    - 重编一下项目即可(CI的情况下直接提交代码)
