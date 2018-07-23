<!-- # 背景

こんにちは、私はAndroidアプリエンジニアをやっております。仕事の中で担当しているアプリのリグレッションテストを自動化しようという流れがあり、Espressoを使って実現することになりました。ただ、非同期処理が多く混じるテストの中で同期的にUIのテストをするのはつらみがあります。そこでここでは、非同期処理の際の便利なライブラリ「RxIder」についてここに至るまでの経緯も含めてご紹介したいと思います。 -->

RxJavaを使ったAndroidアプリで非同期処理のUIテストをやるならRxIdlerが便利

---

## アジェンダ

1. UIテストとは
1. UIテストのつらみ
1. よくある解決策
1. RxIdlerの紹介
1. まとめ

---

## 軽く自己紹介

三堀 裕


- 株式会社エムティーアイ所属
- 社会人2年目
- Androidアプリ担当


- Twitter: @1013Youmeee
- Qiita: https://qiita.com/youmeee

---

## 対象者

- AndroidでUIテストをやってみたいと思っている方
- RxJavaを使っているアプリでUIテストでの非同期処理に困っている方

---

## 背景

- 仕事で担当するアプリにおいてシナリオテストを自動化しようと流れ
- ケースを書いていく中で非同期処理をうまく同期的にテストする仕組みが欲しいと感じるようになった
- Android TestNightに参加した時にDeNAのSWETの方にRxIdlerが良いというお話を聞いたので試してみたらいい感じだった

---

## 前提

- RxJavaを使ってAPIコールなどの非同期処理をしているアプリ
- APIコールのモックなどはせず、シナリオテスト的な観点でテストを行う

---

## UIテストとは

- 画面内に特定のUIコンポーネントがあるかどうか
- このボタンを押した後にこの画面が表示されるか

などの実際に実機で操作する時のアプリの挙動をテストすることができるものです。

---

## AndroidにおけるUIテスト

AndroidでUIテストを行うには、以下の二つの仕組みを使います

- InstrumentalTest
- Espresso

>>>

## `Android Instrumental Test`

UnitTestのようなJVM上で動作するテストと違い、Androidの実機でテストをする仕組み。
導入方法は以下を参照


https://developer.android.com/training/testing/ui-testing/espresso-testing

>>>

## `Espresso`

AndroidのUIテスト用のフレームワーク。

- リソースのidなどを使用したUIの取得 => ViewMatcher
- UIに対するアクション（タップ、スクロールなど） => ViewActions
- アサーション(表示されているか、文字が正しいかなど) => ViewAssertion


https://developer.android.com/training/testing/espresso/

---

## UIテストのつらみ

- APIリクエストなどの非同期処理の完了を待たずに次のリソースの取得、アサーションに映ってしまいテストが失敗してしまう

---

## 具体例


```
    @Test
    fun testNotWaitFailed() {

        /**
         * APIリクエストなどの非同期処理
         */

        //非同期処理の終了を待たずにアサーションが走りテストが失敗する
        assertListText("first")
        assertListText("second")
        assertListText("third")
        assertListText("fourth")
        assertListText("fifth")
    }
```

---

## 主な解決策3つ

1. だいたいこれくらい待てばいけるっしょパターン
1. ダメだったらリトライするパターン
1. UIの状態で判断するパターン

---

## 解決策①

〜だいたいこれくらい待てばいけるっしょパターン〜


- 非同期処理が終わるであろう時間までsleepする

---

## コード例

```

@Test
fun testBySleep() {
    /**
     * APIリクエストなどの非同期処理
     */

    //だいたいこれくらい待てば表示されてるでしょっていう時間待つ(13秒待つ)
    Thread.sleep(13000)

    //待ったあとには表示されているはずなので、テストが通る
    assertListText("first")
    assertListText("second")
    ...
}

```

---

## メリット・デメリット

---

### メリット
- 簡単に実装できる

---

### デメリット
- sleepしている時間よりも早く処理が終わった場合、余分なsleep時間が生まれてしまう。
- 全体のテスト実行時間が長くなる(テストファームなどを使用している時に料金に関わってくる)

---

## 解決策②

〜ダメだったらリトライするパターン〜


- アサーションが成功するまでリトライする
- 実装例は割愛

---

## メリット・デメリット

---

### メリット
- なし

---

### デメリット
- sleepと同様で、実行時間が長くなる

---

## 解決策③

〜UIの状態で判断するパターン〜

- EspressoIdlingResourcesと呼ばれるものを使ってやるパターン
コンポーネントの状態からそのUIがアイドリング状態かを判断し、アイドリングの状態だったら待つみたいなことができる
(例： RecyclerViewのアダプターのsizeが0じゃなかったらアイドルとみなすなど)

- https://developer.android.com/training/testing/espresso/idling-resource

- 実装例は割愛

---

## メリット・デメリット

---

### メリット
- 実行時間の短縮ができ、効率的なテストができる

---

### デメリット
- 「非同期処理の完了通知->画面反映」の画面反映の部分にフォーカスしているため少し冗長(ブラックボックス的)
- プロダクトコードを修正しなければならなかったりする
- 自分でIdlingResourcesのクラスを自作しなければならなかったりと色々面倒

---

## 本題

〜そこでRxIdler〜

---

## `RxIdler`の概要

- Square製のライブラリ(Jake)
- IdlingResourcesをラップしている
- RxJavaの非同期処理をしている別スレッド上の動きから、そのスレッドがアイドル状態かどうかで判断する
- 単純な宣言

https://github.com/square/RxIdler

---

## インストール

`build.gradle`に以下を記述

```
androidTestImplementation 'com.squareup.rx.idler:rx2-idler:0.9.0'
```

---

## コード例

以下のように
`@Before`が付いているメソッド内でcomputationSchedulerHandlerにRxIdlerのスケジューラを定義するだけ

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

## メリット・デメリット

---

### メリット

- とても簡単
  - TestRunnerの@Beforeに書くだけで全てのRxJavaでの非同期処理にIdlingResourcesを適用してくれる
- プロダクトコードをいじらなくて済む

---

### デメリット
- RxJavaでしか使えない。

---

### 注意点
- アプリ起動後すぐの非同期処理などは待ってくれずそのまま落ちたりする(原因わからず)

---

## サンプル

サンプルコードは以下にあげています

https://github.com/youmitsu/RxIdlerTestApp

---

## まとめ

- RxIdlerを使うとRxJavaでの非同期処理をよしなに待ってくれるので便利
- たまに待ってくれない箇所もあるため、一旦実行してみて落ちるようなあればsleepするなどの対策が必要かも？
