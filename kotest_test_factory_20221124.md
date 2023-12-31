この記事は筆者の[ソロ Advent Calender 2022](https://qiita.com/advent-calendar/2022/panda) 13日目の記事です。

今回はKotestのTestFactoriesという機能が気になったので紹介していきます。

Kotestについて詳しく知りたい方は[こちら](https://qiita.com/JY8752/private/f0ffdc6795c57e9e9b63)の記事をどうぞ

# TestFactories
特定の入力に対して同じようなテストを再利用して実施したいケースなどがあったとき、KotestではTestFactoriesを使用して実現することができます。以下は公式ドキュメントの例です。

ListとVectorの2つの実装を持つIndexedSeqというインターフェースを以下のように作成した時

```kotlin
interface IndexedSeq<T> {

    // returns the size of t
    fun size(): Int

    // returns a new seq with t added
    fun add(t: T): IndexedSeq<T>

    // returns true if this seq contains t
    fun contains(t: T): Boolean
}
```

Listのテストを以下のように作成したとします。

```kotlin
class ListTest : WordSpec({

   val empty = List<Int>()

   "List" should {
      "increase size as elements are added" {
         empty.size() shouldBe 0
         val plus1 = empty.add(1)
         plus1.size() shouldBe 1
         val plus2 = plus1.add(2)
         plus2.size() shouldBe 2
      }
      "contain an element after it is added" {
         empty.contains(1) shouldBe false
         empty.add(1).contains(1) shouldBe true
         empty.add(1).contains(2) shouldBe false
      }
   }
})
```

この時、Vectorに対しても同じようなテストを実施したいと思うでしょう。この場合、このテストケースをテストセットとして以下のようなテストファクトリを作成することができます。

```kotlin
fun <T> indexedSeqTests(name: String, empty: IndexedSeq<T>) = wordSpec {
   name should {
      "increase size as elements are added" {
         empty.size() shouldBe 0
         val plus1 = empty.add(1)
         plus1.size() shouldBe 1
         val plus2 = plus1.add(2)
         plus2.size() shouldBe 2
      }
      "contain an element after it is added" {
         empty.contains(1) shouldBe false
         empty.add(1).contains(1) shouldBe true
         empty.add(1).contains(2) shouldBe false
      }
   }
}
```

これをListとVectorに対して適用すると以下のようになります。

```kotlin
class IndexedSeqTestSuite : WordSpec({
   include(indexedSeqTests("vector"), Vector())
   include(indexedSeqTests("list"), List())
})
```

# Repositoryテストのテストファクトリを作成する
このテストファクトリを使得そうなケースは何かないかなと考えていてRepositoryクラスのテストに使えないかなと思ったので実装してみます。

## 共通インターフェイスを実装する
以下のようなレコードの保存とデータ取得するメソッドを持つインターフェイスを作成します。

```kotlin:CrudRepository
interface CrudRepository<T> {
    fun save(entity: T): T
    fun findById(id: Long): T?
}
```

## 実装クラスを作成する
上記で作成したCrudRepositoryの実装クラスを作成します。今回はUserRepositoryとGroupRepositoryを作成します。
実装は手抜きしてますが実際はDBと接続する想定です。

```kotlin:UserRepository.kt
interface UserRepository<T> : CrudRepository<T>

data class UserEntity(val id: Long? = null, val name: String)

class UserRepositoryImpl : UserRepository<UserEntity> {
    override fun findById(id: Long): UserEntity? {
        return UserEntity(id, "user")
    }

    override fun save(entity: UserEntity): UserEntity {
        return entity
    }
}
```

```kotlin:GroupRepository.kt
interface GroupRepository<T> : CrudRepository<T>

data class GroupEntity(val id: Long? = null, val name: String)

class GroupRepositoryImpl : GroupRepository<GroupEntity> {
    override fun findById(id: Long): GroupEntity? {
        return GroupEntity(id, "GroupA")
    }

    override fun save(entity: GroupEntity): GroupEntity {
        return entity
    }
}
```

## テストファクトリを作成する
作成したUserRepositoryとGroupRepositoryのテストを実行するためのテストファクトリを以下のように作成します。

```kotlin:RepositoryTest.kt
fun <T> repositoryTests(name: String, repository: CrudRepository<T>, entity: T) = stringSpec {
    name {
        val saved = repository.save(entity)
        val find = repository.findById(1)
        saved shouldBe find
    }
}
```

## テストスイートを実行する
作成したテストファクトリを使用しUserRepositoryとGroupRepositoryのテストを実行する。

```kotlin:RepositoryTest.kt
internal class RepositoryTestSuite : StringSpec({
    val userRepository = UserRepositoryImpl()
    val groupRepository = GroupRepositoryImpl()
    include(repositoryTests("UserRepository: save and find", userRepository, UserEntity(1, "user")))
    include(repositoryTests("GroupRepositoryTest: save and find", groupRepository, GroupEntity(1, "GroupA")))
})
```

```
./gradlew test --tests com.example.testfactory.RepositoryTestSuite

BUILD SUCCESSFUL in 3s
6 actionable tasks: 1 executed, 5 up-to-date
```

OK

# まとめ
Advent Calenderに書くネタを探してKotestのドキュメントを読み返していてたまたまこのTestFactoriesの機能を見つけ、うまく使えたら便利そう！と思い、なんとなく共通のインターフェイスを持っていて、同じようなテストを書くのってRepositoryの接続テストで使えそうだなーということで試してみました。良い感じに活用できたと思いますが共通のインターフェイスを実装する必要があるのが実際のプロジェクトではもう少し考える必要があるかなと思いました。

他に何かいいユースケースがあれば使ってみたいなと思います。今回は以上です！