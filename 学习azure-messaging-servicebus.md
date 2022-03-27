# azure-messaging-servicebus

## SynchronousReceiveWork


当synchronousReceiveWork 到达timeout 时候后（sync的work就要考虑timeout），那么就会在新的线程上自动结束当前工作。

```Java
this.timeoutSubscriptions.add(
    Mono.delay(timeout).subscribe(
        index -> complete("Timeout elapsed for work."),
        error -> complete("Error occurred while waiting for timeout.", error)));
```
- `Mono.delay()` 在 timeout 的时候会发送一个`onNext()`信号（该信号会跑在新的默认线程上），由于mono，也会继续调用到complete。当时间到达 timeout 时候，就会走到 `index -> complete()` 调用内部 `this.complete` 函数结束这个work，这个工作会跑在新的线程上不会影响当前工作进行。目的就是给当前work 增加一个最大工作时间，如果到达了后，这个工作就自动完成。
 
当两个message 时间间隔大于 `TIMEOUT_BETWEEN_MESSAGES`，同样也会在新的线程上结束当前的工作。
   
```Java
this.timeoutSubscriptions.add(
    Flux.switchOnNext(downstreamEmitter.asFlux().map(messageContext -> Mono.delay(TIMEOUT_BETWEEN_MESSAGES)))
        .subscribe(delayElapsed -> {
            complete("Timeout between the messages occurred. Completing the work.");
        }, error -> {
            complete("Error occurred while waiting for timeout between messages.", error);
        }));
```
- `Flux.switchOnNext(mergedpublisher)` downstreamEmitter.asFlux() 作为 Publisher, map 返回一个一系列的 Mono.delay 的 publisher。如果下一个 Mono.delay 在前 一个 Mono.delay 对象 onNext之前生成，那么就会取代前面的 Mono.delay。 这样如果两个message 之间的时间超过Mono.delay， 那么才会直接运行到 subscribe 的方法 delayElapsed 中，从而执行complete。 否则，依然继续收到下一个消息，生成一个新的 Mono.delay, 直到最后一个delay 时间到达，执行 subscribe 的 onNext 方法，`调用complete` 。