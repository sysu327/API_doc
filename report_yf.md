# 简单web服务与客户端开发报告
这是服务计算第9次作业

## 功能
- 注册
- 登陆
- 查看博客
- 查看评论
- 发表评论

## 工作
```shell
xxxxxxxx:~/Desktop/Server$ tree
.
├── api
│   └── swagger.yaml
├── dal
│   ├── db
│   │   ├── Blog.db
│   │   ├── db.go
│   │   ├── db_test.go
│   │   └── sampleBlog
│   │       ├── 0.html
│   │       ├── 1.html
│   │       ├── 2.html
│   │       └── WriteBlog.go
│   └── model
│       ├── model_article.go
│       ├── model_comment.go
│       ├── model_tag.go
│       └── model_user.go
├── go
│   ├── api_article.go
│   ├── api_user.go
│   ├── jwt.go
│   ├── logger.go
│   ├── README.md
│   ├── response.go
│   └── routers.go
├── LICENSE
├── main.go
└── README.md

6 directories, 22 files
```

- db包存放数据库（BoltDB）相关代码
- model包存放数据结构
- go包实现api

## 开发过程
### db包
db实现了数据库（BoltDB），是以键值对的方式使用的。此处实现了User相关的代码。

GetUser函数的功能为查询用户信息，输入username，输出User结构。若失败则把Username和Password设为空字符串。

```go
func GetUser(username string) model.User {
	db, err := bolt.Open(GetDBPATH(), 0600, nil)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	user := model.User{
		Username: "",
		Password: "",
	}

	err = db.View(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte("user"))
		if b != nil {
			data := b.Get([]byte(username))
			if data != nil {
				err := json.Unmarshal(data, &user)
				if err != nil {
					log.Fatal(err)
				}
			}
		}
		return nil
	})
	if err != nil {
		log.Fatal(err)
	}

	return user
}
```

PutUsers的功能为新增若干用户，输入一个用户slice，返回错误信息。

```go
func PutUsers(users []model.User) error {
	db, err := bolt.Open(GetDBPATH(), 0600, nil)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	err = db.Update(func(tx *bolt.Tx) error {
		b := tx.Bucket([]byte("user"))
		if b != nil {
			for i := 0; i < len(users); i++ {
				username := users[i].Username
				data, _ := json.Marshal(users[i])
				b.Put([]byte(username), data)
			}
		}
		return nil
	})

	if err != nil {
		return err
	}
	return nil
}
```

### model包
定义了一些struct，是该应用中必要的信息表示形式。

User是存储用户信息的数据结构，保存了用户用户名和密码

```go
type User struct {
	Username string `json:"username,omitempty"`

	Password string `json:"password,omitempty"`
}
```

Comment存放评论信息，包括用户、article的id、日期、内容

```go
type Comment struct {
	User string `json:"user,omitempty"`

	ArticleId int64 `json:"article_id,omitempty"`

	Date string `json:"date,omitempty"`

	Content string `json:"content,omitempty"`
}
```

### go包
该包实现了API

response.go用来处理所有Response，MyResponse结构存储响应信息，Options单独处理OPTIONS，Response用来发送响应

```go
type MyResponse struct {
	OkMessage    interface{} `json:"ok,omitempty"`
	ErrorMessage interface{} `json:"error,omitempty"`
}

func Options(w http.ResponseWriter, r *http.Request){
	w.Header().Set("Content-Type", "application/json; charset=UTF-8")
	w.Header().Set("Access-Control-Allow-Methods","PUT,POST,GET,DELETE,OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Access-Control-Allow-Origin, Access-Control-Allow-Credentials, Access-Control-Allow-Methods, Access-Control-Allow-Headers, Authorization, X-Requested-With")
	w.Header().Set("Access-Control-Allow-Origin", "*")
	w.WriteHeader(http.StatusOK)
}

func Response(response interface{}, w http.ResponseWriter, code int) {
	jsonData, jErr := json.Marshal(&response)

	if jErr != nil {
		log.Fatal(jErr.Error())
	}

	w.Header().Set("Content-Type", "application/json; charset=UTF-8")
	w.Header().Set("Access-Control-Allow-Methods","PUT,POST,GET,DELETE,OPTIONS")
	w.Header().Set("Access-Control-Allow-Headers", "Content-Type, Access-Control-Allow-Origin, Access-Control-Allow-Credentials, Access-Control-Allow-Methods, Access-Control-Allow-Headers, Authorization, X-Requested-With")
	w.Header().Set("Access-Control-Allow-Origin", "*")
	w.Write(jsonData)
	w.WriteHeader(code)
}
```

api_user.go实现了User相关的API，我负责的是
- POST /article/{id}/comments
- POST /user/login
- POST /user/register

首先是login和register，这二者比较类似，以login为例

```go
func UserLoginPost(w http.ResponseWriter, r *http.Request) {
	db.Init()
	var user model.User
	err := json.NewDecoder(r.Body).Decode(&user)

	if err != nil {
		Response(MyResponse{
			nil,
			"parameter error",
		}, w, http.StatusBadRequest)
		return
	}

	check := db.GetUser(user.Username)
	if check.Username != user.Username || check.Password != user.Password {
		Response(MyResponse{
			nil,
			"username or password error",
		}, w, http.StatusBadRequest)
		return
	}

	tokenString, err := SignToken(user.Username)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintln(w, "Error while signing the token")
		tokenString = "signing token error"
	}

	Response(MyResponse{
		tokenString,
		nil,
	}, w, http.StatusOK)
}
```

CommentPost也差不多，就是步骤相对多了一点，要先验证token，更新User，获得article的id，生成日期，将评论加到数据库里，最后返回评论数据。
