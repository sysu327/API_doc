## 简单Web服务与客户端开发实战报告

服务计算第9次作业
18342133 张泽健
<!-- TOC -->

- [简单Web服务与客户端开发实战报告](#简单web服务与客户端开发实战报告)
    - [介绍](#介绍)
    - [API的设计介绍](#api的设计介绍)
    - [职责：后端开发](#职责后端开发)
    - [开发过程](#开发过程)
        - [Article部分的api实现](#article部分的api实现)
        - [用户认证](#用户认证)

<!-- /TOC -->
### 介绍

Simple Blog即为极简博客，实现的功能有
- 注册
- 登陆
- 查看博客
- 查看评论
- 发表评论

这次作业中我们实现的一个要求是实现前后端的分离，因此API的设计是必不可少的。

### API的设计介绍

我们的博客一共有6个API，分别是：

- GET /article/{id}/comments
- GET /article/{id}
- GET /articles
- POST /article/{id}/comments
- POST /user/login
- POST /user/register

是三个GET和POST类型的API

API对应的代码：

```

func ArticleIdCommentsGet(w http.ResponseWriter, r *http.Request) {
}

func ArticleIdGet(w http.ResponseWriter, r *http.Request) {
}

func ArticlesGet(w http.ResponseWriter, r *http.Request) {
}

func ArticleIdCommentPost(w http.ResponseWriter, r *http.Request) {
}

func UserLoginPost(w http.ResponseWriter, r *http.Request) {
}

func UserRegisterPost(w http.ResponseWriter, r *http.Request) {
}

```

### 职责：后端开发

这次的作业我主要负责后端开发，具体工作包括：

- 建立数据库，以及实现对数据的CRUD
- 处理来自前端的请求并返回正确的相应
- 利用jwt对用户进行认证

后端代码的结构：

![](1.png)

```
xxxxxxxx:~/Desktop/Server$ tree
.
├── LICENSE
├── README.md
├── api
│   └── swagger.yaml
├── dal
│   ├── db
│   │   ├── Blog.db
│   │   ├── db.go
│   │   ├── db_test.go
│   │   └── sampleBlog
│   └── model
│       ├── model_article.go
│       ├── model_comment.go
│       ├── model_tag.go
│       └── model_user.go
├── go
│   ├── README.md
│   ├── api_article.go
│   ├── api_user.go
│   ├── jwt.go
│   ├── logger.go
│   ├── response.go
│   └── routers.go
└── main.go

```

- 其中dal即数据访问层
- dal/db存放的是数据库
- dal/model定义了四个结构体
- go目录下是路由、API的响应、API的实现、jwt认证模块等

### 开发过程

同样负责后端的另一位同学 @ciaoSora 颜府 已经在他的报告当中对db model go 包的开发进行了阐述，因此我主要在此说一个我还负责的一些其他内容：Article部分，和User的用户认证

#### Article部分的api实现

实现思路：

- 读取Request参数
- 读取到参数后利用db.go的函数CRUD
- 将结果写入Response


#### 用户认证

流程如下:

1. 客户端使用用户名跟密码请求登录
2. 服务端收到请求，去验证用户名与密码
3. 验证成功后，服务端会签发一个 Token，再把这个 Token 发送给客户端
4. 客户端收到 Token 以后可以把它存储起来，比如放在 Cookie 里或者 Local Storage 里
5. 客户端每次向服务端请求资源的时候需要带着服务端签发的 Token
6. 服务端收到请求，然后去验证客户端请求里面带着的 Token，如果验证成功，就向客户端返回请求的数据


这个功能在jwt.go中得以实现：
```
// SecretKey : secret key for jwt
const SecretKey = "123qwe"

func ValidateToken(w http.ResponseWriter, r *http.Request) (*jwt.Token, bool) {
	token, err := request.ParseFromRequest(r, request.AuthorizationHeaderExtractor,
		func(token *jwt.Token) (interface{}, error) {
			return []byte(SecretKey), nil
		})

	if err != nil {
		w.WriteHeader(http.StatusUnauthorized)
		fmt.Fprint(w, "Unauthorized access to this resource")
		return token, false
	}

	if !token.Valid {
		w.WriteHeader(http.StatusUnauthorized)
		fmt.Fprint(w, "token is invalid")
		return token, false
	}

	return token, true
}

func SignToken(userName string) (string, error) {

	token := jwt.New(jwt.SigningMethodHS256)
	claims := make(jwt.MapClaims)
	claims["exp"] = time.Now().Add(time.Hour * time.Duration(1)).Unix()
	claims["iat"] = time.Now().Unix()
	claims["name"] = userName
	token.Claims = claims
	return token.SignedString([]byte(SecretKey))
}

``` 

