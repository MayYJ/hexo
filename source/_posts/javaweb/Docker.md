#### 更换Docker镜像源

编辑 /etc/docker/daemon.json  的文件，如果没有就添加，docker版本要大于1.12

```json
{"registry-mirrors":[]}
```

[] 里面可以填一下的镜像源：

1. Docker官方中国区：https://registry.docker-cn.com 
2. 网易： http://hub-mirror.c.163.com 
3. ustc：https://docker.mirrors.ustc.edu.cn 
4. 去DaoCloud获取地址

#### 如何动态的添加端口映射

去[这里](https://www.ioio.pw/docker/2018/09/16/docker-container-port.html)看看吧