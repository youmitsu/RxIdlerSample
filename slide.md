<!-- # 背景

こんにちは、私はAndroidアプリエンジニアをやっております。仕事の中で担当しているアプリのリグレッションテストを自動化しようという流れがあり、Espressoを使って実現することになりました。ただ、非同期処理が多く混じるテストの中で同期的にUIのテストをするのはつらみがあります。そこでここでは、非同期処理の際の便利なライブラリ「RxIder」についてここに至るまでの経緯も含めてご紹介したいと思います。 -->

### `RxJavaを使ったAndroidアプリで非同期処理のUIテストをやるならRxIdlerが便利`

---

### アジェンダ

1. UIテストとは

1. UIテストのつらみ

1. よくある解決策

1. RxIdlerの紹介

1. まとめ

---

### 軽く自己紹介

三堀 裕


- 株式会社エムティーアイ所属
- 社会人2年目
- Androidアプリエンジニア
- 外部LT初登壇(汗)


- Twitter: @1013Youmeee
- Qiita: https://qiita.com/youmeee
- GitHub: https://github.com/youmitsu

---

### 対象者

- AndroidでUIテストをやってみたいと思っている方
- RxJavaを使っているアプリでUIテストでの非同期処理に困っている方

---

### 前提

- RxJavaを使ってAPIコールなどの非同期処理をしているアプリ
- APIコールのモックなどはせず、実際のユーザがやるようなシナリオテスト的な観点のテスト

---

### 背景

- 仕事で担当するアプリにおいてシナリオテストを自動化しようと流れ
- ケースを書いていく中で非同期処理をうまく同期的にテストする仕組みが欲しいと感じるようになった
- Android TestNightに参加した時にDeNAのSWETの方にRxIdlerが良いというお話を聞いたので試してみたらいい感じだった

---

### UIテストとは

- 画面内に特定のUIコンポーネントがあるかどうか
- このボタンを押した後にこの画面が表示されるか

などの実際に実機で操作する時のアプリの挙動をテストすることができるものです。

---

### AndroidにおけるUIテスト

AndroidでUIテストを行うには、以下の二つの仕組みを使います

- InstrumentalTest
- Espresso

>>>

### `Android Instrumental Test`

UnitTestのようなJVM上で動作するテストと違い、Androidの実機でテストをする仕組み。
導入方法は以下を参照


https://developer.android.com/training/testing/ui-testing/espresso-testing

>>>

### `Espresso`

AndroidのUIテスト用のフレームワーク。

- リソースのidなどを使用したUIの取得（ViewMatcher）
- UIに対するアクション（タップ、スクロールなど）（ViewActions）
- アサーション(表示されているか、文字が正しいかなど)（ViewAssertion）

https://developer.android.com/training/testing/espresso/

---

### UIテストのつらみ

- APIリクエストなどの非同期処理後の画面反映を待たずに次のリソースの取得、アサーションに映ってしまいテストが失敗してしまう

---

### 具体例


```
    @Test
    fun testNotWaitFailed() {

        /**
         * APIリクエストなどの非同期処理
         */

        //非同期処理の終了を待たずにアサーションが走りテストが失敗する
        onView(allOf(withId(R.id.data_str), isDisplayed()))
          .check(matches(withText("completed")))
    }
```

---

### 主な解決策3つ

1. sleepを使って待つパターン
1. 反映されるまでリトライするラッパークラスを作るパターン
1. UIの状態で判断するパターン(IdlingResources)

[Idling Resourcesについてはこちらを参照](https://developer.android.com/training/testing/espresso/idling-resource)

---

どのパターンもそれぞれ辛い...

辛い理由はQiitaに載せます

---

### 本題

〜そこでRxIdler〜

---

### `RxIdler`の概要

- Square製のライブラリ(Jake)
- IdlingResourcesをラップしている
- RxJavaの非同期処理をしている別スレッド上の動きから、そのスレッドがアイドル状態かどうかで判断する
- 単純な宣言

https://github.com/square/RxIdler

---

### インストール

`build.gradle`に以下を記述

```
androidTestImplementation 'com.squareup.rx.idler:rx2-idler:0.9.0'
```

---

### コード例

以下のように
`@Before`が付いているメソッド内でRxJavaPluginsを使って、RxIdlerのスケジューラを定義するだけ

```
    @Before
    fun setUp() {
        //その他初期化処理

        //以下を記述(RxJavaのSchedulerタイプに応じて使うメソッドを変える)
        RxJavaPlugins.setInitIoSchedulerHandler(
                Rx2Idler.create("RxJava 2.x Io Scheduler")
        )
    }
```

---

### メリット・デメリット

---

### メリット

- テストがとても効率的に実行可能
- とても簡単
  - TestRunnerの@Beforeに書くだけで全てのRxJavaでの非同期処理にIdlingResourcesを適用してくれる
- プロダクトコードをいじらなくて済む

---

### デメリット・注意点
- RxJavaでしか使えない。
- たまに待ってくれない箇所もある。WebViewとか(全ての非同期処理これでなんとかなるわけではない)
- サンプルを作ろうとしたが、RxIdlerを使わなくても勝手に待ってくれることもあったりするので、もう少しRxIdlerで対応できるところ、できないところを明確にしていきたい

---

### まとめ

- RxIdlerを使うとRxJavaでの非同期処理をよしなにして待ってくれるので便利
- たまに待ってくれない箇所もあるため、一旦実行してみて落ちるようなあればsleepするなどの対策が必要かも？

---

### 参考

こちらの資料などを参考にさせていただきました

- [RxJavaのアプリをEspressoでテストする簡単な方法](https://speakerdeck.com/kesin11/rxjavafalseapuriwoespressodetesutosurujian-dan-nafang-fa)

- [Test apps on Android -Android Developer-](https://developer.android.com/training/testing/)
