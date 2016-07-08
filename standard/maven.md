# Maven使用规范

### 目的
平时开发中，大部分开发同学经常遇到maven的一些使用问题，排查问题也经常需要花费不少时间；这里给出一些规范，希望大家能遵守规范，尽量避免误用，节省排查时间。

### 关于仓库配置

#### settings.xml配置模板

统一以这个[settings.xml](settings.xml)的配置为准吧，里面配置了snapshot的更新策略为always；
避免SNAPSHOT没更新的问题；

大家如果有什么额外的配置，也可以以这个模板为基础，自己再做一些定制；


#### 去掉pom.xml中的配置
统一使用settings.xml中配置的仓库，避免在工程的pom.xml中配置仓库的地址，譬如我看目前大家很多的pom.xml中都这样配置了，其实没什么必要，统一使用settings.xml中的配置即可；

**建议把pom.xml中的这一段都删掉**

	<repositories>
		<repository>
			<id>home_repo</id>
			<url>http://maven.ihome.com/nexus/content/groups/public/</url>
		</repository>
	</repositories>

### 关于版本号

有一些同学可能还不大清楚maven的构建版本号的演进，这里简单列一下；

一般就是从snapshot再演进到release；

0.0.1-SNAPSHOT--->0.0.1--->0.0.2-SNAPSHOT--->0.0.2--->0.0.3-SHAPSHOT...

目前大家有一些skeleton可能需要发布给其他产品使用，譬如业务平台给消费信托；这种情况推荐都使用release版本；

### pom.xml的配置规范


#### 使用3层的配置

建议每个产品组，都参考业务平台组的3层pom结构；

三层之间的关系很简单

- 第一层为顶级pom.xml
- 第二层的pom.xml的parent为第一层pom.xml
- 第三层的pom.xml的parent为第二层的pom.xml

三层的作用各不相同

- 产品组级别，简称第一层，譬如业务平台组，pom.xml中定义了依赖以及依赖版本、插件；参考业务平台组的[ihome-basicbiz工程](http://git.ihome.com/gitbucket/platform/ihome-basicbiz)
- 模块组级别，简称第二层，譬如消费信托的订单模块，由于第一层pom.xml已经定义了依赖和插件，这一层的pom.xml功能被弱化，仅用于组织下面的子模块，譬如ct-order-server、ct-order-skeleton
- 子模块级别，简称第三层，譬如各个子模块的pom.xml



#### 产品组级别（第一层）

建议每个产品组，定义一个自己的pom.xml，该pom.xml中定义了依赖以及依赖版本、插件，且依赖通过dependencyManagement的方式定义

    <dependencyManagement>
    	<dependencies>
    		<dependency>
    			<groupId>com.ihome.framework</groupId>
    			<artifactId>framework-core</artifactId>
    			<version>${ihome.framework.version}</version>
    		</dependency>
    	</dependencies>
		...
    </dependencyManagement>

##### 什么是dependencyManagement？

- 通常情况下，如果是父子结构的话，我们一般是把所有的公共依赖都配置在父模块中。但是这样就有个问题，某些子模块可能根本就不需要用到那些公共的依赖。
- 定义成dependencyManagement的话，子模块是不会继承父模块中dependencyManagement下配置的那些依赖的。
- 子模块如果需要继承依赖的话，仅需要配置依赖的groupId和artifactId，version和scope继承自父模块。
- 相当于说，父模块配置在dependencyManagement里面的依赖是供子模块继承的，子模块可以选择性地继承。如果是子模块自己需要的那些配置，那就需要配置版本和scope。

##### dependencyManagement的好处

- 这样做，似乎不会减少pom的配置，但是，这样的话，子模块的依赖就会很清晰，而且版本号完全由父pom.xml中去控制，这样不同服务的依赖版本可以到达统一

##### 依赖版本号的指定
建议通过properties的方式指定各种依赖的版本，这样子模块可以通过指定具体的版本号来是实现覆盖；譬如产品级别里面通常会依赖framework；

	<properties>
		<ihome.framework.version>0.0.4</ihome.framework.version>
	</properties>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>com.ihome.framework</groupId>
				<artifactId>framework-core</artifactId>
				<version>${ihome.framework.version}</version>
			</dependency>
		</dependencies>
	</dependencyManagement>

这样的好处是，这个pom的版本可以比较固定；依赖需要特殊版本的话，通过在第二层或者第三层的pom.xml去覆盖配置；

#### 模块级别（第二层）

仅仅需要继承自产品级别，主要的用途是组织一下子模块了~

	<modules>
		<module>account-skeleton</module>
		<module>account-server</module>
		<module>account-tester</module>
	</modules>

如果某些依赖的版本需要跟产品级别定义的不一样的话，可以覆盖一下属性；譬如

	<properties>
		<ihome.framework.version>0.0.3</ihome.framework.version>
	</properties>

#### 子模块级别（第三层）

子模块级别，只有两种配置了

- 继承的依赖，不需要配置具体的版本号
- 子模块自己需要的依赖，需要配置版本号

配置上具体的依赖，子模块的依赖会更清晰一些

#### 实例

以基础平台为例，第一级别为整个基础平台组的，譬如

	<groupId>com.ihome.platform</groupId>
	<artifactId>platform-pom</artifactId>
	<version>0.0.1</version>
	...
	<!-- 定义各种插件，依赖等... -->

第二层级别，为基础平台的某个项目，譬如调度系统

	<!-- parent为基础平台的第一层pom.xml -->
	<parent>
		<groupId>com.ihome.platform</groupId>
		<artifactId>platform-pom</artifactId>
		<version>0.0.1</version>
	</parent>
	
	<!-- 第2层因为不需要定义依赖，所以pom.xml很空，版本定义为SNAPSHOT也没关系；
		 正式发布上线的时候，可以发布release版本；
	-->
	<groupId>com.ihome.platform</groupId>
	<artifactId>ihome-schedule</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	
	<!-- 通过properties来控制某些经常升级的依赖，譬如framework -->
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<ihome.framework.version>0.0.5</ihome.framework.version>
	</properties>
	
	<modules>
		<module>schedule-skeleton</module>
		<module>schedule-server</module>
		<module>schedule-web</module>
		<module>schedule-req-client</module>
	</modules>
	...

第三层，调度系统的某个模块

	<parent>
		<groupId>com.ihome.platform</groupId>
		<artifactId>ihome-schedule</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>
	
	<artifactId>schedule-skeleton</artifactId>
	<!-- 子模块如果是skeleton的话，版本可以自己管理 -->
	<version>0.0.2</version>
	<!-- 
		发布skeleton的时候，第一次需要将parent发布一下
		但skeleton的基本没有什么dependence，所以接下来发布新版的skeleton的时候基本都不用再去发布它的parent了；所以可以在skeleton这个模块第一次新建出来的时候，将它的parent发布一次；
	-->

### 其他的一些建议

#### 同个模块下的子模块互相依赖

譬如ihome-framework下有几个模块

- framework-business
- framework-core：依赖framework-tools
- framework-tools
- framework-web：依赖framework-tools和framework-core

像这种同1个模块下子模块相互依赖的，也可以将各个子模块的信息配置在parent的pom.xml中，用1个${project.version}来指定跟项目同1个版本

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>com.ihome.framework</groupId>
				<artifactId>framework-core</artifactId>
				<version>${project.version}</version>
			</dependency>
			...
		</dependencies>
	</dependencyManagement>
这样framework-web中引用framework-core的时候也可以不需要指定版本

当然这个大家看大家具体的需求了；
