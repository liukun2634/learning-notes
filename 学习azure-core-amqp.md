# 学习 azure-core-amqp 项目

- [学习 azure-core-amqp 项目](#学习-azure-core-amqp-项目)
  - [ReactorConnection](#reactorconnection)
    - [Constructor](#constructor)
    - [getOrCreateConnection](#getorcreateconnection)
    - [createSession](#createsession)
    - [createRequestResponseChannel](#createrequestresponsechannel)
  - [AmqpChannelProcessor](#amqpchannelprocessor)
    - [Constructor](#constructor-1)
    - [subscribe](#subscribe)

## ReactorConnection


### Constructor

**基本初始化**

```Java
this.connectionOptions = connectionOptions;
this.reactorProvider = reactorProvider;
this.connectionId = connectionId;
this.handlerProvider = handlerProvider;
this.tokenManagerProvider = Objects.requireNonNull(tokenManagerProvider,
    "'tokenManagerProvider' cannot be null.");
this.messageSerializer = messageSerializer;
this.handler = handlerProvider.createConnectionHandler(connectionId, connectionOptions);

this.retryPolicy = RetryUtil.getRetryPolicy(connectionOptions.getRetry());
this.operationTimeout = connectionOptions.getRetry().getTryTimeout();
this.senderSettleMode = senderSettleMode;
this.receiverSettleMode = receiverSettleMode;
```

**connectionMono**
`this.connectionMono` 是会返回一个 ACTIVE `Connection` 的 `Mono<Connection>`，也就是说一个`ReactorConnect` 对象里会维护一个 proton-j 的 `Connection`。

```Java
this.connectionMono = Mono.fromCallable(this::getOrCreateConnection)
    .flatMap(reactorConnection -> {
        final Mono<AmqpEndpointState> activeEndpoint = getEndpointStates()
            .filter(state -> state == AmqpEndpointState.ACTIVE)
            .next()
            .timeout(operationTimeout, Mono.error(new AmqpException(true, String.format(
                "Connection '%s' not opened within AmqpRetryOptions.tryTimeout(): %s", connectionId,
                operationTimeout), handler.getErrorContext())));
        return activeEndpoint.thenReturn(reactorConnection);
    })
    .doOnError(error -> {
        final String message = String.format(
            "connectionId[%s] Error occurred while connection was starting. Error: %s", connectionId, error);

        if (isDisposed.getAndSet(true)) {
            logger.verbose("connectionId[{}] was already disposed. {}", connectionId, message);
        } else {
            closeAsync(new AmqpShutdownSignal(false, false, message)).subscribe();
        }
    });
```
- `fromCallable()` 调用 `getOrCreateConnection()` 创建一个 `Connection`
- `flatMap()` block 这个 `Connection` 到 ACTIVE 时候才返回
  - `Mono<AmqpEndpointState>` 先创建一个Mono，里面存的是State 
    - `filter()` 只当state 是 ACTIVE 时候再创建
    - `next()` 接收第一个element
    - `timeout()` 限制了获取第一个element，也就是第一个 ACTIVE status 返回时候的 timeout 时间
  - `thenReturn()` 会一直 block 直到有 ACTIVE 状态返回时候，再把这个`Connection` 返回
- `doOnError()` 当出现error 时候, 会根据 `isDisposed` 是否为 true，向 `closeAsync()` 传入 `AmqpShutdownSignal` 异步清理相应的close 步骤 。也因为是异步close，所以需要额外的变量 `isDisposed` 用于记录是否完成。（Reactor 中的异步结果，通过额外atomic flag判断。）

**endpointStates**

`this.endpointStates` 表示当前 `Connection` 的状态, 由于是`Flux<AmqpEndpointState>`, 可以向其注册的 `subscriber` 发送 connection endpointStates变化。


```Java
this.endpointStates = this.handler.getEndpointStates()
            .takeUntilOther(shutdownSignalSink.asMono())
            .map(state -> {
                logger.verbose("connectionId[{}]: State {}", connectionId, state);
                return AmqpEndpointStateUtil.getConnectionState(state);
            })
            .onErrorResume(error -> {
                if (!isDisposed.getAndSet(true)) {
                    logger.verbose("connectionId[{}]: Disposing of active sessions due to error.", connectionId);
                    return closeAsync(new AmqpShutdownSignal(false, false,
                        error.getMessage())).then(Mono.error(error));
                } else {
                    return Mono.error(error);
                }
            })
            .doOnComplete(() -> {
                if (!isDisposed.getAndSet(true)) {
                    logger.verbose("connectionId[{}]: Disposing of active sessions due to connection close.",
                        connectionId);

                    closeAsync(new AmqpShutdownSignal(false, false,
                        "Connection handler closed.")).subscribe();
                }
            })
            .cache(1);
```
- **`this.handler.getEndpointStates()` 才是真正的state数据的源头**，`handler` 通过可控制 `Sinks<EndpointState>`变量，在相应状态变化时，发出State后，再通过后面 `map()` 操作转换为 `AmqpEndpointState` 发送给注册在 `this.endpointStates` 上的`subscriber`.
- `takeUntilOther()` 是当有 `shutdownSignalSink` 发出的时候，就停止向其 `subscriber` 发生state 变化了。
- `map()` 用于把 `EndpointState` 转换成 `AmqpEndpointState`
- `onErrorResume()` 处理error 时候`closeAsync()`, 可以获取到相应error 接着发送出去 `Mono.error`。（`doOnError` 与 `onErrorResume` 区别就是要不要对error 进行操作。） 
- `doOnComplete()` 当Complete时候 也需要 `closeAsync()`。
- `cache(1)` 保存 history 最后的一个state，当有新的`subscriber`，会将最后一个state 发送出去。

**subscriptions**

`this.subscriptions` 是 `endpointStates` 和 `session executor` 的 `Disposable` 的集合，这样可以一次 `dispose` 所有的 `subscribe`。
```Java
this.subscriptions = Disposables.composite(this.endpointStates.subscribe());
```
### getOrCreateConnection

函数声明为 `synchronized`, 对于一个 `ReactorConnection` 对象，刚好多个线程同时调用，保证了只会生成一个 `Connection`。 因为如果不存在再去创建，存在则不创建。

**这里是否可以用 null double check 降低锁的开销？**
```Java
private synchronized Connection getOrCreateConnection() throws IOException {
   if (connection == null) {
     //Create Connection
   }
   return connection;
}
```

通过传入的 `reactorProvider` 生成 `reactor`, 分配传入的 `connectionId` 和 设置最大 `frameSize`。然后这个 `reactor` 再去设置 `host`，`protocol` 和 `handler` 并返回设置好的 `Connection` 对象。所以一个 `ReactorConnection` 对象里面只包含一个 `Reactor` 对象和一个 `Connection` 对象。


``` Java
final Reactor reactor = reactorProvider.createReactor(connectionId, handler.getMaxFrameSize());
connection = reactor.connectionToHost(handler.getHostname(), handler.getProtocolPort(), handler);

final ReactorExceptionHandler reactorExceptionHandler = new ReactorExceptionHandler();
```

再定义相应的 `pendingDuration` 并新建一个 single-threaded 的 `scheduler` (线程名字为"reactor-executor") 去启动 `connection`。 这是因为 proton-j 的`Reactor` 不是线程安全的。所以每个`reactor` 只跑在一个线程上可以保证线程安全。

```Java
final Duration timeoutDivided = connectionOptions.getRetry().getTryTimeout().dividedBy(2);
final Duration pendingTasksDuration = ClientConstants.SERVER_BUSY_WAIT_TIME.compareTo(timeoutDivided) < 0
    ? ClientConstants.SERVER_BUSY_WAIT_TIME
    : timeoutDivided;
final Scheduler scheduler = Schedulers.newSingle("reactor-executor");
```
- `timeoutDivided` 是 `retryOptions` 里面配置 timeout 时间的一半。
- `pendingTaskDuration` 就取 `SERVER_BUSY_WAIT_TIME = 4s` 和 `timeoutDivided` 时间的最小者 （最多为4s）。
- `Schedulers.newSingle()` 可以对每一个`reactor`都生成一个线程运行一个`connection`。 这里千万不能使用 `Schedulers.single()` 这是因为`single()`是所有该进程上的都使用这一个线程，从而导致所有connection 都运行在一个线程上，降低了效率。


将这些所需要的参数全部传给 proton-j `ReactorExecutor` 的构造函数构建并返回 `exector`， 然后就可以调用 `start()` 方法运行在 `scheduler` 上，也就是额外的线程上面 （名称为reactor-executor-x）。

```Java
executor = new ReactorExecutor(reactor, scheduler, connectionId,
    reactorExceptionHandler, pendingTasksDuration,
    connectionOptions.getFullyQualifiedNamespace());
```

这里还额外配置了`executor` close 的 `Mono`, 而且还 `defer` 到调用的那一刻才会构建, 并且是`closeAsync()`.

```Java
final Mono<Void> executorCloseMono = Mono.defer(() -> {
    synchronized (this) {
        return executor.closeAsync();
    }
});
```

然后配置当`dispatcher` 收到 `ShutdownSignal` 的信号后，先去关闭`connection`，等 connection 关闭后，然后调用 `exectorCloseMono` 去关闭 `executor`。

```Java
 reactorProvider.getReactorDispatcher().getShutdownSignal()
        .flatMap(signal -> {
            reactorExceptionHandler.onConnectionShutdown(signal);
            return executorCloseMono;
        })
        .onErrorResume(error -> {
            reactorExceptionHandler.onConnectionError(error);
            return executorCloseMono;
        })
        .subscribe();

executor.start();
```

这些都配置完成后，启动 `executor`, 并返回 `connection`。


### createSession

`createSession` 是在创建 `RequestResponseChannel` 时候会被调用，也就是为 `channel` 在 `connection` 上 创建一个可依附的 `session`。该函数最后返回的是一个 ACTIVE 的 `session`。 由于返回是 `Mono<AmqpSession>`，所以是异步订阅方式。

**函数并没有声明为 `synchronize`?**

```jAVA
public Mono<AmqpSession> createSession(String sessionName) {
  return connectionMono.map(connection -> {
      ...
  }).flatMap(sessionSubscription -> {
      ...
  });
```
- `map()` 在当前所在的 `connection` 上面创建一个 `session`， 返回的`SessionSubscription` 的对象。
- `flatMap()` 保证 `session` 创建并为 ACTIVE 时候再返回


在创建 `session` 时， 首先会根据传入的 `sessionName` 判断是否已经存在该 `connection` 的 `sessionMap` 上，只有不存在的时候才会去创建。如果存在，则直接返回已有的sessionSubscription。
```Java
.map(connection -> {
    final SessionSubscription sessionSubscription = sessionMap.computeIfAbsent(sessionName, key -> {
        final SessionHandler sessionHandler = handlerProvider.createSessionHandler(connectionId,
            getFullyQualifiedNamespace(), key, connectionOptions.getRetry().getTryTimeout());
        final Session session = connection.session();

        BaseHandler.setHandler(session, sessionHandler);
        final AmqpSession amqpSession = createSession(key, session, sessionHandler);
        final Disposable subscription = amqpSession.getEndpointStates()
            .subscribe(state -> {
            }, error -> {
                /* If we were already disposing of the connection, the session would be removed. */
                if (isDisposed.get()) {
                    return;
                }

                logger.info("connectionId[{}] sessionName[{}]: Error occurred. Removing and disposing"
                    + " session.", connectionId, sessionName, error);
                removeSession(key);
            }, () -> {
                // If we were already disposing of the connection, the session would be removed.
                if (isDisposed.get()) {
                    return;
                }

                logger.verbose("connectionId[{}] sessionName[{}]: Complete. Removing and disposing session.",
                    connectionId, sessionName);
                removeSession(key);
            });

        return new SessionSubscription(amqpSession, subscription);
    });

    return sessionSubscription;
})
```
- `connection.session()` 会在 proton-j `Connection` 里面维护的 `session` 上增加一个 `Session` 对象, 并返回。
- `createSession(key, session, sessionHandler)` 根据传入的`sessionName`，生成的 proton-j `session` 和 `sessionHandler`，生成`AmqpSession` 的对象。
- `amqpSession.getEndpointStates()` 返回的 `Disposable subscription` 会被记录到 `SessionSubscription` 中，随当前 `AmpqSession` 对象一起返回。


前面处理返回的 `SessionSubscription` 会获取到里面的 `session`对象，一直block到该对象的 `endpointState` 是ACTIVE时候才会返回。
```Java
.flatMap(sessionSubscription -> {
    final Mono<AmqpEndpointState> activeSession = sessionSubscription.getSession().getEndpointStates()
        .filter(state -> state == AmqpEndpointState.ACTIVE)
        .next()
        .timeout(retryPolicy.getRetryOptions().getTryTimeout(), Mono.error(new AmqpException(true,
            String.format("connectionId[%s] sessionName[%s] Timeout waiting for session to be active.",
                connectionId, sessionName), handler.getErrorContext())));

    return activeSession.thenReturn(sessionSubscription.getSession());
});
```
该部分逻辑与创建 Connection 时 block 到 ACTIVE 的逻辑相同。唯一区别是这里增加了一个 `SessionSubscription` 对象来保存 `session` 和对应的 `subscription`, 这是因为 Connection 只有一个，保存在当前对象的私有对象里。而 Session 是有多个的，需要组成一个整体，保存在 `sessionMap` 里。

### createRequestResponseChannel

`createRequestResponseChannel` 构建了client 与 message broker 之间的
双向连接（link）。需要传入参数是 `sessionName` (不存在就构建新的session), `linkName`, 和 `entityPath`(message broker 的地址)。

```Java
protected AmqpChannelProcessor<RequestResponseChannel> createRequestResponseChannel(String sessionName, String linkName, String entityPath) {

    final Flux<RequestResponseChannel> createChannel = createSession(sessionName)
        .cast(ReactorSession.class)
        .map(reactorSession -> new RequestResponseChannel(this, getId(), getFullyQualifiedNamespace(), linkName,
            entityPath, reactorSession.session(), connectionOptions.getRetry(), handlerProvider, reactorProvider,
            messageSerializer, senderSettleMode, receiverSettleMode))
        .doOnNext(e -> {
            logger.info("connectionId[{}] entityPath[{}] linkName[{}] Emitting new response channel.",
                getId(), entityPath, linkName);
        })
        .repeat();

    return createChannel
        .subscribeWith(new AmqpChannelProcessor<>(connectionId, entityPath,
            channel -> channel.getEndpointStates(), retryPolicy,
            new ClientLogger(RequestResponseChannel.class.getName() + ":" + entityPath)));
    }
```
- `createSession()` 根据 `sessionName` 返回一个 `AmqpSession` 对象
- `cast()` 向下转型成 `ReactorSession`
- `map()` 根据该 `reactorSession` 对象，生成一个新的 `RequestResponseChannel` 对象。实现了Flux类型转换，现在返回的就是 `Mono<RequestResponseChannel>`。
- `doOnNext()` 增加 `onNext()` 时候额外的log信息，用于记录channel 已经创建并emit了。
- `repeat()` 使得 `Mono` 对象转换成了 `Flux` 对象。可以在Session上不断的创建新的 Channel。
- `subscribeWith()` 订阅了创建的createChannel，并对应生成一个新的 `AmqpChannelProcessor` （Processor extends Subscriber）对相应生成的channel 增加相应的处理逻辑，并最后把这个 `AmqpChannelProcessor` 返回。 


## AmqpChannelProcessor

`AmqpChannelProcessor` 扩展了 Mono 所以可以被subscribe，然后实现了Proccesor，从而可以作为 channel 的中间处理过程，对生成的channel增加相应的方法。目的是监控channel的创建状态，一旦创建成功，可以同时通知所有的下游订阅。

```Java
public class AmqpChannelProcessor<T> extends Mono<T> implements Processor<T, T>, CoreSubscriber<T>, Disposable
```

### Constructor

`fullyQualifiedNamespace` 和 `entityPath` 记录了 message broker 的地址信息，`endpointStatesFunction` 其实就是获取channel的endpointState，这里选择传入，可以减少重复code。

```Java
public AmqpChannelProcessor(String fullyQualifiedNamespace, String entityPath,
    Function<T, Flux<AmqpEndpointState>> endpointStatesFunction, AmqpRetryPolicy retryPolicy, ClientLogger logger) {
    this.fullyQualifiedNamespace = Objects
        .requireNonNull(fullyQualifiedNamespace, "'fullyQualifiedNamespace' cannot be null.");
    this.entityPath = Objects.requireNonNull(entityPath, "'entityPath' cannot be null.");
    this.endpointStatesFunction = Objects.requireNonNull(endpointStatesFunction,
        "'endpointStates' cannot be null.");
    this.retryPolicy = Objects.requireNonNull(retryPolicy, "'retryPolicy' cannot be null.");
    this.logger = Objects.requireNonNull(logger, "'logger' cannot be null.");
    this.errorContext = new AmqpErrorContext(fullyQualifiedNamespace);
}
```

### subscribe

传入一个 `CoreSubscriber` 的对象 `actual`, 订阅到这个 `AmqpChannelProcessor` 对象上，可以理解actual 就是下游。

```Java
public void subscribe(CoreSubscriber<? super T> actual) {
 ...
}
```

如果`AmqpChannelProcessor` 对象出现了问题，`isDisposed()` 会返回false，这样下游`actual` `emptySubscription` 上，并 emit 一个 error 给 `actual`。

```Java
if (isDisposed()) {
    if (lastError != null) {
        actual.onSubscribe(Operators.emptySubscription());
        actual.onError(lastError);
    } else {
        Operators.error(actual, logger.logExceptionAsError(new IllegalStateException(
            String.format("namespace[%s] entityPath[%s]: Cannot subscribe. Processor is already terminated.",
                fullyQualifiedNamespace, entityPath))));
    }
    return;
}
```

如果`AmqpChannelProcessor` 对象正常， 那么会用 `actual` 和 `this` 去创建一个 `ChannelSubscriber`。`ChannelSubscriber` 是个 private 的内部类，目的是包装每个注册在`ChannelSubscriber` 上的下游`actual`，当有收到新建的channel后，就通知所有的下游。 这里虽然叫 `subscriber`，但其实可以理解是 `actual` 的 `publisher`, 是下游的真正数据源。
```Java
final ChannelSubscriber<T> subscriber = new ChannelSubscriber<T>(actual, this);
actual.onSubscribe(subscriber);
```

注意：这里虽然使用的是new，但是因为传入的是 `this`, 所以维护的是同一个 `processor`，只不过根据 `actual`的多少，相应生成多个 `ChannelSubscriber`。 这里的 `subscriber` 是对 `processor` 而言的, 对 `actual` 可以理解成 `publisher` （或者 `subscription`，因为可以`onSubscriber()`）。

下游订阅好 `subscriber` 后，就去判断 `currentChannel` 是否存在。

如果 `currentChannel` 不为空，说明 `channel` 已经通知过了，需要直接调用`subscriber` 发出 `complete`消息，通知到上面所有的`actual`。

```Java
synchronized (lock) {
    if (currentChannel != null) {
        subscriber.complete(currentChannel);
        return;
    }
}
```
如果 `currentChannel` 为空，就把 `subscriber` 就加入 `Processor` 的 `subscribers` 里面存储起来, 等待 `onNext()` 消息，再通知。这里还额外判断 如果`Processor` 不是在 `retryPending` 过程中，就去 `requestUpstream()`，设置 `subscription.request(1)`。

```Java
subscribers.add(subscriber);
logger.verbose("namespace[{}] entityPath[{}] subscribers[{}]: Added a subscriber.",
    fullyQualifiedNamespace, entityPath, subscribers.size());

if (!isRetryPending.get()) {
    requestUpstream();
}
```

理解一下就是 `channel` 不为空就是晚到了，前面通知过了，额外单独再通知一下。否则，就得等到 `channel` 好了，大家一起被通知。



