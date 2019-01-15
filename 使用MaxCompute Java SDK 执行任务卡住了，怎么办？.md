# 使用MaxCompute Java SDK 执行任务卡住了，怎么办？

<h4>场景一</h4>

用户A

A: “亲，用 MaxCompute Java SDK 跑作业，为什么卡住不动了？”</br>
me: “有 Logview 吗？发来看下”</br>
A: “没有，我用的是SDK，没Logview”</br>

<h4>场景二</h4>

用户B

B ：“亲，用 MaxCompute Java SDK 访问 Table，为什么卡住半天没反应？”</br>
me：“卡在哪一行了？”</br>
B："就 RestClient retry 然后卡住了"</br>

<h4>去繁就简</h4>

用户 A 的问题在于没有 instance 的 logview，导致无法追踪 instance 的运行过程。</br>
通常用户在创建 instance 后会调用 instance.waitForSuccess() 来等待作业运行完成，一旦作业耗时巨大，程序就卡在这一步了，此时如果有 logview ，就能查看追踪查看作业等待的具体原因了。
用户 B 的问题在于 sdk 的 Restclient 本身有重试机制，从表面来看就是卡住了，没有任何输出。</br>
如果在每次重试的时候都输出错误，就可以快速定位问题节约时间了。我已经遇到好几个公共云用户因为缺包导致一直卡住几分钟才丢出异常，严重影响了工作效率。

那么问题可以归结为下面两点：

1【 怎么使用 MaxCompute Java SDK 生成 instance Logview 】</br>
答案很简单， MaxCompute Java SDK 提供了 logview 接口，详情可查看 SDK Java Doc</br>

```js
String logview = odps.logview().generateLogView(instance, 7 * 24);
```

两个参数： instance 对象，logview token 超时时间 (单位：小时)</br>
再次提醒用户，在使用 SDK 的时候，请为每个 instance 记录 Logview，一旦遇到问题可快速追踪。</br>
当然如果改代码很麻烦，那还有一个绝招。在 MaxCompute Console 中使用 wait <instance_id> 命令也可以得到Logview。</br>

2【 能不能在每次重试的时候，都把错误输出呢？】</br>
当然可以。MaxCompute Java SDK 提供了抽象类 RetryLogger 详情可查看 SDK Java Doc

```js
public static abstract class RetryLogger {

    /**
     * 当 RestClent 发生重试前的回调函数
     *
     * @param e
     *     错误异常
     * @param retryCount
     *     重试计数
     * @param retrySleepTime
     *     下次需要的重试时间
     */
    public abstract void onRetryLog(Throwable e, long retryCount, long retrySleepTime);
  }
```

用户只需实现一个自己的 RetryLogger 子类，然后在初始化 odps 对象的时候使用 odps.getRestClient().setRetryLogger(new UserRetryLogger()); 就可以将日志输出。
一个典型的实现如下：

```js
// init odps
odps.getRestClient().setRetryLogger(new UserRetryLogger());

// your retry logger
public class UserRetryLogger extends RetryLogger {

    @Override
    public void onRetryLog(Throwable e, long retryCount, long sleepTime) {
      if (e != null && e instanceof OdpsException) {
        String requestId = ((OdpsException) e).getRequestId();
        if (requestId != null) {
          System.err.println(String.format(
              "Warning: ODPS request failed, requestID:%s, retryCount:%d, will retry in %d seconds.",
              requestId, retryCount, sleepTime));
          return;
        }
      }
      System.err.println(String.format(
          "Warning: ODPS request failed:%s, retryCount:%d, will retry in %d seconds.", e.getMessage(),retryCount,
          sleepTime));
    }
  }
```

掌握上面两种技巧，就可以快速定位问题。
