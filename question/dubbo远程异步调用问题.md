# dubbo服务异步调用
问题描述：
	在调度系统中配置了商家结算的服务，结算服务里面又调用了计费系统的实时计费服务。但有时调用计费模块返回的结果为NULL。查询日志，调用时传参都没有问题，业务逻辑也没有问题，为什么还返回NULL？跟踪代码，查看dubbo源码，如果调用的服务为异步远程调用，有一个dubbo调用上下文的概念

查看dubbo源码的DubboInvoker类的doInvoke方法

    @Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);
        
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY,Constants.DEFAULT_TIMEOUT);
            if (isOneway) {
            	boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
            	ResponseFuture future = currentClient.request(inv, timeout) ;
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
            	RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }


从上面代码中可以看出，如果参数isAsync为true,会进入else if分支，判断为异步调用，则返回一个新建的RpcResult对象，所以业务dubbo接口的返回值为null。

解决办法：
在调用的方法的里面，先设置一下RpcContext，将异步标志设置为false即可。
RpcContext.getContext().setAttachment(com.alibaba.dubbo.common.Constants.ASYNC_KEY, "false");