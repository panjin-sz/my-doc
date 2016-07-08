# 应用版本、zip包版本及灰度发布规范

## 一、目标 

* 定义app与zip包的版本规范，方便管理
* 服务端可通过版本标准进行分流，实现灰度发布等功能 

## 二、版本命名规范

### app版本

app的版本号分为以下几个部分：

* 第一部分为主版本号
* 第二部分为次版本号
* 第三部分为修订版本号
* 第四部分为日期版本号加希腊字母版本号
	* 希腊字母版本号3种，test（只在内部系统出现） gray release
	
如：
	0.1.3.20150813_gray
	
* 0 主版号
* 1 次版本号
* 3 修订版本号
* 20150813_gray 日期版本号加希腊字母
	
### html5 zip包版本

zip包版本号分三部分：

* 第一部分为日期版本号
* 第二部分为字母版本号
* 第三部分为是否强制更新
	* force 必须强制更新
	* provide 非强制更新

如： 20151001.release.force

zip包更新规则

 * 如果最新版本定义为force， 则必须强制更新
 * 如果本地版本与最新版中间的任何版本有force， 也必须强制更新 
	
## 三、路由规则

* 如果是release版本， 全部路由到线上
* 如果是gray版本
	* 如果系统的最新版本号（可配置在NLB）比当前app版号大，路由正式环境
	* 如果两个版本一样，路由到灰度环境



## 四、app大版本更新的灰度测试

![img_gray_flow](img/gray-flow.png)



## 五、只更新html5 zip包的灰度测试

![img_gray_flowhtml5](img/gray-flow-html5.png)

## 六、发送请求规范 -- 待讨论

为了方便app分流， 需要所有的客户端app请求和cookie里都设上 app-version字段。

* head   app-version: 0.1.3.20150813_gray
* cookie www.wjrrc.cc域下  app-version: 0.1.3.20150813_gray

所有下载的HTML5的请求head上都设 app-{package}-version字段：

* app-wm-version 20150815.release.force





