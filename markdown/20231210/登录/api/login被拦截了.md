![20180727141421_ojmml.jpg](http://ayhz.art:8084/zh-east-1/img/20180727141421_ojmml.jpg)

现在的问题是部署到服务器之后登录的时候似乎被jwt拦截器拦截了，问题是/login应该是springsecurity拦截的antMatcher,它不应该被拦截。

而且在本地环境就是正常的换到服务器nginx环境就有问题了，考虑本地也装个nginx测试一下到底为神马被拦截，另外服务器使用的域名而本机使用的是局域网ip。

考虑用arthas调试一下.

使用arthas过程不太顺利，连接不上，有人说因为我的docker镜像是基于apline的

建议改成centos或者ubuntu的基础镜像

所以，需要先改为用centos构建自己的oracle jdk镜像

然后再尝试运行

参考：https://blog.csdn.net/weixin_48040732/article/details/129087662

我们先构建基础的镜像centos-jdk8

```
FROM centos:7

# 描述作者和邮箱,可只写其中一个,也可二个都写
MAINTAINER ayhz

# 时区与字符设置UTF-8并配置环境
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8

RUN yum -y update && \
    yum -y install kde-l10n-Chinese && \
    yum -y reinstall glibc-common && \
    localedef -c -f UTF-8 -i zh_CN zh_CN.utf8 && \
    echo 'LANG="zh_CN.UTF-8"' > /etc/locale.conf

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN yum clean all
# 在容器里面创建一个java目录,用来放拷贝过来的文件,RUN用来执行linux命令
RUN mkdir /usr/local/java
# 把jdk-8u381-linux-x64.tar.gz添加到容器中,文件必须要和你的Dockerfile在同一级目录里面,ADD命令会自动将.gz文件拷贝到容器里面并自动解压
ADD jdk-8u381-linux-x64.tar.gz /usr/local/java/
# 配置java环境变量
ENV     JAVA_HOME       /usr/local/java/jdk1.8.0_381
ENV     JRE_HOME        /usr/local/java/jdk1.8.0_381/jre
ENV     CLASSPATH       .:$JAVA_HOME/lib:$JRE_HOME/lib:$CLASSPATH
ENV     PATH            $PATH:$JAVA_HOME/bin:$JRE_HOME/bin
```

参考构建镜像并推送至dockerhub

```
docker build -t centos-jdk8u381:v2.0 .
docker tag centos-jdk8u381:v2.0 ayuehunzi/centos-jdk8u381:v2.0
docker login
docker push ayuehunzi/centos-jdk8u381:v2.0
```

备份之前的jar包的Dockerfile

```
#设置镜像基础，jdk8
FROM ayuehunzi/alpine-jdk8u261:v.1.0
#维护人员信息
MAINTAINER AYHZ
# Alpine Linux 使用的不是标准gnu libc (glibc)，而是musl libc,导致arthas连接不上

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' > /etc/timezone
ENV LANG='zh_CN.UTF-8'
#设置镜像对外暴露端口
EXPOSE 8100
#将当前 target 目录下的 jar 放置在根目录下，命名为 app.jar，推荐使用绝对路径。
ADD target/blog-rest.jar /blog-rest.jar
COPY arthas-bin.zip /arthas.zip
# RUN  apk update && apk upgrade && apk add unzip && apk add curl
RUN mkdir /arthas-tool && unzip /arthas.zip -d /arthas-tool

#执行启动命令
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/blog-rest.jar"]
```

新的：

```
#设置镜像基础，jdk8
FROM ayuehunzi/centos-jdk8u381:v2.0
#维护人员信息
MAINTAINER AYHZ
#设置镜像对外暴露端口
EXPOSE 8081
#RUN mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
#RUN wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
#RUN yum update && yum clean
RUN yum install -y unzip
#将当前 target 目录下的 jar 放置在根目录下，命名为 app.jar，推荐使用绝对路径。
ADD target/blog-rest.jar /blog-rest.jar
COPY arthas-bin.zip /arthas.zip
RUN mkdir /arthas-tool && unzip /arthas.zip -d /arthas-tool

#执行启动命令
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/blog-rest.jar"]
```

> Error response from daemon: manifest for ayuehunzi/centos-jdk8u261:v1.0 not found: manifest unknown: manifest unknown
>
> 原因在于之前写错了镜像名称。。。因为偷懒写了个只把apline改成了centos，但是我们jdk版本也变了，所以镜像名称是错的找不到

怀疑之前一直写错用的错误的apline-jdk的镜像

arthas解压后，有一个arthas.properties的文件

```
#arthas.config.overrideAll=true
arthas.telnetPort=3658
arthas.httpPort=8563
arthas.ip=127.0.0.1

# seconds
arthas.sessionTimeout=1800

#arthas.enhanceLoaders=java.lang.ClassLoader

# https://arthas.aliyun.com/doc/en/auth
# arthas.username=arthas
# arthas.password=arthas

# local connection non auth, like telnet 127.0.0.1 3658
arthas.localConnectionNonAuth=true

#arthas.appName=demoapp
#arthas.tunnelServer=ws://127.0.0.1:7777/ws
#arthas.agentId=mmmmmmyiddddd

#arthas.disabledCommands=stop,dump
```

其中配置的arthas.telnetPort、arthas.httpPort、arthas.tunnelServer等要与与springboot的yaml中配置一致

```
arthas:
  agent-id: 1234567890
  tunnel-server: ws://localhost:7777/ws
  http-port: 8563
  telnet-port: 3658
  username: xiangpeach
  password: xiangpeach
  app-name: ${spring.application.name}
```

然后通过 `java -jar arthas-boot.jar` 启动arthas,再配合idea的插件和arthas的使用教程

我查看了一下拦截器，发现登录请求的地址在后端依然是 `/api/login`

至此我确信是nginx的配置问题，因为我是要去除/api的前缀的，但是没有去除

参考：[nginx反向代理后总是有前缀，2个小技巧，教会你去除UR前缀无残留](https://zhuanlan.zhihu.com/p/291144872)

修改完nginx并重启后后再次登录，发现直接登录成功了，后端的请求url也变成了/login

但是我打开文章看的时候，发现中文还是乱码，但是乱码的形式和之前不太一样，之前是升级mysql版本后BLOB和java的数据转换的问题，我确信已经解决了。

我猜想应该是我配置的这个centos的镜像的中文环境的问题，经过一番尝试，即前面的 `ayuehunzi/centos-jdk8u381:v2.0` 镜像中增加了中文环境变量和安装了中文环境的包等，之后就正常了。

但是现在的问题是，当我尝试使用github登录时，发现还是报错，猜想应该是fastGithub的代理的问题，可能需要在镜像打包时将fastGithub打包进去并以服务方式启动代理。

---

发现一个新的问题，登录成功后查询列表的时候出现了问题，报空指针异常

```
2023-12-10 01:54:54.519 DEBUG 1 --- [ XNIO-1 task-21] s.w.s.m.m.a.RequestMappingHandlerMapping : Mapped to art.ayhz.blog.controller.ArticleController#listArticlePage(Integer, PageReq)
2023-12-10 01:54:54.519 DEBUG 1 --- [ XNIO-1 task-21] o.s.w.s.handler.SimpleUrlHandlerMapping  : Mapped to ResourceHttpRequestHandler ["classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", "classpath:/public/", "/"]
2023-12-10 01:54:54.521 ERROR 1 --- [ XNIO-1 task-21] io.undertow.request                      : UT005023: Exception handling request to /article/all/list/1

java.lang.NullPointerException: null
	at java.util.LinkedList$ListItr.next(LinkedList.java:893) ~[na:1.8.0_381]
	at art.ayhz.blog.filter.JwtAuthenticationFilter.ignore(JwtAuthenticationFilter.java:182) ~[classes!/:na]
	at art.ayhz.blog.filter.JwtAuthenticationFilter.doFilterInternal(JwtAuthenticationFilter.java:68) ~[classes!/:na]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.2.2.RELEASE.jar!/:5.2.2.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:334) ~[spring-security-web-5.2.1.RELEASE.jar!/:5.2.1.RELEASE]
	at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:200) ~[spring-security-web-5.2.1.RELEASE.jar!/:5.2.1.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:334) ~[spring-security-web-5.2.1.RELEASE.jar!/:5.2.1.RELEASE]
	at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:200) ~[spring-security-web-5.2.1.RELEASE.jar!/:5.2.1.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:334) ~[spring-security-web-5.2.1.RELEASE.jar!/:5.2.1.RELEASE]
	at org.springframework.security.web.authentication.AbstractAuthenticationProcessingFilter.doFilter(AbstractAuthenticationProcessingFilter.java:200) ~[spring-security-web-5.2.1.RELEASE.jar!/:5.2.1.RELEASE]
	at org.springframework.security.web.FilterChainProxy$VirtualFilterChain.doFilter(FilterChainProxy.java:334) ~[spring-security-web-5.2.1.RELEASE.jar!/:5.2.1.RELEASE]
```

查看这一段代码，发现原来是代码中使用了LinkedList遍查遍插入的线程安全问题

```
   private boolean ignore(HttpServletRequest request) {
        List<String> curIgnore = securityProperties.getHttpIgnore();
        if(CollectionUtil.isEmpty(curIgnore)){
            return true;
        }
        for (String ignore : curIgnore) {
            if (new AntPathRequestMatcher(ignore).matches(request)) {
                return true;
            }
        }
        return false;
    }
//    private boolean ignore(HttpServletRequest request) {
//        List<String> curIgnore = securityProperties.getHttpIgnore();
          //这里存在线程安全问题，我们将HTTP_IGNORE的初始化删掉，忽略的url都放入yaml配置中
//        if (!HTTP_IGNORE.containsAll(curIgnore)) {
//            HTTP_IGNORE.addAll(curIgnore);
//        }
//        for (String ignore : HTTP_IGNORE) {
//            if (new AntPathRequestMatcher(ignore).matches(request)) {
//                return true;
//            }
//        }
//        return false;
//    }
```

修改之后就正常了。

但是目前github登录还是失败，经过验证fastgithub是正常运行的，明天再看
