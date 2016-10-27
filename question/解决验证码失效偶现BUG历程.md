**问题描述**：

在商户门户系统，登录时输入正确的验证码报“验证码失效”的错误，该问题偶然出现，开始只在测试环境测试，没引起重视，该BUG一直记录在企业管理平台，标记为未解决，今天迭代任务完成，就分析一下這个问题，并记录一下问题的解决思路。问题分析过程中，感谢彭阳大师的支持。

**出现问题的几种场景：**

1. 输入链接，首次登陆时，点击刷新验证码，再输入正确的验证码能正常登录。
1. 退出登录，再重新登录。
1. 断点调试，问题无法重现。

**解决思路：**

1、先看报错语句是在哪里抛出，检查代码发现在登录的时候，代码如下：

    Object session_captcha = getSession().getAttribute(Constant.MERCHANT_CAPTCHA_CODE);
    if (session_captcha == null) {
    	return JsonResultUtil.resultError(PirateMerchantCode.CAPTCHA_FAIL.getCode(),
    		PirateMerchantCode.CAPTCHA_FAIL.getDesc());
    }
也就是说在登录的时候，从session里面获取的验证码为null，而查看代码，在项目中有一个interceptor，在调用preHandle和afterCompletion方法的时候，会设置一个万能验证码（只在测试环境使用，这样不用每次输入固定的验证码即可），并设置到session中。在出现這个问题之前，该万能验证码功能一直正常，说明不是這个引起的。

那为什么這里获取不到验证码，因为已经设置到session中了，這个问题之前同事已经解决过，说断点调试是模拟不出来，一切正常，所以我没有设置断点调试，只是在各个关键节点加一些日志，如登录获取验证码、生成验证码图片、拦截器里覆盖验证码的session，日志记录时间戳、sessionID、验证码的值信息。从日志上查看，sessionID有变化时，这时就有可能获取不到验证码，日志信息如下：

    1477547949492---获取验证码---1111---sessionId=1uzwy1l5sibpo!1477547846149
    1477547949492---移除验证码---null---sessionId=1uzwy1l5sibpo!1477547846149
    
    1477547968273--------------preHandle----------------
    1477547968273---1---null---sessionId=x0uqkv3al9el!1477547968243
    1477547968273---2---null---sessionId=x0uqkv3al9el!1477547968243
    1477547968274---生成验证码---DPQN---sessionId=x0uqkv3al9el!1477547968243
    1477547968280--------------afterCompletion----------------
    1477547968281---1---DPQN---sessionId=x0uqkv3al9el!1477547968243
    1477547968281---2---1111---sessionId=x0uqkv3al9el!1477547968243

从日志中可以看出sessionID有变化，那为什么会变化？分布式session的超时时间是设置成30分钟，不可能是超时。

    /** 会话超时时间，单位为秒，默认1800秒 */
    private int sessionTimeout = 1800;

难道是登录的时候拿到的session和拦截器里面的session不是同一个session，导致重新生成了新的session么？查看代码登录的session是从thread Local里面拿出来的，封装的session服务的service设置为prototype了，那每一个线程的session是独立的，应该和這个也没关系。查看這个session是哪里创建的，查看代码，在HttpRequestInterceptors這个拦截器里面设置的，***找到這个设置session的地方是关键***，代码如下。

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    HttpBaseVo httpBaseVo = new HttpBaseVo();
    httpBaseVo.setRequest(request);
    httpBaseVo.setResponse(response);
    HttpSession session = request.getSession();
    httpbasevo.setsession(session);
    IHttpBaseSerivce httpBaseSerivce  = SpringContext.getBean(IHttpBaseSerivce.class);
    logger.info("httpBaseService对象HashCode:"+httpBaseSerivce.hashCode());
    httpBaseSerivce.setHttpBaseInfo(httpBaseVo);
    Constant.httpThreadLocal.set(httpBaseSerivce);
    return true;
    }

查看request.getSession()這个方法的说明：“Returns the current session associated with this request, or if the request does not have a session, creates one”，意思是如果请求中没有session就创建一个，有session对象存在是不会重新创建的。sessionID变了，说明有重新创建session。那么在创建session這里打印日志看看是那个请求的URL进来创建了session，发现是调用下面两个请求重建了session

    ---url=http://172.16.22.32:8082/pirate-merchant-web/code
    ---url=http://172.16.22.32:8082/pirate-merchant-web/merchant/getCaLicense

这里如果是并发的话，两个请求都以为session为null，则都去创建session，后面那个请求会覆盖前面一个请求的session，故sessionID以最后一个为准，登录的时候，用后面的那个生成的sessionID是获取不到验证码的。

解决办法，前端控制，先调用CaLicense在调用code请求，代码如下:

    // 获取CaLisence
    var getCaLicense = function(cb) {
    $http.post({
    url: 'merchant/getCaLicense',
    success: function(data) {
    if (data.code == $comm.INFOS.XTB_SUCCESS) {
    var license = data.data;
    localStorage.setItem('CaLicense', license);
    }
    typeof cb === 'function' && cb();
    },
    error: function(){
    	typeof cb === 'function' && cb();
    }
    });
    };
    $core.Ready(function() {
    sessionStorage.clear();
    getCaLicense(function(){
    	getCaptcha();
    });
    $('#_backToLogin').remove();
    });

再重新测试，问题解决。**