BootStrap是怎么把EventloopGroup 、Eventloop、Channel、ChannelHandlerContext、ChannelPipeline结合起来的

我们从Bootstrap的connect方法进去

```java
    public ChannelFuture connect(SocketAddress remoteAddress) {
        if (remoteAddress == null) {
            throw new NullPointerException("remoteAddress");
        } else {
            //这里判断了在connect之前是否添加了Channelhandler
            this.validate();
            return this.doResolveAndConnect(remoteAddress, this.config.localAddress());
        }
    }
```

```java
    private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    // 这里的initAndRegister()方法很重要，所以下面是详细解释
        ChannelFuture regFuture = this.initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.isDone()) {
            return !regFuture.isSuccess() ? regFuture : this.doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
        } else {
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        promise.setFailure(cause);
                    } else {
                        promise.registered();
                        Bootstrap.this.doResolveAndConnect0(channel, remoteAddress, localAddress, promise);
                    }

                }
            });
            return promise;
        }
    }
```

```java

    final ChannelFuture initAndRegister() {
        Channel channel = null;

        try {
            //这里是通过我们前面传给Bootstrp 的SocketChannel类型反射创建channel
            channel = this.channelFactory.newChannel();
            // 这里对channel进行了初始化，这里也比较重要，下面也进行了详细解释
            this.init(channel);
        } catch (Throwable var3) {
            if (channel != null) {
                channel.unsafe().closeForcibly();
                return (new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE)).setFailure(var3);
            }

            return (new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE)).setFailure(var3);
        }

        ChannelFuture regFuture = this.config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        return regFuture;
    }
```

```java
 // 这里使用的代码是Bootstrp的init方法
 void init(Channel channel) throws Exception {
        ChannelPipeline p = channel.pipeline();
        p.addLast(new ChannelHandler[]{this.config.handler()});
        Map<ChannelOption<?>, Object> options = this.options0();
        synchronized(options) {
            setChannelOptions(channel, options, logger);
        }

        Map<AttributeKey<?>, Object> attrs = this.attrs0();
        synchronized(attrs) {
            Iterator var6 = attrs.entrySet().iterator();

            while(var6.hasNext()) {
                Entry<AttributeKey<?>, Object> e = (Entry)var6.next();
                channel.attr((AttributeKey)e.getKey()).set(e.getValue());
            }

        }
    }
```