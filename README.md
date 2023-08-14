# login-example-go-part10の目標
- ユーザーのログイン処理を実装する！

前回はJWTの作成をできるようにしました。
今回は実際のログイン処理を実装したいと思います。

# 全体の流れの確認
- 仮登録 /auth/register/initial
    - クライアントからemail, passwordを受け取る
    - email宛に本人確認トークンを送信する
- 本登録 /auth/register/complete
    - クライアントからemailと本人確認トークンを受け取る
    - ユーザーの本登録を行う
- ログイン /auth/login
    - クライアントからemail, passwordを受け取る
    - 認証トークンとしてJWTを返す
- ユーザー情報の取得 /restricted/user/me
    - クライアントからJWTを受け取る
    - ユーザー情報を返す

全体としてこんな感じでした。
ログインにはどんな機能が必要でしょうか

# 必要な機能を考えよう！

箇条書きしてみます。

- クライアントから受け取ったemail, passwordのバリデーション
- emailを使ってDBからユーザー情報の取得
- ユーザーがアクティブじゃないならエラーを返す
- ユーザーのパスワード検証
- JWTを作成する
- レスポンスにJWTを含める

ざっと書き出すとこんな感じです。
ただ、このまま１つのファイルに書き出すとどこに何があるかわからなくなるので役割でまとめてみます。

| パッケージ | 役割 | 機能 |
|:-----------|:------------|:------------|
| Repository       | DBとのやりとり        | ・emailを使ってDBからユーザー情報の取得|
| Usecase     | ログイン処理を行う	 | ・ユーザーがアクティブじゃないならエラーを返す<br/>・ユーザーのパスワード検証<br/>・JWTを作成する|
| Handler       | リクエストボディの取得レスポンスの作成| ・クライアントから受け取ったemail, passwordのバリデーション<br/>・レスポンスにJWTを含める|

こんな感じで実装していきます。
ではやっていきましょう！

## Repository

「emailを使ってDBからのユーザー情報の取得」はすでにできていますね

## Usecase
- ユーザーがアクティブじゃないならエラーを返す
- ユーザーのパスワード検証
- JWTを作成する

が必要な機能でしたね。

usecase/user_usecase.go
```
type IUserUsecase interface {
	PreRegister(ctx context.Context, email, pw string) (*entity.User, error)
	Activate(ctx context.Context, email, token string) error
+	Login(ctx context.Context, email, password string) ([]byte, error)
}

- func NewUserUsecase(ur repository.IUserRepository, mailer mail.IMailer) IUserUsecase {
-	return &userUsecase{ur: ur, mailer: mailer}
- }

+ func NewUserUsecase(ur repository.IUserRepository, mailer mail.IMailer, jwter auth.IJwtGenerator) IUserUsecase {
+	return &userUsecase{ur: ur, mailer: mailer, jwter: jwter}
+ }
```
usecase/user_usecase.go
```
func (uu *userUsecase) Login(ctx context.Context, email, password string) ([]byte, error) {
	// emailからユーザー情報を取得する
	u, err := uu.ur.GetByEmail(ctx, email)
	if err != nil {
		return nil, err
	}
	// ユーザーがアクティブでないならエラー
	if !u.IsActive() {
		return nil, errors.New("user inactive")
	}
	// ユーザーのパスワードを検証
	if err := u.Authenticate(password); err != nil {
		return nil, err
	}
	// ユーザー情報からJWTを作成
	tok, err := uu.jwter.GenerateJWT(u)
	if err != nil {
		return nil, err
	}
	return tok, nil
}
```

はい、これで Usecase はOKです。

## Handler
Handlerでは次の内容を実装します。

- クライアントから受け取ったemail, passwordのバリデーション
- レスポンスにJWTを含める

handler/user_handler.go
```
type IUserHandler interface {
	PreRegister(c echo.Context) error
	Activate(c echo.Context) error
+	Login(c echo.Context) error
}
```

handler/user_handler.go
```
func (h *userHandler) Login(c echo.Context) error {
	// リクエストボディを受け取るための構造体を作成
	rb := struct {
		Email    string `json:"email" validate:"required,email"`
		Password string `json:"password" validate:"required,gte=6,lte=20"`
	}{}

	// リクエストボディの中身をrbに書き込みます
	if err := c.Bind(&rb); err != nil {
		return err
	}
	// validateタグの内容通りかどうか検証します。
	if err := c.Validate(rb); err != nil {
		return err
	}

	// context.ContextをPreRegisterに渡す必要があるので、echo.Contextから取得します。
	ctx := c.Request().Context()

	tok, err := h.uu.Login(ctx, rb.Email, rb.Password)
	if err != nil {
		return err
	}

	// ログイン成功、としてJWTを返す
	return c.JSON(http.StatusOK, echo.Map{
		"access_token": string(tok),
	})
}
```

これでHandlerも準備できました！

## Router
最後にLoginをrouterに登録していきましょう

router.go
```
- func NewRouter(db *sqlx.DB, mailer mail.IMailer) *echo.Echo {
+ func NewRouter(db *sqlx.DB, mailer mail.IMailer, jwter *auth.JwtBuilder) *echo.Echo {
	e := echo.New()

	ur := repository.NewUserRepository(db)
-	uu := usecase.NewUserUsecase(ur, mailer, jwter)
+	uu := usecase.NewUserUsecase(ur, mailer, jwter)
	uh := handler.NewUserHandler(uu)

	a := e.Group("/api/auth")
	a.POST("/register/initial", uh.PreRegister)
	a.POST("/register/complete", uh.Activate)
+	a.POST("/login", uh.Login)

	return e
}
```

main.go
```
func main() {
	db, err := db.NewDB()
	if err != nil {
		fmt.Println(err)
		return
	}
	defer db.Close()

	mailer := mail.NewMailhogMailer()

+	jwter, err := auth.NewJwtBuilder()
+	if err != nil {
+		fmt.Println(err)
+		return
+	}

+	e := NewRouter(db, mailer, jwter)

	e.HTTPErrorHandler = customHTTPErrorHandler
	e.Validator = &CustomValidator{validator: validator.New()}

	e.Logger.Fatal(e.Start(":8000"))
}
```

以上でLoginが完成しました！

# 確認

では本当にJWTを返すのか確認してみましょう。

```
$ curl -XPOST localhost:8000/api/auth/login \
        -H 'Content-Type: application/json; charset=UTF-8' \
        -d '{"email": "test-user-1@example.xyz", "password": "foobar"}'
```

```
{"access_token":"
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
eyJleHAiOjE2ODg0NzQ2OTEsImlhdCI6MTY4ODM4ODI5MSwiaXNzIjoibG9naW4tZ28iLCJzdWIiOiJhY2Nlc3MtdG9rZW4iLCJ1c2VyX2lkIjoxMDAwMDR9.
egjNAhE701ErJns5uCL1GwpZba55Z_N1s_tQpZD2LUMWdCxbr5vDQN5wMkySGOdJEajIBlhsa_XK3w0DZr97LZw_getsmTuSMEQYVhpnum-IhocQ_fBKta2bghH7Xs43E7pPQ39ochMr5RC93pbtAoeFibkRiqUXHwyFlHhjTFO3AbqInb84_M2nB7omJy7edJsRjO1qX6rlEFH3KyTWjWUuQcGuQpUCg71Ooli9NvAOrvw95FftlLaXSOzWCkDOjMgtJyZeEQ5Y_G-z0Io9wAtNi0ivNyv4IbIDvgABDOzSUXFXSmKwPkCWVUaq666MBZsvt_Mwx6eX4zu4bETqnw
"}
```

ちゃんとaccess_tokenがかえってきました！
これでログインが完成です！

# まとめ
今回行ったことはこんな感じです。
- ログイン処理の実装