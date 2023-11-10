この記事は筆者の[ソロ Advent Calender 2022](https://qiita.com/advent-calendar/2022/panda) 11日目の記事です。

今回はKotlin製のテスティングフレームワークであるKotestのデータ駆動テスト(Data Driven Testing)について紹介いたします！

# Kotestとは
Javaでは古くからJUnitでテストを書かれることが多く、KotlinでもJUnitでテストを書くことは可能ですが純正Kotlinで実装されたテスティングフレームワークがKotestです。詳しくは公式がかなり読みやすく作られているので公式を見ていただければと思いますが大きな特徴としては10種類のSpecと呼ばれるクラスが用意されており好きなSpecを選択してテストを書くことができます。Specはそれぞれ様々な言語、テスティングフレームワークの影響を受けて作られており他言語からKotlinを始めた人は自分の母国語のテストSpecを選択できるでしょう。

他にも実験的な機能も含めて多くの機能、アサーション、エクステンションが用意されており筆者は大変気に入っております。その中でも、データ駆動テストを書くための機能が用意されておりこれが大変書き心地が良いのでkotestのデータ駆動テストを普及させるべく記事を書くことにしました。

詳しくは公式を

https://kotest.io/

# setup
適当なKotlinプロジェクトを用意して、依存関係を追加します。

```kotlin:build.gradle.kts
    // kotest
    val kotest_version: String = "5.5.4"
    testImplementation("io.kotest:kotest-runner-junit5-jvm:$kotest_version")
    testImplementation("io.kotest:kotest-framework-datatest:$kotest_version")
```

Kotestを使うだけであれば上記の依存関係のみで大丈夫ですが、データ駆動テストを書く場合は別モジュールで用意されているため上記2つを追加します。

# 基本的はKotestの使い方
追加できたら適当なテストクラスを用意し、一応動作を確認。

```kotlin:Test.kt
internal class Test : StringSpec({
    "test" {
        1 + 1 shouldBe 2
    }
})
```

上記は一番シンプルに書けるStringSpecによるテストの書き方です。テストクラスに好きなSpecクラスを継承させコンストラクタ引数に処理ブロックを渡すことでテストを書いていきます。上記テストは以下のようにinitブロックにしても書くことはできます。

```kotlin:Test.kt
internal class Test : StringSpec() {
    init {
        "test" {
            1 + 1 shouldBe 2
        }
    }
}
```

好きな方でいいかなと思いますが、筆者はネストが深くなるとテストが読みづらくなると思っているので前者の方を好みます。

アサーションにはKotestで用意されている```shouldBe```を使用していますが、JUnitとかでいうとassertThatとほぼ同じと思っていただいて大丈夫です。大体はこれでいける

# データ駆動テスト
データ駆動テストは入力の値と出力結果の値のパターンを作成して、テストを実行する手法のことです。データ駆動なのでこのデータパターンをまず作成することが主な仕事で、アサーション部分は関数呼ぶだけみたいなのが理想だと思っています。例を見た方がわかりやすいかと思うので簡単なデータ駆動テストを作成すると以下のようになります。

```kotlin:DataDrivenTest.kt
enum class Operator {
    ADD, SUBTRACTION, MULTIPLICATION, DIVIDE
}

fun calculate(num1: Int, num2: Int, operator: Operator): Int {
    return when (operator) {
        Operator.ADD -> num1 + num2
        Operator.SUBTRACTION -> num1 - num2
        Operator.MULTIPLICATION -> num1 * num2
        else -> num1 / num2
    }
}

internal class DataDrivenTest : FunSpec({
    context("test calculate") {
        data class TestPattern(val num1: Int, val num2: Int, val operator: Operator, val result: Int)
        withData(
            TestPattern(1, 1, Operator.ADD, 2),
            TestPattern(3, 1, Operator.SUBTRACTION, 2),
            TestPattern(2, 3, Operator.MULTIPLICATION, 6),
            TestPattern(10, 5, Operator.DIVIDE, 2),
        ) { (num1, num2, operator, result) ->
            calculate(num1, num2, operator) shouldBe result
        }
    }
})
```

引数で指定された数値と演算子で計算した結果を返す関数をテストしています。まず、データ駆動テストをKotestで書く場合はネストできるSpecである必要があるのでStringSpec以外のSpecを選択する必要があります。なんでもいいんですが筆者はFunSpecを選択することが多いです。

FunSpecの場合はcontextブロックの中に入力と期待する出力を格納するデータテーブルをデータクラスで表現します。今回はcalculateメソッドの3つの引とその期待する戻り値を格納するよう設定します。

withDataメソッドの引数に定義したデータクラスを使用しデータパターンを与えます。テストしたいパターンが全て書けたら最後にデータクラスの全ての引数を引数にしたラムダの中で処理を実行します。

これを実行すると以下のような結果が得られます。

//画像

どうでしょう、一つ一つテストメソッドを作成するよりもはるかに簡潔に書けています。実際のプロジェクトではこんな綺麗に入力と出力が副作用なく定義された関数はなかなかできないでしょうが、このような入力と出力のパターンが複数考えられる時には積極的にデータ駆動テストを採用するようにしています。

ところで、上記のcaluculateメソッドは完璧でしょうか？テストパターンを一つ追加してみます。

```diff_kotlin

internal class DataDrivenTest : FunSpec({
    context("test calculate") {
        data class TestPattern(val num1: Int, val num2: Int, val operator: Operator, val result: Number)
        withData(
            TestPattern(1, 1, Operator.ADD, 2),
            TestPattern(3, 1, Operator.SUBTRACTION, 2),
            TestPattern(2, 3, Operator.MULTIPLICATION, 6),
            TestPattern(10, 5, Operator.DIVIDE, 2),
+            TestPattern(5, 0, Operator.DIVIDE, 0)
        ) { (num1, num2, operator, result) ->
            calculate(num1, num2, operator) shouldBe result
        }
    }
})
```

これを実行すると0除算で```ArithmeticException```が発生してしまいます。このままエラーでもいい気がしますし、別のエラーでラップしたあげた方がいいかもしれませんし0徐算の時はnullで返してあげた方がいいかもしれません。何が言いたいかというとデータ駆動テストを書くことで「このパターンどうなるんだろう？」とか「あれこの入力の時はどうしよう？」みたいなパターン網羅が容易に書けるので自分の実装したコードに自信が持てるということです。

# テストケースの命名
デフォルトの場合データクラスのtoString()がテスト名として表示されます。この表示を変えたい場合にはいくつか方法がありますがまずWithDataTestNameをデータクラスに継承することで表示を変更することができます。

```kotlin:DataDrivenTest.kt
internal class DataDrivenTest : FunSpec({
    context("test calculate") {
        data class TestPattern(val num1: Int, val num2: Int, val operator: Operator, val result: Int) : WithDataTestName {
            override fun dataTestName(): String = "when num1: $num1 num2: $num2 operator: $operator, result is $result"
        }
        withData(
            TestPattern(1, 1, Operator.ADD, 2),
            TestPattern(3, 1, Operator.SUBTRACTION, 2),
            TestPattern(2, 3, Operator.MULTIPLICATION, 6),
            TestPattern(10, 5, Operator.DIVIDE, 2),
        ) { (num1, num2, operator, result) ->
            calculate(num1, num2, operator) shouldBe result
        }
    }
})
```

表示はこのようになります。

//画像

これは以下のような書き方もできます。

```kotlin:DataDrivenTest.kt
internal class DataDrivenTest : FunSpec({
    context("test calculate") {
        data class TestPattern(val num1: Int, val num2: Int, val operator: Operator, val result: Int)
        withData(
            nameFn = { "when num1: ${it.num1} num2: ${it.num2} operator: ${it.operator}, result is ${it.result}" },
            TestPattern(1, 1, Operator.ADD, 2),
            TestPattern(3, 1, Operator.SUBTRACTION, 2),
            TestPattern(2, 3, Operator.MULTIPLICATION, 6),
            TestPattern(10, 5, Operator.DIVIDE, 2)
        ) { (num1, num2, operator, result) ->
            calculate(num1, num2, operator) shouldBe result
        }
    }
})
```

また、mapを使用しkeyにテスト名を指定することもできます。

```kotlin:DataDrivenTest.kt
internal class DataDrivenTest : FunSpec({
    context("test calculate") {
        data class TestPattern(val num1: Int, val num2: Int, val operator: Operator, val result: Int)
        withData(
            mapOf(
                "1 + 1 = 2" to TestPattern(1, 1, Operator.ADD, 2),
                "3 - 1 = 2" to TestPattern(3, 1, Operator.SUBTRACTION, 2),
                "2 x 3 = 6" to TestPattern(2, 3, Operator.MULTIPLICATION, 6),
                "10 / 5 = 2" to TestPattern(10, 5, Operator.DIVIDE, 2)
            )
        ) { (num1, num2, operator, result) ->
            calculate(num1, num2, operator) shouldBe result
        }
    }
})
```

