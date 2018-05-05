---
title: linux软件推荐
date: 2018-02-26 22:48:28
tags: linux软件
categories: linux
---

##### linux端的ssr

[github地址](https://github.com/erguotou520/electron-ssr)

---

##### 跨平台百度网盘

[github地址](https://github.com/iikira/BaiduPCS-Go#%E4%B8%BE%E4%B8%80%E4%BA%9B%E4%BE%8B%E5%AD%90)

1. 操作方法github上很详细
2. 直接在命令行操作方便

---

#####自制webapp

1. 首先安装appifier

   [github地址](https://github.com/quanglam2807/appifier/releases)

   安装deb包

   `sudo dpkg -i Appifier_x.x.x_amd64.deb`

2. 安装npm包

   sudo apt-get install npm

3. 安装electron

   npm install electron --save-dev --save-exact

4. 安装electron-installer-debian包

   npm install -g electron-installer-debian

5. 然后是在appifier上生成文件

6. 然后通过以下命令生成deb包，安装(以下命令安装或者通过软件管理器安装)

   electron-installer-debian --src /home/jingle/Desktop/Wechat-linux-x64/ --dest /home/jingle/Desktop/wechat/ --arch amd64

   dpkg -i dir

在这里可能会遇到类似错误：

```
Error: No Description or ProductDescription provided
    at getOptions (/usr/lib/node_modules/electron-installer-debian/src/installer.js:149:11)
    at getDefaults.then.defaults (/usr/lib/node_modules/electron-installer-debian/src/installer.js:418:23) 'Error: No Description or ProductDescription provided\n    at getOptions (/usr/lib/node_modules/electron-installer-debian/src/installer.js:149:11)\n    at getDefaults.then.defaults (/usr/lib/node_modules/electron-installer-debian/src/installer.js:418:23)'
```

只需要在/home/.../Desktop/Gmail-linux-x64/resources/app.asar.unpacked/package.json文件里面按照格式添加description和productDescription属性。

**参考链接**

http://ijingle.cc/2018/02/11/appifier-build-webapp/

---