# dubbo服务调用重试问题 #
dubbo服务调用，一般会设置超时时间，dubbo默认是1秒，而在我们的项目中不同的系统设置的超时时间不一样，比如商户后台系统设置为10秒。

    <dubbo:provider retries="0"  timeout="10000"/>

今天遇到的问题是，第三方调用我们的商户门户来完善注册信息，但第三方没有收到我们的后端回调通知。所以根据线索先在日志中查询商户号（“2000438588”）
查询的结果如下：

    2016-12-22 19:13:27,475|thread-197|INFO|SubMerchantInfoImprove.improveMerchantInfo|{"merchantNo":"2000438588"}|begin  ------
    2016-12-22 19:13:37,481|thread-196|INFO|SubMerchantInfoImprove.improveMerchantInfo|{"merchantNo":"2000438588"}|begin  ------
    2016-12-22 19:13:47,508|thread-197|INFO|SubMerchantInfoImprove.improveMerchantInfo|{"merchantNo":"2000438588"}|begin  ------

上述日志去掉了部分非关键信息，从日志上可以看出同一个dubbo服务被调用的时间相差刚好10秒，而我们的服务提供者设置的超时时间刚好是10秒，而为什么同一个服务会调用三次了，因为在我们的服务消费方配置了重试机制，如下：
   
    <dubbo:consumer retries="3" check="false" />
这里的retries=3应该是不包括第一次的，也就是说第一次调用失败后，会重试三次。

这里商户门户调用商户后台接口，商户后台的dubbo服务10秒没有处理完，这时会返回给商户门户超时，而服务消费方设置的重试3次，所以出现了上面的服务被多次调用的情况，而服务多次调用的过程中修改了部分数据，导致最后一次调用的时候，校验数据已经被修改，返回异常到调用方，最终导致第三方收不到回调通知。

