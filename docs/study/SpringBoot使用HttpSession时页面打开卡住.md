# SpringBoot使用HttpSession时页面打开卡住

###原因

项目中使用了Tomcat Embed做为内嵌WEB服务器，而Tomcat在生成session ID时会使用org.apache.catalina.util.SessionIdGeneratorBase来产生安全随机类SecureRandom实例。为了算法保密性较强，须要用到伪随机数生成器，Tomcat用到的是SHA1PRNG算法，为了获得随机种子，在Linux中，通常从/dev/random或/dev/urandom中产生，二者原理都是利用系统的环境噪声产生必定数量的随机比特，区别在于系统环境噪声不够时，random会阻塞，而urandom会牺牲安全性避免阻塞。



### 解决方案

解决方法1：

启动参数添加-Djava.security.egd=file:/dev/urandom，如：架构

```
java -Djava.security.egd=file:/dev/urandom -jar xxxxx.jar
```



解决方法2：

修改$JAVA_HOME/jre/lib/security/java.security，找到securerandom.source并修改：dom

```
securerandom.source=file:/dev/urandom
```



解决方法3：

不使用tomcat作为容器，改用undertow，修改pom文件：

```
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
			<exclusions>
				<!--排除tomcat依赖-->
				<exclusion>
					<artifactId>spring-boot-starter-tomcat</artifactId>
					<groupId>org.springframework.boot</groupId>
				</exclusion>
			</exclusions>
		</dependency>

		<!--undertow容器-->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-undertow</artifactId>
		</dependency>

```

