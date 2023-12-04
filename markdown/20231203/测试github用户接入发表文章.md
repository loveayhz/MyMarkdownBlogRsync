# 测试github用户接入发表文章

之前我们看OAuth2User的文档，然后结合自己的jwtToken校验，但是之前没有token，Filter校验不通过，我看了一下Filter的token的验证过程，成功后需要写入

```
SecurityContextHolder.getContext().setAuthentication(authenticationToken);
```

---

```
private UsernamePasswordAuthenticationToken getAuthentication(String header) {
        String tokenPrefix = this.securityProperties.getTokenPrefix();
        if (!StringUtils.hasLength(tokenPrefix)) {
            tokenPrefix = "Bearer";
        }
        String token = header.replace(tokenPrefix, "");
        String principal = this.jwtTokenUtil.getUserName(token);
        if (principal != null) {
            SysUser user = this.jwtTokenUtil.getSecurityUser(token);
            UserContextHolder.setUser(user);
            return new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
        } else {
            return null;
        }
    }
```

在之前一直觉得我们用Oauth2了authenticationToken应该是使用OAuth2对应的 `OAuth2AuthenticationToken`或者其他的一个authenticationToken，于是想到我们在解析的时候已拿到OAuth2User了，`OAuth2AuthenticationToken`的构造方法也只需要一个OAuth2User即可，于是在Oauth2验证成功后将实现了OAuth2User的用户CustomeOauth2User作为user创建token，但是这个信息太大了而我们的token要作为header进行校验，这样的话，直接报了`431 (Request Header Fields Too Large)`

尝试修改header大小也没起作用，这个返回的token就有20多kb，我觉得其实不应该这样，因为我们已经验证成功了，只是给校验一个凭证即可。`OAuth2AuthenticationToken`应该是如果有这个就能直接验证了，所以它应该不是这样用的，因为我们已经验证通过了，所以这个凭证不一定是 `OAuth2AuthenticationToken`，用之前的 `UsernamePasswordAuthenticationToken` 也是可以的，而且我们之前基于SysUser开发，Oauth2验证的用户也必须要创建关联的SysUser，这样我们直接用原来的jwt token的逻辑,将关联的SysUser写入token即可，然后Filter的校验逻辑完全不用变，也不用区分Oauth2验证的用户还是用户名密码登录的用户。

经过测试，完全可行。

感觉自己就是被绕进去了，我们的目标只是利用三方去验证登录，并且我们关联到我们自己的user了，验证登录的逻辑完全沿用原来的逻辑就可以了。

![1613798142934spH5SFph.jpeg](http://ayhz.art:8084/zh-east-1/img/1613798142934spH5SFph.jpeg)
