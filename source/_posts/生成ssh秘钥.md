---
title: 生成ssh秘钥
date: 2018-02-25 23:46:56
tags: ssh秘钥
---

1. 首先检查有没有ssh秘钥 

   ls -al ~/.ssh

2. 生成秘钥

   ssh-keygen -t rsa -C "your_email@example.com"

