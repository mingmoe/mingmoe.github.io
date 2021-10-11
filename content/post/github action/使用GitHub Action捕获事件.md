+++
title = "使用GitHub Action捕获事件"
date = 2021-10-11T20:02:02+08:00
draft = false
archives = 2021
categories = "github action"
tags = ["github action","git"]
+++

GitHub Action可以捕获Github的一些事件（不是所有）

通常分两部分:
 - 通过某些事件触发workflow
 - 在jobs\steps获取事件参数


<!--more-->

第一部分算是比较简单的:
```yaml
name: XXX

# 设置一些触发条件
on:
  workflow_run:
    workflows: [ "CI And CD" ]
    types: [ completed ]
```
github[官方文档](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#example-using-multiple-events-with-activity-types-or-configuration)有叙述


-----

第二部分，可能有些小问题，但总体问题不大:

```yaml
name: Test Event Catch

# 在workflows "CI And CD"完成时触发这个workflow
on:
  workflow_run:
    workflows: [ "CI And CD" ]
    types: [ completed ]

jobs:
  update-badge:
    runs-on: ubuntu-latest

    # 获取事件参数
    env:
      pte_check_suite_url: ${{ github.event.workflow_run.check_suite_url }}

    steps:
      - name: Print event result
        run: echo ${pte_check_suite_url}

```
注意: github.event后不能直接获取事件的work load。还需要加上事件类型:
```yaml
# 错误的做法
jobs:
  env:
    pte_check_suite_url: ${{ github.event.check_suite_url }}

# 正确的做法
jobs:
  env:
    pte_check_suite_url: ${{ github.event.workflow_run.check_suite_url }}

```

关于事件所携带的数据和可以使用的事件，可以见GitHub 官方文档:

https://docs.github.com/en/actions/learn-github-actions/events-that-trigger-workflows

完.
