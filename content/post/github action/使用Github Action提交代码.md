+++
title = "使用Github Action提交代码"
date = 2021-10-11T19:53:17+08:00
draft = false
archives = 2021
categories = "programming helper"
tags = ["github action","git"]
+++

<!--TODO-->
笔者最近在研究如何使用GitHub来提交代码。

只需要使用github官方提供的github动作就可以了:
```yaml
jobs:
   push-code:

     steps:
      # 检出代码
      - name: Checkout repository
        uses: actions/checkout@v2

      # 配置git
      - name: Config git
        run: |
          git config --global user.name   mingmoe
          git config --global user.email  sudo.free@qq.com

      - name: Push Code
        run: |
            git add .
            git commit -m "update"
            git push
```
即可提交代码。

<!--more-->


完.
