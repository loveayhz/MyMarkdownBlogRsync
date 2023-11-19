访问github会报超时404等，目前使用本地代理okhttp设置代理可以访问，v2RayN的代理可以通过

FastGithub还需要尝试

giuthub返回后的user信息

```
{
  "login": "loveayhz",
  "id": 83393411,
  "node_id": "MDQ6VXNlcjgzMzkzNDEx",
  "avatar_url": "https://avatars.githubusercontent.com/u/83393411?v=4",
  "gravatar_id": "",
  "url": "https://api.github.com/users/loveayhz",
  "html_url": "https://github.com/loveayhz",
  "followers_url": "https://api.github.com/users/loveayhz/followers",
  "following_url": "https://api.github.com/users/loveayhz/following{/other_user}",
  "gists_url": "https://api.github.com/users/loveayhz/gists{/gist_id}",
  "starred_url": "https://api.github.com/users/loveayhz/starred{/owner}{/repo}",
  "subscriptions_url": "https://api.github.com/users/loveayhz/subscriptions",
  "organizations_url": "https://api.github.com/users/loveayhz/orgs",
  "repos_url": "https://api.github.com/users/loveayhz/repos",
  "events_url": "https://api.github.com/users/loveayhz/events{/privacy}",
  "received_events_url": "https://api.github.com/users/loveayhz/received_events",
  "type": "User",
  "site_admin": false,
  "name": null,
  "company": null,
  "blog": "",
  "location": null,
  "email": null,
  "hireable": null,
  "bio": null,
  "twitter_username": null,
  "public_repos": 42,
  "public_gists": 0,
  "followers": 1,
  "following": 7,
  "created_at": "2021-04-29T07:21:37Z",
  "updated_at": "2023-10-05T06:14:07Z",
  "private_gists": 1,
  "total_private_repos": 11,
  "owned_private_repos": 11,
  "disk_usage": 6271,
  "collaborators": 0,
  "two_factor_authentication": false,
  "plan": {
    "name": "free",
    "space": 976562499,
    "collaborators": 0,
    "private_repos": 10000
  }
}
```

这里我们已经大致了解了auth2认证的一个流程

可以再参考这里的：https://blog.csdn.net/hou_ge/article/details/120291287

---

📅2023-11-14

接上次的记录，这里我们的github登录,

首先我们需要到github注册一个Auth applications,登录github后右键头像选择 `Settings/Developer settings/OAuth apps`,点击右边的 `new OAuth app`创建我们自己的 `OAuth app`

![github new OAuth app.png](http://ayhz.art:8084/zh-east-1/img/github%20new%20OAuth%20app.png)

地址填写

![github三方登陆auth app.png](http://ayhz.art:8084/zh-east-1/img/github三方登陆auth%20app.png)

点击完成后会生成一个 `Client Id`和 `Client secrets`这两个值要记住，记不住也可以删掉重新生成，生成的 `Client Id`和 `Client secrets`，需要在spring的yaml中配置oauth2.client，如下所示

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: eqweqeqewqewq
            client-secret: dasdadsadsadsad
            redirect-uri: http://192.168.1.2/login
            redirect-uri-template: http://192.168.1.2
            proxy:
              type: SOCKS
              host: 127.0.0.1
              port: 10808

```

其中 `spring.security.oauth2.client.registration`这个前缀是oauth2.client默认的，`github`是我们第三方登录app的名词，如果你还可以qq登录那就是qq,然后前四项是我们配置的github的OAuth app生成和填写的内容，其中特别需要注意的是这个 `spring.security.oauth2.client.registration.github.redirect-uri`

它是我们点击github登录按钮后转发的地址

我们首先是点击github登录按钮，如下

```
<v-btn href="http://192.168.1.2/api/oauth2/authorization/github">Github</v-btn>
```

这是auth2.client默认的地址，即

```
${severUrl}+/oauth2/authorization/ +${appName}
```

${severUrl}即配置中的spring.security.oauth2.client.registration.github.redirect-uri-template

也就是github的auth app设置中的Home url

点击这个地址，当我们的项目中引入了oauth2-client的依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-client</artifactId>
        </dependency>
```

同时yaml中配置了上面的oauth2的内容，那么就会在我们点击github登录按钮时，自动跳转到github的登录页或者授权页面去授权，授权成功后就会跳转到一个地址

```
Request URL: https://github.com/login/oauth/authorize?response_type=code&client_id=4e92d7dcf0dc4f7dafb4&scope=read:user&state=KmHOkUFj3zdoTm-hmtRD_F2K17An6QMtVPmsUNcz0GQ%3D&redirect_uri=http://192.168.1.2/login
```

这个地址最终会跳转到我们的redirect-uri的地址去，这里我们是调回了http://192.168.1.2/login，其实这里需要改。

这里我们在login页面或者说跳转到的这个页面做了一个处理，即拿到它返回的code和state去执行下一步，

```
mounted() {
    console.log("$route.query:", this.$route.query);
    if (this.$route.query.code) {
      this.code = this.$route.query.code;
      this.state = this.$route.query.state;
      this.client_id = this.$route.query.client_id;
      this.scope = this.$route.query.scope;
      console.log(this.code);
      this.axios({
        url: "/api/auth/callback",
        method: "post",
        params: {
          code: this.code,
          state: this.state,
        },
      })
        .then((response) => {
          console.log(response);
          //TODO 获取token并设置Authorization等，重定向到首页，即登录成功
        })
        .catch((err) => {
          console.log(err);
        });
    }
  },
```

这里我们是用“/auth/callbak”这个controller方法去处理了，这个方法里面我们要做的事情是，通过code和state去获取github的access_token，即这个认证用户的access_token，拿到了这个我们可以保存起来，然后下一步用access_token为header去获取用户的信息，

```
    //获取access_token
    public String getGithubAccessToken(String code, String state) {
        GithubAccessTokenDTO githubAccessTokenDTO = GithubAccessTokenDTO.builder()
                .client_id(clientId)
                .client_secret(clientSecret)
                .redirect_uri(redirectUri)
                .code(code)
                .state(state)
                .build();
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .proxy(githubProxy)
                .build();
        MediaType MediaType_JSON
                = MediaType.get("application/json; charset=utf-8");
        okhttp3.RequestBody requestBody = okhttp3.RequestBody.create(JSONObject.toJSONString(githubAccessTokenDTO),MediaType_JSON);
        Request request = new Request.Builder()
                .url("https://github.com/login/oauth/access_token")
                .post(requestBody)
                .build();
        try(Response response = okHttpClient.newCall(request).execute()){
            String responseBody = response.body().string();
            log.info("getGithubAccessToken:{}",responseBody);
            String token  = responseBody.split("&")[0].split("=")[1];
            return token;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
    //通过access_token去获取用户信息
    public GithubUserDTO getGithubUser(String token) {
        //curl --request GET \
        //--url "https://api.github.com/user" \
        //--header "Accept: application/vnd.github+json" \
        //--header "Authorization: Bearer USER_ACCESS_TOKEN" \
        //--header "X-GitHub-Api-Version: 2022-11-28"
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
                .proxy(githubProxy)
                .build();
        //Map<String,String> headersMap = new HashMap<>();
        //headersMap.put("Accept","application/vnd.github+json");
        //headersMap.put("Authorization","Bearer "+token);
        //headersMap.put("X-GitHub-Api-Version","2022-11-28");
        //Headers headers = Headers.of(headersMap);
        Request request = new Request.Builder()
                .url("https://api.github.com/user")
                //.headers(headers)
                .header("Accept","application/vnd.github+json")
                //.header("Authorization","Bearer "+token)
                .header("Authorization","token "+token)
                .header("X-GitHub-Api-Version","2022-11-28")
                .get()
                .build();
        try (Response response = okHttpClient.newCall(request).execute();){
            String responseBody = response.body().string();
            log.info("getGithubUser:{}",responseBody);
            GithubUserDTO githubUserDTO = JSON.parseObject(responseBody,GithubUserDTO.class);
            return githubUserDTO;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }
```

其实到这里我们就能获得用户的信息了。

但是jwt token的验证肯定是过不了这个验证的。

我们接下来要做的是，返回一个可以用于身份验证的token给前端以及编写合适的身份校验，让前端拿着这个token获取请求的时候，可以通过filter的身份校验，以及通过ContextHolder获取到对应的用户身份信息。

同时保存用户信息，让用户设置密码和邮件等等。

---

一个非常简单的错误：https://blog.csdn.net/Qingshan_z/article/details/128218701

okhttp调用的时候把 `String responseBody = response.body().string();`写成 `String responseBody = response.body().toString();`，一直返回

```
okhttp3.internal.http.RealResponseBody@5043b93c
```

---

⏲2023-11-14 19:02,未完待续，上面只是粗略的捋了一遍github三方登录的流程。

接下来我们还需要重新认证编写代码才能完成三方登录：

1. 跳转到redirect-uri的页面我们需要通过后端来处理，后端直接获取到code和state后，去处理
2. 后端拿到code和state后按照前面的内容去获取access_token和用户信息后，判断用户是否已存在，不存在则生成用户，保存username和access_token，与之前的截图token结合返回给用户jwttoken。

---

📅2023-11-15 15:47

继续昨天的内容，今天一番尝试后发现昨天说的不对，因为转发的redirect-uri的请求头accept的值为

```
Accept:
text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
```

也就是它请求的其实是html资源，当我们把redire-uri直接改为向服务器发送请求即/api/login?app=Github的时候，它被当作了静态资源请求进行拦截鉴权。

但是让我困惑的是为什么我的JwtLoginFilter没有对他进行拦截，因为前端页面的表单登录请求post /login是被我们自定义的这个过滤器拦截了的，而我们设置到转发的get /login却没有被拦截，其实只要它被拦截我们就能获取code和state,然后去请求acces_token和user信息。

于是debug跟踪了一下发现我们的JwtLoginFilter继承了UsernamePasswordAuthenticationFilter，而UsernamePasswordAuthenticationFilter又继承了AbstractAuthenticationProcessingFilter，

```
public class JwtLoginFilter extends UsernamePasswordAuthenticationFilter {
//...
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        if (!REQUEST_METHOD.equals(request.getMethod())) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        } else {
            //TODO 这里需要对github或者其他三方登录增加一个处理分支，即如果判断url（即redirect-uri中增加一个url?app=github）中的app=github且含有code和state字段，则调用github处理的方式
            //同样如果是其他app的三方登录，编写对应的处理方式。可以使用策略模式。
            String contentType = request.getHeader("Content-Type");
            String appName = request.getParameter("app");
            SysUser user = null;
            //json提交
            if (contentType.contains(DATA_FORMAT)) {
                user = this.getSecurityUser(request);
                if (user == null || user.getUsername() == null || user.getPassword() == null) {
                    throw new AuthenticationServiceException("Authentication failure: username or password can't be null.");
                }
                logger.info("账号登录：" + user.getUsername());
            } else {
                //表单提交
                String username = this.obtainUsername(request);
                String password = this.obtainPassword(request);
                if (username == null || password == null) {
                    throw new AuthenticationServiceException("Authentication failure: username or password can't be null.");
                }
                logger.info("账号登录：" + username);
//                user = (new SysUser()).setUsername(username.trim()).setPassword(password);
                user = new SysUser();
                user.setUsername(username.trim());
                user.setPassword(password);
            }
            request.setAttribute("account", user.getUsername());
            return this.getAuthenticationManager().authenticate(new UsernamePasswordAuthenticationToken(user.getUsername(), user.getPassword(), new ArrayList()));
        }
    }
}
```

其中AbstractAuthenticationProcessingFilter的处理逻辑中对是要需要认证进行了验证，验证的时候校验必须post /login才可以，否则直接进入一下个filter

```
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean implements ApplicationEventPublisherAware, MessageSourceAware {
    //...省略
    private RequestMatcher requiresAuthenticationRequestMatcher;
    //...省略
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        HttpServletRequest request = (HttpServletRequest)req;
        HttpServletResponse response = (HttpServletResponse)res;
        //判断是否需要当前的Filterj进行认证
        if (!this.requiresAuthentication(request, response)) {
            //不需要，下一个filter
            chain.doFilter(request, response);
        } else {
            //不需要，执行filter的逻辑
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Request is to process authentication");
            }

            Authentication authResult;
            try {
                //JwtLoginFilter继承并复写了attemptAuthentication(request, response)方法，在其中实现具体的登录校验
                authResult = this.attemptAuthentication(request, response);
                if (authResult == null) {
                    return;
                }
            }
            //省略
        }
    }
    //构造方法被子类UsernamePasswordAuthenticationFilter super
    protected AbstractAuthenticationProcessingFilter(RequestMatcher requiresAuthenticationRequestMatcher) {
        Assert.notNull(requiresAuthenticationRequestMatcher, "requiresAuthenticationRequestMatcher cannot be null");
        this.requiresAuthenticationRequestMatcher = requiresAuthenticationRequestMatcher;
    }
    //判断是否需要当前filter处理
    protected boolean requiresAuthentication(HttpServletRequest request, HttpServletResponse response) {
        return this.requiresAuthenticationRequestMatcher.matches(request);
    }
}
```

子类

```
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {
    public static final String SPRING_SECURITY_FORM_USERNAME_KEY = "username";
    public static final String SPRING_SECURITY_FORM_PASSWORD_KEY = "password";
    private String usernameParameter = "username";
    private String passwordParameter = "password";
    private boolean postOnly = true;
    //构造方法添加了new AntPathRequestMatcher("/login", "POST")，即必须为Post /login才执行
    public UsernamePasswordAuthenticationFilter() {
        super(new AntPathRequestMatcher("/login", "POST"));
    }
    //被JwtLoginFilter 覆写
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
       //省略
    }
}
```

也就是说，这里如果我们要修改登录拦截的path即/login，或者说要修改只允许POST请求，需要复写构造方法覆盖父类RequestMatcher requiresAuthenticationRequestMatcher属性，因为它影响判定是否执行filter。

一开始跟踪debug的时候找不到头绪，直到跟踪到FilterChainProxy这个类的doFilter方法

```
public class FilterChainProxy extends GenericFilterBean {
    public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
            if (this.currentPosition == this.size) {
                if (FilterChainProxy.logger.isDebugEnabled()) {
                    FilterChainProxy.logger.debug(UrlUtils.buildRequestUrl(this.firewalledRequest) + " reached end of additional filter chain; proceeding with original chain");
                }

                this.firewalledRequest.reset();
                this.originalChain.doFilter(request, response);
            } else {
                ++this.currentPosition;
                //这里可以跟踪到filterchain当中的所有filter以及它们的执行顺序，currentPosition就是它执行到filterchain的顺序的序号
                Filter nextFilter = (Filter)this.additionalFilters.get(this.currentPosition - 1);
                if (FilterChainProxy.logger.isDebugEnabled()) {
                    FilterChainProxy.logger.debug(UrlUtils.buildRequestUrl(this.firewalledRequest) + " at position " + this.currentPosition + " of " + this.size + " in additional filter chain; firing Filter: '" + nextFilter.getClass().getSimpleName() + "'");
                }

                nextFilter.doFilter(request, response, this);
            }
        }
    }
}
```

![FilterChainProxy跟踪debug.png](http://ayhz.art:8084/zh-east-1/img/FilterChainProxy跟踪debug.png)

如图所示，我们只需要跟踪到currentPosition为8的时候，就可以跟踪到JwtLoginFilter的执行逻辑，同理，对于里面的每一个Filter都可以找到它的具体逻辑。

在跟踪的时候我们也看到了到OAuth2AuthorizationRequestRedirectFilter执行的时候它拦截了 `/oauth2/authorization/github`并Response.sendRedirect到https://github.com/login/oauth/authorize，之后，因为给github发送的请求中有redirect-uri，github返回内容到redirect-uri,redirect-uri就是我们之前定义的http://192.168.1.2/api/login?app=Github这个地址，

因为我们的服务器已经Response.sendRedirect()返回了，所以Filter到这里就已经结束了。![redireuri.png](http://ayhz.art:8084/zh-east-1/img/redireuri.png "redire-uri")

但是因为github会重定向到redirect-uri，所以浏览器又向我们发送了redire-uri的请求。

接下来继续处理redirect-uri这个请求。

继续跟踪的时候发现RedireUri那个filter之后还紧跟着一个Oauth2开头的filter,跟踪进去之后发现它的antMatcher的地址为

```
Ant [pattern='/login/oauth2/code/*']
```

哈哈哈。我发现了什么？原来它Oauth2有自定义处理的内容，但是你的redire-uri的地址必须匹配'/login/oauth2/code/*'才能被处理。

![AbstractAuthenticationProcessingFilter.png](http://ayhz.art:8084/zh-east-1/img/AbstractAuthenticationProcessingFilter.png)

---

于是我搜索关键字/login/oauth2/code/github的时候，发现了这篇博客

[【OAuth2.0 Client 总结】对接github第三方登录以及其他第三方登录总结](https://blog.csdn.net/qq_43430343/article/details/130271536)

[oauth2](https://docs.spring.io/spring-security/site/docs/5.2.12.RELEASE/reference/html/oauth2.html "oauth2")

突然发现，原来我之前在csdn上看到的那几篇都是垃圾，一个个互相抄，我也试图在那个级别的认知逻辑里把这个东西写出来，当然如果按照这个逻辑去写，我再转发回/login页面，由login页面去发送post /login请求，携带上code和state，然后再JwtLoginFilter的逻辑里去判断app去处理，是可以实现的。

但是我不知道的是，原来Oauth2的依赖里已经预置了很多类来处理Oauth2的登录，我还是继续去造轮子，只是知道了Oauth2的流程但是不知道借助框架干了什么，还能做什么。其实这样是很蠢的。

而且在CSDN的搜索其实我找了不只那么几篇博客，但是为什么没有看到这篇呢？在我点赞之前只有一个赞两个收藏，按照csdn的推荐逻辑，确实不太容易能刷到，我能找到还是因为匹配了完整的关键词。

也就是说，如果技术博客按照点赞量和收藏量这些交互数据作为推荐系统的依据，那么毫无疑问在国内这个二手知识盛行而且存在语言壁垒的情况下，CSDN的这种推荐逻辑或者说国内的博客论坛的推荐逻辑将会把很多人带到沟里去。

但是不得不说，我其实之前连spring-security的大部分逻辑都没有搞清楚，只是按照csdn的热门博客了解了最主要的逻辑，就直接写了出来，虽然说实现了逻辑但是对于其流程依然不是特别清楚，今天下午debug跟踪的时候才明白其中Filter的执行链路和怎么跟踪到具体逻辑。

也是到这里才意识到框架内预设了处理的类，需要我们去继承去实现，而不是重复造轮子。

甚至于由于默认的心理，依赖CSDN这种二手知识，我都没有去寻找官方文档的意识。这是很可怕的。

---

⏲2023-11-18 03:37

这两天研究了一下，发现OAuth2的认证，它是这样的，如果我们要使用spring-boot-starter-oauth2-client的框架，需要了解它内部预制的一些类和机制。

```
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-client</artifactId>
        </dependency>
```

其中前面那两个filter，它们分别监听{baseUrl}/oauth/authorize/{clientId}和{baseUrl}/login/oauth2/code/{clientId},所以我们的href={baseUrl}/oauth/authorize/{clientId}?redirect-uri=${redirect-uri}，

因为后端OAuth2AuthorizationRequestRedirectFilter匹配到{baseUrl}/login/oauth/authorize/{clientId}之后，会帮我们向{clientId}服务器发起请求,这个请求后面带着我们要重定向的地址。

```
https://github.com/login/oauth/authorize?response_type=code&client_id=2564242a8df76123134324&scope=user:email%20read:user&state=sdwrwr42423ENjrD3oB7sP4234pBg%3D&redirect_uri=http://192.168.1.2/login/oauth2/code/github
```

github服务器处理这个请求得到一个code和state，这个state会和之前的state保持一致确保没有被篡改，估计框架会处理这个细节。

然后github把这个code和state作为url参数放在我们重定向的地址后面发起重定向,所以我们重定向的这个地址应该是一个前端页面，用来接收code和state。

```
http://192.168.1.2/login/oauth2/code/github?code=fsdfffsd244228f3b9&state=sdwrwr42423ENjrD3oB7sP4234pBg%3D
```

我们拿到code和state之后再去发送请求到{baseUrl}/login/oauth2/code/{clientId}

```
mounted() {
            let _this = this;
            console.log("$route.query:"+ this.$route.querym+"$route.path:"+this.$route.path);
            // const token = this.getUrlParameter('token');
            // const error = this.getUrlParameter('error');
            if (this.$route.query.code) {
                this.code = this.$route.query.code;
                this.state = this.$route.query.state;
                this.client_id = this.$route.query.client_id;
                this.scope = this.$route.query.scope;
                console.log(this.code);
                this.axios({
                    url: "/api/login/oauth2/code/github",
                    method: "get",
                    params: {
                        code: this.code,
                        state: this.state,
                    },
                })
                    .then((response) => {
                        //console.log("hhh:"+response);
                        console.log(response);
          // 将用户token保存到vuex中
          // _this.changeLogin({ Authorization: _this.userToken });
          var user = response.data.data;
          console.log(user);
          _this.changeLogin({
            Authorization: user.token,
            username: user.username,
            avatar: user.avatar,
          });
          _this.loading = false;
          _this.$router.push("/article/all/list");
                        //TODO 获取token并设置Authorization等，重定向到首页，即登录成功
                    })
                    .catch((err) => {
                        console.log(err);
                    });
            }
        },
```

这个时候第二个过滤器OAuth2AuthentcationFilter截取到请求{baseUrl}/login/oauth2/code/{clientId}，它会拿着code和state去获取access_token，然后又拿着access_token去获取用户信息，也就是我们之前手动去做的那些事情，他帮我们做了，然后我们只需要写个

```
CustomOauth2Service
```

去处理登录的逻辑

以及成功登录后的处理逻辑

```
OAuth2LoginSuccesHandler
```

这样我们就可以拿着这个Handler处理完后的数据可能包含token等信息返回给rediret-uri页面，redirect-uri页面收到返回数据，判断成功登录后设置用户数据跳转home页。这样就算完成了。

但是需要注意的是，不要在yml中设置 `redirect-uri-template`

我们的redirect-uri的前端router，应该设置为

```
{
    //path: "/login/oauth2/code/github",
    path: "/login/oauth2/code/{clientId}",
    component: Auth2Redirect
  },
```

这样，我们可以监听不同的{clientId｝,在mounted里根据{clientId｝做处理。

---

# 这里面最令人困惑的问题在于，这个redirect-uri到底能不能自定义配置，比如我配制成/oauth2/redirect?

如果你打算使用框架内去自动帮你获取access_token和userinfo用户信息的部分，并且没有配置ClientRegistration的response的redirect-uri-template，那么就是不能自定义，必须设置成 `{baseUrl}/login/oauth2/code/{clientId}`的形式,否则因为它默认的地址就是这个形式，并且还会对request和response的redirect-uri进行校验，你请求的时候自定义了但是没有自定义response的redirect-uri-template，这个校验就过不了。

![大坑.png](http://ayhz.art:8084/zh-east-1/img/大坑.png)

自定义response的redirect-uri-template如下

```
Important
You also need to ensure the ClientRegistration.redirectUriTemplate matches the custom Authorization Response baseUri.

The following listing shows an example:

return CommonOAuth2Provider.GOOGLE.getBuilder("google")
    .clientId("google-client-id")
    .clientSecret("google-client-secret")
    .redirectUriTemplate("{baseUrl}/login/oauth2/callback/{registrationId}")
    .build();
```

---

当然如果你打算自己处理拿到code和state之后去获取access_token和用户信息的逻辑，那么就无所谓了，你可以自己写controller方法去处理，就像一开始我们那样的写法，或者自定义一个Filter去匹配监听。

这样就无所谓了。

---

也正是由于它默认的response的redirect-uri-template为 `{baseUrl}/login/oauth2/code/{clientId}`的形式，同时，请求request的redirect-uri和返回response的redirect-url-template又必须一致，所以限制了request的redirect-uri必须是 `{baseUrl}/login/oauth2/code/{clientId}`

看了好多博客都是说redirect-uri即github的callback可以自定义，但是却没有一个人提到要修改response的redirect-uri-template和redirect-uri必须保持一致，也许它们都是自定义处理的，我记不清了。

甚至没有人讲清楚redirect-uri是前端页面还是后端地址，同时它如果是后端地址的话又恰好能被filter捕获触发连锁反应，更增加了迷惑性。

所以，最简单的就是redirect-uri保持 `{baseUrl}/login/oauth2/code/{clientId}`的形式，前段的路由监听 `/login/oauth2/code/{clientId}`，根据clientId去分别处理。

---

另外由于github的访问的特殊性，需要使用代理，自定义的话可以使用Proxy代理，

如果使用框架的话，则需要去实现它发送请求的类并覆盖它

```
@Configuration
public class WebServerAutoConfiguration {

    @Bean
    public ConfigurableServletWebServerFactory webServerFactroy() {
        UndertowServletWebServerFactory factory = new UndertowServletWebServerFactory();
//        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        ErrorPage errorPage403 = new ErrorPage(HttpStatus.FORBIDDEN, "/error/403");
        ErrorPage errorPage404 = new ErrorPage(HttpStatus.NOT_FOUND, "/error/404");
        ErrorPage errorPage500 = new ErrorPage(HttpStatus.INTERNAL_SERVER_ERROR, "/error/500");
        factory.addErrorPages(errorPage403, errorPage404, errorPage500);
        return factory;
    }

    @Bean
    public CustomUserInfoRestTemplateCustomizer customUserInfoRestTemplateCustomizer() {
        return new CustomUserInfoRestTemplateCustomizer();
    }

    private static class CustomUserInfoRestTemplateCustomizer implements UserInfoRestTemplateCustomizer {
        @Value("${hspring.security.oauth2.client.registration.github.proxy.connect-timeout:500}")
        private int connectTimeout;

        @Value("${spring.security.oauth2.client.registration.github.proxy.read-timeout:30000}")
        private int readTimeout;

        @Value("${spring.security.oauth2.client.registration.github.proxy.hostname:127.0.0.1}")
        private String proxyHost;

        @Value("${spring.security.oauth2.client.registration.github.proxy.port:10808}")
        private int proxyPort;

        @Override
        public void customize(OAuth2RestTemplate template) {
            template.setRequestFactory((uri, httpMethod) -> {
                SimpleClientHttpRequestFactory clientHttpRequestFactory = new SimpleClientHttpRequestFactory();
                clientHttpRequestFactory.setConnectTimeout(connectTimeout);
                clientHttpRequestFactory.setReadTimeout(readTimeout);
                if (StringUtils.isNoneEmpty(proxyHost)) {
                    Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress(proxyHost, proxyPort));
                    clientHttpRequestFactory.setProxy(proxy);
                }
                return clientHttpRequestFactory.createRequest(uri, httpMethod);
            });
        }
    }

}
```

除此之外还涉及cors跨域的处理，前段的proxy设置要和baseUrl保持一致，不能使用localhost等

---

1. github的Settings Developer settings中设置的Authorization callback URL：`http://192.168.1.2/login/oauth2/code/github`
2. yml中的spring.security.oauth2.client.registration.{clientId}配置

```
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: 42325454654464
            client-secret: huadjaashkdha723748274
            #redirect-uri: http://192.168.1.2/api/login/oauth2/code/github
            #redirect-uri: http://192.168.1.2:8100/login/oauth2/code/github
            redirect-uri: http://192.168.1.2/login/oauth2/code/github
            #redirect-uri: http://192.168.1.2/oauth2/redirect
            #redirect-uri-template: http://192.168.1.2
            scope:
              - user:email
              - read:user
            proxy:
              type: SOCKS
              host: 127.0.0.1
              port: 10808
              connect-timeout: 500
              read-timeout: 30000
```


vscode中发起github登录的href链接

处理redirect-uri的前端页面Auth2Redirect.vue的路由router/index.js的设置

```
const routes = [
  {
    path: "/",
    component: Home
  },
  {
    path: "/login/oauth2/code/:clientId",
    component: Auth2Redirect
  },
  {
    path: "/login",
    component: Login
  },
  //...
}
```

处理redirect-uri的前端页面Auth2Redirect.vue的初始加载方法mounted发起的请求

```
mounted() {
            let _this = this;
            console.log("$route.query:"+ this.$route.querym+"$route.path:"+this.$route.path);
            // const token = this.getUrlParameter('token');
            // const error = this.getUrlParameter('error');
            if (this.$route.query.code) {
                this.code = this.$route.query.code;
                this.state = this.$route.query.state;
                this.client_id = this.$route.query.client_id;
                this.scope = this.$route.query.scope;
                console.log(this.code);
                this.axios({
                    url: "/api/login/oauth2/code/github",
                    method: "get",
                    params: {
                        code: this.code,
                        state: this.state,
                    },
                })
                    .then((response) => {
                        //console.log("hhh:"+response);
                        console.log(response);
          // 将用户token保存到vuex中
          // _this.changeLogin({ Authorization: _this.userToken });
          var user = response.data.data;
          console.log(user);
          _this.changeLogin({
            Authorization: user.token,
            username: user.username,
            avatar: user.avatar,
          });
          _this.loading = false;
          _this.$router.push("/article/all/list");
                        //TODO 获取token并设置Authorization等，重定向到首页，即登录成功
                    })
                    .catch((err) => {
                        console.log(err);
                    });
            }
        },
```

其中的redirect-uri都必须要把持一致
