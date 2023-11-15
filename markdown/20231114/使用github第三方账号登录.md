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

其中 `spring.security.oauth2.client.registration`这个前缀是oauth2.client默认的，`github`是我们第三方登录app的名词，如果你还可以qq登录那就是qq,然后前四项是我们配置的github的OAuth app生成和填写的内容，其中特别需要注意的是这个`spring.security.oauth2.client.registration.github.redirect-uri`

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

⏲2023-11-04 19:02,未完待续，上面只是粗略的捋了一遍github三方登录的流程。

接下来我们还需要重新认证编写代码才能完成三方登录：

1. 跳转到redirect-uri的页面我们需要通过后端来处理，后端直接获取到code和state后，去处理
2. 后端拿到code和state后按照前面的内容去获取access_token和用户信息后，判断用户是否已存在，不存在则生成用户，保存username和access_token，与之前的截图token结合返回给用户jwttoken。
