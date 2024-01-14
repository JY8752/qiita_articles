---
title: 'Java製のバリデーションライブラリYAVI(ヤバイ)を使ってみた[API導入編]'
tags:
  - Java
  - Kotlin
  - SpringBoot
  - AdventCalendar2022
  - YAVI
private: false
updated_at: '2022-12-15T14:46:52+09:00'
id: 568eee783ba1bc2bcb6c
organization_url_name: null
slide: false
ignorePublish: false
---
この記事は筆者の[ソロ Advent Calendar 2022](https://qiita.com/advent-calendar/2022/panda) 10日目の記事です。

前回までにJava製のバリデーションライブラリであるYAVI(ヤバイ)の使い方を紹介しましたが、せっかくなのでもう少し実践的に簡単なAPIを作成してみましたのでその紹介です。

[Java製のバリデーションライブラリYAVI(ヤバイ)を使ってみた](https://qiita.com/JY8752/items/e72d228eb49a42c3cbb0)
[Java製のバリデーションライブラリYAVI(ヤバイ)を使ってみた[応用編]](https://qiita.com/JY8752/items/d59c76c2574d2782088b)
[Java製のバリデーションライブラリYAVI(ヤバイ)を使ってみた[API導入編]](https://qiita.com/JY8752/items/568eee783ba1bc2bcb6c) <- 今ここ

今回までの成果物はこちらです

https://github.com/JY8752/YAVI-demo

# 準備
今回はSpringを使用しますのでSpring Initializerでプロジェクトを作成します。
全体的な構成は以下のようなものを作っていこうと思います。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/551753/a936f365-cb70-5bdc-24e1-da7975191c21.png)


# data層(Entity, Repository)
ちょっと手抜きでインターフェイスの定義のみします。

```kotlin:UserEntity.kt
data class UserEntity(
    val id: Long? = null,
    val name: String,
    val email: String,
    val age: Int,
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)
```

```kotlin:UserRepository.kt
interface UserRepository {
    fun create(name: String, email: String, age: Int): UserEntity
    fun update(name: String?, email: String?, age: Int?): UserEntity
    fun get(id: Long): UserEntity?
    fun delete(id: Long)
}
```

# service層
サービス層は以下のような感じ。

```kotlin:UserService.kt
@Service
class UserService(
    private val userRepository: UserRepository
) {
    fun create(request: User): User {
        return this.userRepository.create(request.name, request.name, request.age).toModel()
    }

    private fun UserEntity.toModel() = User(this.name, this.email, this.age)
}
```

# controller層
本題のapplicationレイヤーの実装。今回はAPIの実装なのでUser作成のパラメーターをYAVIを使用しバリデーションしてみます。

まず、リクエストパラメーターのマッピングクラスの定義と合わせてバリデーター定義を以下のように実装します。

```kotlin:UserForm.kt
data class CreateUserRequest(val name: String?, val email: String?, val age: Int?) {
    companion object {
        private val nameValidator = StringValidatorBuilder
            .of("name") { it.notBlank().greaterThan(0).lessThanOrEqual(50) }
            .build()
            .compose<CreateUserRequest> { it.name }

        private val emailValidator = StringValidatorBuilder
            .of("email") { it.notBlank().lessThanOrEqual(100) }
            .build()
            .compose<CreateUserRequest> { it.email }

        private val ageValidator = IntegerValidatorBuilder
            .of("age") { it.greaterThanOrEqual(0).lessThanOrEqual(100) }
            .build()
            .compose<CreateUserRequest> { it.age }

        val validator = ArgumentsValidators.combine(nameValidator, emailValidator, ageValidator)
            .apply { name, email, age -> User(name!!, email!!, age!!) }
    }
}
```
nameとemailとageのパラメーターでUserを作成するためのデータクラスです。Kotlin特有の話ですがフィールドをnonNullで定義してしまうとパラメーターがnullで指定されない状態で来るとバリデーションの前にBad Requestで弾かれてしまいます。それでもまあいいのですがレスポンスを完全にコントロールできた方がいいかなと思ったので意図的にnullableで定義しています。

validatorの実装をどこに置くかは少し迷ったのですがバリデーション対象クラスにcompanion objectで定義しました。前回の記事でやったように各フィールドのバリデーターを定義し最終的に結合してます。validatorはcompose()を使用しているのでCreateUserRequestを直接渡すことでバリデーションの実施とドメインモデルへの変換まで実行できるように作成しています。

呼び出し元のControllerはこんな感じ

```kotlin:UserController.kt
@RestController
@RequestMapping("/user")
class UserController(
    private val userService: UserService
) {
    @PostMapping("/create")
    fun create(@RequestBody request: CreateUserRequest): Response {
        val validated = CreateUserRequest.validator.validate(request)
        return if (validated.isValid) {
            val created = this.userService.create(validated.value())
            Response.success(created)
        } else {
            Response.error(validated.errors())
        }
    }
}

class Response private constructor(val success: Boolean, val payload: Any? = null, val errors: List<String> = emptyList()) {
    companion object {
        private val logger = LoggerFactory.getLogger(this::class.java)
        fun success(payload: Any) = Response(true, payload)
        fun error(ex: Exception): Response {
            this.logger.error(ex.message, ex)
            return Response(false, errors = listOf(ex.message ?: "unknown error..."))
        }
        fun error(errors: ConstraintViolations): Response {
            val messages = errors.map { it.message() }
            this.logger.error("errors: {}", messages)
            return Response(false, errors = messages)
        }
    }
}
```

実際にレスポンスを確認してみる。

正常系
```
% curl -X POST -H "Content-Type: application/json" localhost:8080/user/create -d '{"name": "user", "email": "test@test.com", "age": 32}' | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   140    0    87  100    53   2446   1490 --:--:-- --:--:-- --:--:--  5600
{
  "success": true,
  "payload": {
    "name": "user",
    "email": "test@test.com",
    "age": 32
  },
  "errors": []
}
```

異常系
```
curl -X POST -H "Content-Type: application/json" localhost:8080/user/create -d '{"name": "", "email": "", "age": -1}' | jq   % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   250    0   214  100    36   5903    993 --:--:-- --:--:-- --:--:--  8064
{
  "success": false,
  "payload": null,
  "errors": [
    "\"name\" must not be blank",
    "The size of \"name\" must be greater than 0. The given size is 0",
    "\"email\" must not be blank",
    "\"age\" must be greater than or equal to 0"
  ]
}
```

# まとめ
今回はバリデーションライブラリであるYAVIを実プロジェクトを想定して導入するとどんな感じになるかを簡単なAPIを作成して試してみました。今回はエラー応答にvalidatorのmessageをそのまま配列で指定しましたが、他にもいろいろ応用できそうなのでプロジェクトの要件に合わせて変更できるかなと思います。ただ実装していて

- validatorの適切な置き場について
- フレームワークを使うならばInterceptorでエラー応答をいい感じにまとめられるかもしれない。(SpringであればControllerAdviceなどで)

のような点が気になったので今後使う機会があればもう少し考えてみようと思います。

以上！
