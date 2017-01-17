dubbo泛化

假设有系统A、系统B。。。，系统之间Rpc调用使用dubbo服务实现，系统A依赖系统B的某个dubbo服务。现在系统A使用了泛化调用dubbo服务的功能，那么在系统A第一次泛化调用的时候，会创建一个缓存对象，缓存了zk节点中所有的服务提供方连接。假如系统B没有启动，那么系统B中的服务提供者在zk上面是没有地址信息的，这样A系统中的缓存对象不会保存这个服务的地址信息，泛化调用这个接口的话，这时候找不到服务提供者，会抛出IllegalStateException异常。究其原因是这个缓存对象很重，如果缓存对象已经存在的话，假如后面zk上有新的服务提供者加入的话，缓存对象是不知道的，也就是说dubbo没有维护这个缓存，那么出现这种情况的话，现在的做法是抛出这个异常的时候，清空一下这个缓存对象，下次请求的时候就会重新创建这个缓存对象。

下面代码从dubbo开发指南上摘录：

    import com.alibaba.dubbo.rpc.service.GenericService; 
    ... 
     
    // 引用远程服务 
    ReferenceConfig<GenericService> reference = new ReferenceConfig<GenericService>(); // 该实例很重量，里面封装了所有与注册中心及服务提供方连接，请缓存
    reference.setInterface("com.xxx.XxxService"); // 弱类型接口名 
    reference.setVersion("1.0.0"); 
    reference.setGeneric(true); // 声明为泛化接口 
     
    GenericService genericService = reference.get(); // 用com.alibaba.dubbo.rpc.service.GenericService可以替代所有接口引用 
     
    // 基本类型以及Date,List,Map等不需要转换，直接调用 
    Object result = genericService.$invoke("sayHello", new String[] {"java.lang.String"}, new Object[] {"world"}); 
     
    // 用Map表示POJO参数，如果返回值为POJO也将自动转成Map 
    Map<String, Object> person = new HashMap<String, Object>(); 
    person.put("name", "xxx"); 
    person.put("password", "yyy"); 
    Object result = genericService.$invoke("findPerson", new String[]{"com.xxx.Person"}, new Object[]{person}); // 如果返回POJO将自动转成Map 
     
    ...

我们代码中实现的泛化代码

    ReferenceConfig<GenericService> reference = new ReferenceConfig<GenericService>();
        reference.setApplication(dubboAppCfg);
        reference.setRegistry(dubboRegistry);
        reference.setInterface(interFaceName);// 弱类型接口名
        reference.setVersion(versionId);
        reference.setGroup(dubboGroup);
        // 声明为泛化接口
        reference.setGeneric(true);
        // 缓存
        ReferenceConfigCache cache = ReferenceConfigCache.getCache();
        GenericService genericService = cache.get(reference);
        // 基本类型以及Date,List,Map等不需要转换，直接调用
        try {
            Object obj = genericService.$invoke(methodName, parameterTypes, args);
            return obj;
        } catch (GenericException e) {
            // 必须清除缓存，否则下次会获取到空指针
            logger.error(Log.op("geneclient call").msg("泛化接口").kv("interFaceName", interFaceName).kv("methodName", methodName).kv("versionId", versionId)
                    .kv("dubboGroup", dubboGroup).kv("parameterTypes", parameterTypes==null?"":parameterTypes[0]).kv("args", args==null?"":args[0]).toString(), e);
            cache.destroy(reference);
            // 必须清除缓存，否则下次会获取到空指针
            OpenApIException appexcetpion = getExceptionInfo(e);
            throw appexcetpion;
        }

从我们的实现代码中可以看出只有在抛出GenericException异常时才清除缓存。