# Java代码开发规范

### 目的

- 本规范的目的是使本公司开发团队能以标准的、规范的方式设计和编码
- 通过建立编码规范,以使每个开发人员养成良好的编码风格和习惯;
- 并以此形成开发小组编码约定,提高程序的可靠性、可读性、可修改性、可维护性和一致性等
- 增进团队间的交流,保证软件产品的质量。

#### 命名相关
| 规范名称 | 规范内容 |
|--------|--------|
| 项目工程名		  | 项目工程命名以业务为前缀，front、skeleton、server、oms等作后缀
| 模型类命名		 | 持久层相关用domain，web层用VO，相关的package名为domain、vo|
| 工具类命名		 | 业务相关工具类命名叫Tool，包名tools；通用工具类放在framework里面 |
| mvc配置文件	  | SpringMVC的servlet配置文件名称统一叫spring-servlet.xml |
| 统一的命名      | Dto、Dao       |
| 接口的命名		 | 接口全部以I开头，实现类不需要加Impl，譬如ITimeService，TimeService|
| 内部service包名 | 对外暴露的包名命为service，内部service的包名为service.inner |


#### 代码格式
推荐使用Eclipse，且使用统一的Java Formatter配置：[Eclipse-zsy.xml](Eclipse-zsy.xml)
下载之后，通过Eclipse导入Windows—Preferences—Java—Code Style—Formatter—Import

| 规范名称 | 规范内容 |
|--------|--------|
| 代码行宽        | 160个字符      |
| 内部service	   | 不需要对外暴露的service，可以自由选择是否用接口				|

#### 服务间交互规范

| 规范名称 | 规范内容 |
|--------|--------|
| 使用release版本        | 对外发布的jar包版本号要去掉SNAPSHOT      |




