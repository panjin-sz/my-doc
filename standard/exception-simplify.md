## 异常处理规范(简版)

#### 关于异常处理

异常处理是一门艺术；

异常处理，其实没有标准答案，只是不同的哲学理念下的不同选择而已；

#### 一些词汇
服务，代表1个实际在服务器上运行着的进程
这里简单地把服务分为面向用户的服务和非面向用户的服务

- 面向用户的服务：譬如web页面，或者APP的后台，或者这边习惯的称为前置的服务；
- 非面向用户的服务：譬如后台处理程序，定时任务处理程序之类的服务；

下面会针对这两种服务，简单地定义一些异常处理的规范；

#### 面向用户的服务

##### 需要提示用户的异常

当一个系统错误出现时，系统本身需要反馈给用户一种可理解的业务相近的信息，从而用户可以根据这些信息去尽可能解决问题；譬如登陆出错提醒用户：用户名为空，密码错误，验证码出错；如果是操作数据库出错，那么可以提示用户登陆失败等；

实现方式：

- 定义自定义的异常类
- 自定义异常需带上需要通知给用户的错误代码和信息，由异常拦截器统一处理
- controller上提前检测异常情况并返回错误信息(譬如参数不合法，这种可以不抛异常)，或者抛出自定义异常
- service捕获一些异常并封装成用户自定义异常并往上抛，由异常拦截器统一处理

##### 不需要提示用户的异常：

有一类错误属于系统内部运行异常或错误，用户对此类错误根本无能为力。而这类异常同样需要提供足够详细的信息，系统管理员可根据这类异常尽可能解决问题；

异常处理：

- 运行时异常：譬如空指针等，不需要刻意捕获，由异常拦截器统一处理
- 声明式异常：
	- 如果程序能在catch块处理异常并恢复，则捕获异常并处理；
	- 否则声明throws由上层处理，或者封装成运行时异常往上层抛出（需要包含原始异常）

#### 非面向用户的程序

非面向用户的程序，只要保持较好的异常处理规则即可；

#### 一些异常处理准则

仅仅是一些处理的规则

##### 异常处理不应该做的事情
###### 不应该吞掉异常
吞掉异常是万恶的行为，譬如下面的代码

	public void eatException() {
        try {
            // ..some code that throws SQLException
            throw new SQLException("connection fail");
        } catch (SQLException ex) {
            ex.printStackTrace(); 
            logger.error(ex.getMessage()); 
            throw new RuntimeException(ex.getMessage(), ex.getCause());
        }
    }

上面catch块里面的3种处理方式，是典型的吞异常的例子：

- 第1种是直接把异常打印到终端，但到了生产环境有可能并不会截获终端的日志
- 第2种是只记录了一下异常的信息，没有把堆栈记录下来
- 第3种是把异常抛出了，但是ex.getCause()会导致把异常堆栈里面最顶的那层


###### 不应该把检查异常持续往上层抛出
这样每个层次都得处理该异常，譬如传统MVC程序下的SQLException

我们经常将代码分Controller-->Service-->Dao 的层次结构，DAO层中会包含抛出异常的方法

    public Customer retrieveCustomerById(Long id) throws SQLException {
     	//根据 ID 查询数据库
    }

从设计耦合角度仔细考虑一下，这里的 SQLException 污染到了上层调用代码，调用层需要显式的利用 try-catch 捕捉，或者向更上层次进一步抛出；

通常Service层也只是throws声明一下，这样最终就会由Controller那个层次来处理，而Controller层次一般都是业务流程代码，看到SQLException会茫然不知所措的；

其他的层次调用的场景同理；

###### 不应该重复记录异常
捕获了异常，如果还要重新抛出则不记录异常的详细堆栈，由上层做记录，譬如下面代码

    public void duplicateException() {
        try {
            // ..some code that throws SQLException
            throw new SQLException("connection fail");
        } catch (SQLException ex) {
            logger.error(ex.getMessage(), ex);
            throw new RuntimeException(ex.getMessage(), ex);
        }
    }

cache块里面的2行代码，第1行记录了完整堆栈，然后又把整个堆栈包装再重新抛出，假如调用方也记录异常的话，最终日志里面会看到许多的重复异常，不利于排查问题；

比较恰当的处理方式

- 如果不需要重新抛出，则要记录完整的异常堆栈
- 如果还需要重新抛出，则不记录，或者简单记录一下异常信息，业务异常信息

	logger.error("xxx异常:{}", ex.getMessage());

###### 不应该捕获顶层的异常
因为有可能会把一些运行时异常也捕获了，看例子

    public void copy(File file){
        try{	
            //…抛出 IOException 的代码调用
        }catch(Exception e){
			logger.error("fail", e);
        }
		// 继续业务处理
    }

这种代码在比较长的方法里面比较常见；

这里主要问题是有可能那个cache块抛出的是运行时异常（数组溢出，下标溢出之类），那么也被cache(Exception)捕获。通常出现运行时异常的话，是不会继续做下面的业务处理的，但是这个情况，下面的业务代码还是会继续执行，运气好下面的业务代码也抛出空指针之类的异常，那么问题还能被发现，运气不好下面的业务处理代码不抛出异常，有可能最终的结果会比较诡异难发现；


###### 不应该过多的异常包装
异常封装过多的话，会严重影响性能

需要记住的一件事是异常代价高昂，同时让你的代码运行缓慢。


##### 异常处理应该做的事情
###### 应该尽量使用JDK标准的异常
如果能复用JDK自带异常的，则尽量不要自定义异常，减少开发和运维人员的压力。

###### 应该抛出尽量具体的异常信息
譬如一个FileNotFoundException异常或比一个IOException节省了很多的排查时间；
	
###### 应该尽量提早抛出异常
譬如一些参数肯定不允许为空的场景，可以程序先做判断或者抛出自己的异常，如果放之不管最终抛出空指针，会增加排查难度

    public void login(String username, String password) {
        if(username == null || password == null) {
             throw new BusinessException("...");
        }
        if(password.equals(oldPassword)) {
            // ..
        }
    }

###### 应该尽量将运行时异常文档化
将可能抛出的运行时异常，通过@throws表明；

    /**
     * 用户登录
     * @param username 用户名
     * @param password 密码
     * @throws BusinessException
     */
    public void login(String username, String password) {
        if(username == null || password == null) {
             throw new BusinessException("...");
        }
    }


