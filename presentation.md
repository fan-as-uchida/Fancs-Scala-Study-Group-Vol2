# Arrows

F@N Communications, Inc.

石川 裕樹 (Yuki Ishikawa)

---

## 自己紹介

### 石川 裕樹 (Yuki Ishikawa)

- ファンコミ入社4年目

- nend, nex8, viidleの配信システムをScalaで開発

- ScalaCache, sttp, refinedとかのメンテナンス

- バイク好き🏍

- Twitter: @rider_yi

- GitHub: rider-yi

---

## traneio/arrows

- https://github.com/traneio/arrows

- 4月19日に正式公開されたばかりのライブラリ

![announce](./images/announce.png)


## Arrows

高パフォーマンスなArrowとTask型を提供する


FutureをArrow/Taskに置き換えることでスループットを高めることができる

---

## 実装が2種類ある

基本的にどちらか1つを選択して使う

- arrows-stdlib
- arrows-twitter


## arrows-stdlib

`scala.concurrent.Future`をベースにしている

```scala
libraryDependencies ++= Seq(
  "io.trane" %% "arrows-stdlib" % "0.1.19"
)
```

```scala
import arrows.stdlib._
```


## arrows-twitter

twitter/utilの`Future`をベースにしている

(finch, Finagleユーザ向け)

```scala
libraryDependencies ++= Seq(
  "io.trane" %% "arrows-twitter" % "0.1.19"
)
```

```scala
import arrows.twitter._
```


両方とも似たような動きをするが、インターフェースがそれぞれのFutureに合わせてある

```scala
import scala.concurrent.Future
import arrows.stdlib._

// Scala Future
Future.successful("foo")
Task.successful("foo")
```

```scala
import com.twitter.util.Future
import arrows.twitter._

// Twitter Future
Future.value("foo")
Task.value("foo")
```

Note:
- 今日はarrows-twitterを使って説明したいと思います



使っているライブラリやフレームワークに合わせて選ぶと良い

---

## Arrow[T, U]

- 引数を1つ取り1つ値を返す再利用可能な計算
- runするまで計算は行われない(lazy)
- andThenで結合可能


### Arrowの作り方

```scala
import arrows.twitter._

// Identity arrowは受け取った値を返すだけ
val identityArrow: Arrow[Int, Int] = Arrow[Int]
```

```scala
val stringify: Arrow[Int, String]
  = Arrow[Int].map(_.toString)
```

Note:
- Arrow.applyとmapを使うことでArrowの計算を定義できる
- Identity arrowはそのままでは役に立たないが、新しいArrowを作るためのスターティングポイントとして役に立つ



### Arrowの実行

```scala
val stringify: Arrow[Int, String]
  = Arrow[Int].map(_.toString)

// runするとFutureを返す(=計算が始まる)
val fut1: Future[String] = stringify.run(123)
```

```scala
// Arrowはただの計算なので再利用が可能
val fut2: Future[String] = stringify.run(321)
```


### andThenでArrow同士を結合

```scala
val callServiceA: Arrow[Int, Int] = Arrow[Int].map(_ + 1)
val callServiceB: Arrow[Int, Int] = Arrow[Int].map(_ * 2)
val callServiceC: Arrow[Int, Int] = Arrow[Int].map(_ - 1)
```

```scala
val myArrow: Arrow[Int, Unit] =
  callServiceA
    .andThen(callServiceB)
    .andThen(callServiceC)

val fut = myArrow.run(5)
Await.result(fut) // 11
```

Note:
- flatMapの説明をする前にもう一つの`Task`という型を紹介したいと思います

---

## Task[T]

- cats-effect, Scalaz, Monixなどの`IO`,`Task`と同等
- 値を1つ返す計算(引数はない)
- 入力のないArrow


### TaskはArrowの型エイリアス

```scala
package arrows

package object twitter {
  type Task[+T] = Arrow[Unit, T]
}
```


### Task.apply and Task.value

```scala
import arrows.twitter._

val random: Task[Int] = Task {
  scala.util.Random.nextInt(100)
}

val fut: Future[Int] = task.run()
```

```scala
// 既に計算された値にはvalueを使う
val one = Task.value(1)
```

Note:
- catsのpure(Applicative)とdelay(Sync)



### Task.async

```scala
def performSomeAsyncSideEffect: Future[Unit] = ???

Task.async(performSomeAsyncSideEffect())
```

```scala
// Bad practice
val fut = performSomeAsyncSideEffect()
Task.async(fut)
```

Note:
- asyncはcall-by-nameでパラメータを受け取るので、実際にはperformSomeAsyncSideEffectは評価されない

---

### Arrow.flatMap

```scala
final def flatMap[V](f: U => Task[V]): Arrow[T, V] =
  this match {
    case a: Exception[_] => a.cast
    case a               => FlatMap(a, f)
  }
```

- ArrowのflatMapは`U => Task[V]`を引数に取る
- TaskはMonadだけどArrowはMonadではない


### flatMapは処理の分岐などに利用

```scala
import arrows.twitter._

// サービスの呼び出し
val callServiceA = Arrow[Int].map(_ * 2)
val callServiceB = Arrow[Int].map(_ + 1)

val myArrow =
  callServiceA.flatMap {
     case 0 => Task.value(0)
     case i => callServiceB(i) // Arrowに値を与えるとTaskになる
  }

val result: Future[Int] = myArrow.run(1)
```


### for comprehension

```scala
val callServiceA: Arrow[Int, Int] = Arrow[Int].map(_ + 1)
val callServiceB: Arrow[Int, Int] = Arrow[Int].map(_ * 2)
val callServiceC: Arrow[Int, Int] = Arrow[Int].map(_ - 1)

val myArrow: Arrow[Int, Int] = 
  for {
    a <- callServiceA
    b <- callServiceB(a)
    c <- callServiceC(b)
  } yield c

// ただ順番にArrowを実行すれば良い場合はandThenを使った方が効率的
val myArrow: Arrow[Int, Int] = 
  callServiceA.andThen(callServiceB).andThen(callServiceC)
```

---

## 利用例

IPを国に変換するAPIをarrowsとfinchで書いてみる


```scala
final case class Country(name: String, code: String)

trait Database {
  def lookup(ip: InetAddress): Country
}
```


### Arrowを作る

```scala
// キャッシュされた最新のデータベースを取得
val getInMemoryDb: Task[Database] = ???

// IPアドレスとDBから国を判定
val lookup: Arrow[(InetAddress, Database), Country] =
  Arrow[(InetAddress, Database)].map {
    case (ip, db) => db.lookup(ip)
  }

// IPアドレスを国に変換するArrowを作る
val ip2Country: Arrow[InetAddress, Output[Country]] =
  Arrow[InetAddress]
    .join(getInMemoryDb)
    .andThen(lookup)
    .map(Ok)
```


```scala
// /country?ip=8.8.8.8
val endpoint: Endpoint[Country] =
  get("country" :: param[InetAddress]("ip")) {
    (ip: InetAddress) =>
    ip2Country.run(ip) // Future[Output[Country]]
  }
```

---

## cats-effect integration

- 現状ない😢
- サブモジュールで提供予定らしい(現在開発中?)
- MonadError, Syncくらいなら簡単につくれる

---

## ベンチマーク結果


### Scala Future x Arrows Stdlib (Async)

#### Throughput (ops/s)

![async-thrpt-scala](./images/async-thrpt-scala.png)

Note:
- arrow: 25,385
- scala: 2,817



### Scala Future x Arrows Stdlib (Sync)

#### Throughput (ops/s)

![sync-thrpt-scala](./images/sync-thrpt-scala.png)

Note:
- arrow: 1,306,246
- scala: 9,804



### Twitter Future x Arrows Twitter (Async)

#### Throughput (ops/s)

![async-thrpt-twitter](./images/async-thrpt-twitter.png)


### Twitter Future x Arrows Twitter (Sync)

#### Throughput (ops/s)

![sync-thrpt-twitter](./images/sync-thrpt-twitter.png)


## Other libraries x Arrows (Async)

#### Throughput (ops/s)

![async-thrpt-others](./images/async-thrpt-others.png)


### Other libraries x Arrows (Sync)

#### Throughput (ops/s)

![sync-thrpt-others](./images/sync-thrpt-others.png)


- Arrowは他のソリューションより高パフォーマンスな傾向
- Taskはちょっとだけ良い

---

## `Future`からの移行

- `Future`と`Task`はI/Fが似ているのでとりあえず全部Task化
  - READMEにScalafixのルールがある

- 後で可能な部分は全てArrow化する

```scala
def callService(x: Int): Task[Unit] = Task { ... }
// ↓
val callService: Arrow[Int, Unit] = Arrow[Int].map { ... }
```

- See: https://github.com/traneio/arrows#migrating-from-future

---

## ありがとうございました
