# Netty 源码分析

## 初始化

#### group

```
    public ServerBootstrap() { }

    private ServerBootstrap(ServerBootstrap bootstrap) {
        super(bootstrap);
        childGroup = bootstrap.childGroup;
        childHandler = bootstrap.childHandler;
        synchronized (bootstrap.childOptions) {
            childOptions.putAll(bootstrap.childOptions);
        }
        synchronized (bootstrap.childAttrs) {
            childAttrs.putAll(bootstrap.childAttrs);
        }
    }
```

```
    public ServerBootstrap group(EventLoopGroup group) {
        return group(group, group);
    }

    public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
        super.group(parentGroup);
        if (childGroup == null) {
            throw new NullPointerException("childGroup");
        }
        if (this.childGroup != null) {
            throw new IllegalStateException("childGroup set already");
        }
        this.childGroup = childGroup;
        return this;
    }
```

```
AbstractBootstrap.java

    public B group(EventLoopGroup group) {
        if (group == null) {
            throw new NullPointerException("group");
        }
        if (this.group != null) {
            throw new IllegalStateException("group set already");
        }
        this.group = group;
        return (B) this;
    }
```

#### channel

```
AbstractBootstrap.java

    public B channel(Class<? extends C> channelClass) {
        if (channelClass == null) {
            throw new NullPointerException("channelClass");
        }
        return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
    }
```

```
ReflectiveChannelFactory.java

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }
```

```
    public B channelFactory(ChannelFactory<? extends C> channelFactory) {
        if (channelFactory == null) {
            throw new NullPointerException("channelFactory");
        }
        if (this.channelFactory != null) {
            throw new IllegalStateException("channelFactory set already");
        }

        this.channelFactory = channelFactory;
        return (B) this;
    }
```

#### option

```
AbstractBootstrap.java

    public <T> B option(ChannelOption<T> option, T value) {
        if (option == null) {
            throw new NullPointerException("option");
        }
        if (value == null) {
            synchronized (options) {
                options.remove(option);
            }
        } else {
            synchronized (options) {
                options.put(option, value);
            }
        }
        return (B) this;
    }
 ```
 
#### childOption
 
 ```
 ServerBootstrap.java
 
     public <T> ServerBootstrap childOption(ChannelOption<T> childOption, T value) {
        if (childOption == null) {
            throw new NullPointerException("childOption");
        }
        if (value == null) {
            synchronized (childOptions) {
                childOptions.remove(childOption);
            }
        } else {
            synchronized (childOptions) {
                childOptions.put(childOption, value);
            }
        }
        return this;
    }
 ```
 
## bind
 
```
     public ChannelFuture bind() {
        validate();
        SocketAddress localAddress = this.localAddress;
        if (localAddress == null) {
            throw new IllegalStateException("localAddress not set");
        }
        return doBind(localAddress);
    }

    public ChannelFuture bind(int inetPort) {
        return bind(new InetSocketAddress(inetPort));
    }

    public ChannelFuture bind(String inetHost, int inetPort) {
        return bind(new InetSocketAddress(inetHost, inetPort));
    }

    public ChannelFuture bind(InetAddress inetHost, int inetPort) {
        return bind(new InetSocketAddress(inetHost, inetPort));
    }

    public ChannelFuture bind(SocketAddress localAddress) {
        validate();
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        return doBind(localAddress);
    }
```

doBind() 方法负责具体的绑定服务端端口，监听请求操作
  
```
   private ChannelFuture doBind(final SocketAddress localAddress) {
        //初始化ServerSocketChannel
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.executor = channel.eventLoop();
                    }
                    doBind0(regFuture, channel, localAddress, promise);
                }
            });
            return promise;
        }
    }
```

#### initChannel

```
    final ChannelFuture initAndRegister() {
        final Channel channel = channelFactory().newChannel();
        try {
            init(channel);
        } catch (Throwable t) {
            channel.unsafe().closeForcibly();
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = group().register(channel);
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

```
    @Override
    public T newChannel() {
        try {
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }
```

#### init

```
    void init(Channel channel) throws Exception {
        //设置socket参数
        final Map<ChannelOption<?>, Object> options = options();
        synchronized (options) {
            channel.config().setOptions(options);
        }

        //设置socket附加属性
        final Map<AttributeKey<?>, Object> attrs = attrs();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }

        ChannelPipeline p = channel.pipeline();

        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }

        //添加handler
        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(Channel ch) throws Exception {
                ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }
                pipeline.addLast(new ServerBootstrapAcceptor(
                        currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
```

#### register

```
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            ...
            AbstractChannel.this.eventLoop = eventLoop;

            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new OneTimeTask() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    ...
                }
            }
        }

```

```
        private void register0(ChannelPromise promise) {
            try {
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;
                doRegister();
                neverRegistered = false;
                registered = true;
                safeSetSuccess(promise);
                //触发通道注册事件
                pipeline.fireChannelRegistered();
                if (firstRegistration && isActive()) {
                    //出发通道就绪事件
                    //如果是服务端，表示启动监听
                    //如果是客户端，表示连接完成
                    pipeline.fireChannelActive();
                }
            } catch (Throwable t) {
                ...
            }
        }

```

```
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
                selectionKey = javaChannel().register(eventLoop().selector, 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    throw e;
                }
            }
        }
    }
```

在触发通道就绪事件后，根据配置决定是否自动出发channel的读操作（默认触发）

```
    public ChannelPipeline fireChannelActive() {
        head.fireChannelActive();

        if (channel.config().isAutoRead()) {
            channel.read();
        }

        return this;
    }
```

```
//DefaultChannelPipeline.java

        public void read(ChannelHandlerContext ctx) {
            unsafe.beginRead();
        }
```

```
//AbstractChannel.java

        public final void beginRead() {
            if (!isActive()) {
                return;
            }

            try {
                doBeginRead();
            } catch (final Exception e) {
                invokeLater(new OneTimeTask() {
                    @Override
                    public void run() {
                        pipeline.fireExceptionCaught(e);
                    }
                });
                close(voidPromise());
            }
        }
```

```
AbstractNioChannel.java

    protected void doBeginRead() throws Exception {
        if (inputShutdown) {
            return;
        }

        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }

        readPending = true;

        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```

将selectionkey注册为channel初始化时设置的事件  
对于NioServerSocketChannel，注册时所设置的事件为OP_ACCEPT

```
    public NioServerSocketChannel(ServerSocketChannel channel) {
        super(null, channel, SelectionKey.OP_ACCEPT);
        config = new NioServerSocketChannelConfig(this, javaChannel().socket());
    }
```

