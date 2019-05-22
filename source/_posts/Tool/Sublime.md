---
title: Sublime
date: 2019-03-24 16:32:40
tags: sublime
---

#### Install Package 的时候出现 There are no packages available for installation

这是因为Sublime 无法读取可安装列表，也就是无法访问到设置的网址

Preference -> PackageSetting-> setting-Default中的

```java
"channels": [
		"https://packagecontrol.io/channel_v3.json"
	]
```

修改的方式是：

将这个文件下载到本地  https://packagecontrol.io/channel_v3.json

Preference -> PackageSetting-> setting-User 里面覆盖上面的设置,比如我自己的设置

```java
"channels":
	[
		"E:\\software\\Sublime Text 3\\channel_v3.json"
	],
```



