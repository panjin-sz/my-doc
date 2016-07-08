# maven构建多web项目 #

首先构建普通java项目，命令如下

    mvn archetype:generate -DgroupId={project-packaging} -DartifactId={project-name} -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false

    mvn archetype:generate -DgroupId=com.panjin.framework -DartifactId=my-framework -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false

在文件夹下可以看到src文件夹和pom.xml文件，目录结构如下：
![](http://i.imgur.com/2gSYVxL.png)
删除掉src文件夹，然后修改pom.xml文件，将<packaging>jar</packaging>修改为<packaging>pom</packaging>，pom表示它是一个被继承的模块，修改后的内容如下：
![](http://i.imgur.com/rtHgRBr.png)

同理，在my-framework目录下，使用上面命令，构建普通java项目。这时在my-framework目录下的pom.xml文件内容会自动变成：
![](http://i.imgur.com/6CP1NC8.png)

修改framework-basic目录中的pom.xml文件，把<groupId>me.gacl</groupId>和<version>1.0-SNAPSHOT</version>去掉，加上<packaging>jar</packaging>，因为groupId和version会继承my-framework中的groupId和version，packaging设置打包方式为jar，同时添加对framework-basic模块的依赖

按照上面的步骤，依次添加framework-business、framework-core、framework-tool、framework-web，最终目录结构如下：
![](http://i.imgur.com/iBFD5Sm.png)

目录结构如下

![](http://i.imgur.com/5l7X9YN.png)

