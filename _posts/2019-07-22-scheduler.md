---
layout: post
title: "定时任务调度器"
subtitle: 'scheduler'
author: "cslqm"
header-style: text
tags:
  - Linux
---

偶然看到一个人在写goCron，是一个cron工具，简单搜索一下相关文档，做下记录，以后有时间看。

原始的方案： crontab

多机方案
resque   创建任务管理任务状态。
resque-scheduler   调度器

The way Resque scheduler works is it runs a separate scheduler worker, which polls the configured schedules after every 5 seconds and checks if there are any tasks that should be ran. If any are found, it looks up the task definition and pushes the task to the configured queue for Resque to handle. From here on everything is handled like a regular background job.

Sidekiq  https://draveness.me/sidekiq

基于clockwork的新轮子
goCron  use clockwork