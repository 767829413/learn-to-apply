# OAuth2介绍

## 定义和用途

OAuth2是一种授权框架，用于允许用户授权第三方应用访问其受保护的资源，而无需共享其凭据。它提供了一种安全且灵活的方式来管理用户的授权和访问权限。

## 工作原理

OAuth2使用一种基于令牌（Token）的机制来实现授权。当用户授权第三方应用访问其资源时，OAuth2会颁发一个访问令牌（Access Token），该令牌将用于访问受保护的资源。通过这种方式，用户的凭据得以保持私密，同时第三方应用仅能访问其被授权的资源。

## 核心概念

- 身份提供者（Identity Provider）：负责用户身份验证和授权
- 客户端（Client）：第三方应用程序，请求访问用户受保护的资源

- 授权服务器（Authorization Server）：负责颁发访问令牌（Access Token）

- 资源服务器（Resource Server）：存储受保护的资源

## 授权流程

OAuth2支持多种授权流程，以下是常见的几种：

- 授权码授权流程（Authorization Code Grant）
  1. 客户端导向用户到授权服务器。
  2. 用户提供授权，授权服务器返回授权码。
  3. 客户端使用授权码请求访问令牌。
  4. 授权服务器验证授权码，颁发访问令牌给客户端
  5. 客户端使用访问令牌访问受保护的资源。
  6. 资源服务器返回客户端请求的资源
  7. ![授权码授权流程](https://pic.imgdb.cn/item/6576fb99c458853aef8826c9.jpg)

- 隐式授权流程（Implicit Grant）
  1. 客户端导向用户到授权服务器。
  2. 用户提供授权，授权服务器直接返回访问令牌。
  3. 客户端从前端获取访问令牌。
  4. 客户端使用访问令牌访问受保护的资源。
  5. 资源服务器返回客户端请求的资源
  6. ![隐式授权流程](https://pic.imgdb.cn/item/6576fc5cc458853aef8b3293.jpg)

- 密码授权流程（Resource Owner Password Credentials Grant）
  1. 用户提供用户名和密码给客户端。
  2. 客户端使用用户提供的凭据请求访问令牌。
  3. 授权服务器验证凭据，颁发访问令牌给客户端。
  4. 客户端使用访问令牌访问受保护的资源。
  5. 资源服务器返回客户端请求的资源
  6. ![密码授权流程](https://pic.imgdb.cn/item/6576fd7ac458853aef901362.jpg)

- 客户端凭证授权流程（Client Credentials Grant）
  1. 客户端使用自己的凭证请求访问令牌。
  2. 授权服务器验证客户端凭证，颁发访问令牌给客户端。
  3. 客户端使用访问令牌访问受保护的资源。
  4. 资源服务器返回客户端请求的资源
  5. ![客户端凭证授权流程](https://pic.imgdb.cn/item/65770011c458853aef9aac2f.jpg)

## 安全性考虑

OAuth2的安全性考虑包括：

- 令牌的安全性和保护
- 保护令牌传输的重要性
- 令牌刷新和撤销的重要性

## 使用案例和实际应用场景

OAuth2广泛应用于多种场景，例如：

- 社交媒体登录：用户可以使用其社交媒体帐号登录第三方应用
- API访问：第三方应用可以通过OAuth2获取访问API的权限

## 演示示例（Go语言）

### 需要实现了一个简单的OAuth2授权服务器

这里直接就是使用一个github上开源的[OAuth2授权服务器](https://github.com/767829413/tmp-exec/tree/main/oauth2-demo)

运行OAuth2授权服务器

```bash
go run ./main.go 
2023/12/19 17:40:09 Dumping requests
2023/12/19 17:40:09 Server is running at 9096 port.
2023/12/19 17:40:09 Point your OAuth client Auth endpoint to http://localhost:9096/oauth/authorize
2023/12/19 17:40:09 Point your OAuth client Token endpoint to http://localhost:9096/oauth/token
```

运行一个客户端

```bash
go run ./main.go 
2023/12/19 17:23:32 Client is running at 9094 port.Please open http://localhost:9094
```

### 模拟OAuth2的四种授权模式

#### 授权码模式（Authorization Code Grant）：

- 直接请求: http://localhost:9094

client代码:

```go
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		u := config.AuthCodeURL("xyz",
			oauth2.SetAuthURLParam("code_challenge", genCodeChallengeS256("s256example")),
			oauth2.SetAuthURLParam("code_challenge_method", "S256"))
		http.Redirect(w, r, u, http.StatusFound)
	})
```

- ![显示登录](https://pic.imgdb.cn/item/6581656bc458853aeffec196.jpg)

- 进行登录授权

- 进行回调

http://localhost:9094/oauth2?code=N2ZJYWE5ZTATZWRKNI0ZYZUWLTHMNTMTNMJKNTAWMTC4OWQ5&state=xyz

client代码:

```go
	http.HandleFunc("/oauth2", func(w http.ResponseWriter, r *http.Request) {
		r.ParseForm()
		state := r.Form.Get("state")
		if state != "xyz" {
			http.Error(w, "State invalid", http.StatusBadRequest)
			return
		}
		code := r.Form.Get("code")
		if code == "" {
			http.Error(w, "Code not found", http.StatusBadRequest)
			return
		}
		token, err := config.Exchange(context.Background(), code, oauth2.SetAuthURLParam("code_verifier", "s256example"))
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}
		globalToken = token

		e := json.NewEncoder(w)
		e.SetIndent("", "  ")
		e.Encode(token)
	})
```

- 返回信息

```json
{
  "access_token": "MJHLODQ2OTUTNTJKNS0ZODQ1LTG3YZGTYMQ5NMZKYWQ4NZDK",
  "token_type": "Bearer",
  "refresh_token": "MZC3MMEYNJETNTU4ZI01MTHLLWFHYWETNGI2YMRJNDEXNZQ5",
  "expiry": "2023-12-19T19:42:33.281686958+08:00"
}
```

- 可以刷新token

http://localhost:9094/refresh

client代码:

```go
	http.HandleFunc("/refresh", func(w http.ResponseWriter, r *http.Request) {
		if globalToken == nil {
			http.Redirect(w, r, "/", http.StatusFound)
			return
		}

		globalToken.Expiry = time.Now()
		token, err := config.TokenSource(context.Background(), globalToken).Token()
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}

		globalToken = token
		e := json.NewEncoder(w)
		e.SetIndent("", "  ")
		e.Encode(token)
	})
```

返回:

```json
{
  "access_token": "ZWY3YTMXY2QTNJGXMI0ZMMQZLTGWOGMTMJU5YJZKN2JKZTAW",
  "token_type": "Bearer",
  "refresh_token": "OTC2YMRHNMMTZJA1NY01ODNJLWIWY2ITN2I5YMYXMJI4NZE1",
  "expiry": "2023-12-19T19:46:53.644729926+08:00"
}
```

#### 隐式授权流程（Implicit Grant）：

- 直接获取用户资源,比如用户id

http://localhost:9094/try

client代码:

```go
	http.HandleFunc("/try", func(w http.ResponseWriter, r *http.Request) {
		if globalToken == nil {
			http.Redirect(w, r, "/", http.StatusFound)
			return
		}

		resp, err := http.Get(fmt.Sprintf("%s/test?access_token=%s", authServerURL, globalToken.AccessToken))
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}
		defer resp.Body.Close()

		io.Copy(w, resp.Body)
	})
```

返回:

```json
{
  "client_id": "222222",
  "expires_in": 7158,
  "user_id": "test"
}
```

#### 密码模式（Resource Owner Password Credentials Grant）：

- 直接请求client设置好的接口: http://localhost:9094/pwd

client代码:

```go
	http.HandleFunc("/pwd", func(w http.ResponseWriter, r *http.Request) {
		token, err := config.PasswordCredentialsToken(context.Background(), "test", "test")
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}

		globalToken = token
		e := json.NewEncoder(w)
		e.SetIndent("", "  ")
		e.Encode(token)
	})
```

返回:

```json
{
  "access_token": "MZJHYZEYYTITNTMYOS0ZOWE4LWJIM2QTYJKWZWI5NMYZYZU5",
  "token_type": "Bearer",
  "refresh_token": "ZDVLNJJKZWETZJI0ZI01MZA1LWI3NTATM2RLOWE2ZJKXOTEZ",
  "expiry": "2023-12-19T19:48:41.6977701+08:00"
}
```

#### 客户端模式（Client Credentials Grant）：

- 请求: http://localhost:9094/client

client代码:

```go
	http.HandleFunc("/client", func(w http.ResponseWriter, r *http.Request) {
		cfg := clientcredentials.Config{
			ClientID:     config.ClientID,
			ClientSecret: config.ClientSecret,
			TokenURL:     config.Endpoint.TokenURL,
		}

		token, err := cfg.Token(context.Background())
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
			return
		}

		e := json.NewEncoder(w)
		e.SetIndent("", "  ")
		e.Encode(token)
	})
```

返回:

```json
{
  "access_token": "OTNLNWM0MTGTNTDLNS0ZNWQYLTK3NZATZTMXMDVKMDFIZMI5",
  "token_type": "Bearer",
  "expiry": "2023-12-19T19:55:04.94925986+08:00"
}
```

## 补充说明

确保按照实际需求配置OAuth2，特别关注安全性和令牌管理。

## 参考资料和进一步学习资源

- [OAuth 2.0官方规范](https://oauth.net/2/)
- [RFC 6749 - OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [Golang OAuth库示例](https://github.com/golang/oauth2)