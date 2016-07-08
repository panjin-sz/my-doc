# logback.xml模板详解

### 目的
对目前的logback.xml的配置做出一些解释，配置模板见[logback.xml](logback.xml "logback.xml")

### 日志切割

目前配置文件中，配置了按照时间和大小来切割日志文件：

	<rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
		<FileNamePattern>${log.base}/${log.proj}_%d{yyyy-MM-dd}.%i.log</FileNamePattern>
		<maxHistory>30</maxHistory>
		<timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
			<maxFileSize>100MB</maxFileSize>
		</timeBasedFileNamingAndTriggeringPolicy>
	</rollingPolicy>

对于上述配置来说，默认是保留30天的日志文件，每天的日志文件可能会有多个，但每个的大小都不超过100MB；日志文件列表看起来会像这样：

	ihome-framework-usage.log
	ihome-framework-usage_2015-11-17-0.log
	ihome-framework-usage_2015-11-17-1.log
	ihome-framework-usage_2015-11-18-0.log
	ihome-framework-usage_2015-11-18-1.log


这里的配置引用了2个变量，`${log.base}`和`${log.proj}`，会在下一节说明。

### 变量设置
	
	<configuration scan="true" scanPeriod="30 seconds" debug="true">
		<property name="log.base" value="logs" />
		<property name="log.proj" value="${project.artifactId}" />
	</configuration>

目前为了所有的工程都能统一使用同1份logback.xml配置，但是打出来的日志文件能够区分工程，我们都配置了一个变量，就是`project.artifactId`

这个变量是需要maven在编译的时候，过滤一下resource，因此需要在pom.xml或者parent pom中配置如下：

	<build>
		<finalName>${project.artifactId}</finalName>
		<resources>
			<resource>
				<directory>src/main/resources</directory>
				<filtering>true</filtering>
				<excludes>
					<exclude>sentry-javaagent-home/*.jar</exclude>
				</excludes>
			</resource>
		</resources>
	</build>
maven的resource filter，就是会在编译的时候，过滤一下resource下的文件，发现有`${project.artifactId}`这样的标识的时候，会替换成当前project的artifactId，譬如对于ihome-framework-usage来说，配置成这样以后，编译后，target目录下的logback.xml看起来会是这样的：
	
	<configuration scan="true" scanPeriod="30 seconds" debug="true">
		<property name="log.base" value="logs" />
		<property name="log.proj" value="ihome-framework-usage" />
	</configuration>


注意：这里excludes掉了哨兵的资源文件，是因为之前发现替换哨兵的文件只会，会导致哨兵的jar包内容变掉

### 日志异步化

	<appender name="ASYNC-COMMON-APPENDER" class="ch.qos.logback.classic.AsyncAppender">
		<discardingThreshold>0</discardingThreshold>
		<queueSize>1024</queueSize>
		<appender-ref ref="COMMON-APPENDER" />
	</appender>

- queueSize：异步的日志队列的长度
- discardingThreshold：默认情况下，当队列长度达到80%的时候，会丢弃INFO级别以下的日志，来保证队列不阻塞；设置为0则表示不丢弃日志；

### 多环境问题

之前大家的logback配置，默认都是配置到stdout，但是上面我们也配置了将日志输出到文件；

- 如果只输出到stdout，则服务器上1天的日志量保存在单个文件，文件会太大
- 如果只输出到文件，则开发人员本地调试不方便
- 如果同时输出到stdout和文件，那么服务器上的日志则是会有2份

为了解决这个问题，引入了logback的Variable substitution和condition的功能：

	<property resource="disconf.properties" />
	<root>
		<level value="info" />
		<if condition='property("env").equals("rd")'>
			<then>
				<appender-ref ref="stdout" />
			</then>
		</if>
		<appender-ref ref="COMMON-APPENDER" />
	</root>

这里的解决办法就是，配置logback去读取resource下的`disconf.properties`，主要是读取到里面的`env`字段，读取后env会变成类似我们上面自定义的`log.base`这样的变量

然后在下面用if判断env是否为rd（即我们目前常用的开发环境），只有环境为rd的话，才把日志打印到stdout；

**注意：**
为了使用这个功能，需要把原来framework引入的`janino`给exclude掉，引入groupId为org.codehaus.janino的库；或者直接升级framework到最新版

		<dependency>
			<groupId>com.ihome.framework</groupId>
			<artifactId>framework-core</artifactId>
			<version>${ihome.framework.version}</version>
			<exclusions>
				<exclusion>
					<artifactId>janino</artifactId>
					<groupId>janino</groupId>
				</exclusion>
			</exclusions>
		</dependency>
		<dependency>
			<groupId>org.codehaus.janino</groupId>
			<artifactId>janino</artifactId>
			<version>2.7.8</version>
		</dependency>
	



