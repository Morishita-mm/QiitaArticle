---
title: Notionで最強のタスク計測ツール作ってみた
tags:
  - タスク管理
  - エンジニア
  - Notion
  - '#効率化'
private: false
updated_at: '2024-11-27T18:39:31+09:00'
id: ab765c8e3269cf27c9da
organization_url_name: null
slide: false
ignorePublish: false
---

:::note warn
Qiita で記事を書くのは初めてなので、至らぬところがあってもお目溢しいただけると幸いです 🙇
:::

## はじめに

皆さんは普段のタスク管理をどのようにして行なっていますか？
私は今まで、スマホの **DotTimer** というアプリを使っていました

タスク管理とタスク計測を同時に行ってくれて、タグづけしたタスクごとに、集計をしてくれるアプリで非常に気に入っていました

しかし勉強が紙面上から PC 上に変わるにつれて、スマホでいちいち操作するのが面倒になってきました

そこで、Notion を使えばそれが再現できて、さらに他の情報とも紐付けられるのではないかと考えて作成したものが今回紹介したいテンプレートになります

### 外観

![header.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/5d6d3746-6403-b582-7120-87e9f519c91e.png)
ヘッダーです。今日のタスク、よく使うリンク、時計と天気を共有ブロックで囲んでいます。
新しいページを作成した時に、上につけることでどのページでも同じヘッダーを持たせることができるようになっています。
![calender.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/165797dd-75bb-4cfb-0afc-5e8bf4554c85.png)
タスクをカレンダー形式で表示したものと、ポモドーロタイマーをつけています
![mytask.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/73bb89ac-23d1-8e45-bda1-824c7e16a80a.png)
メインとなる機能を実現している部分です。使い方は後ほど説明します。
![taglibrary.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/f8054a50-85ce-86c4-010a-4e37e625e37c.png)
あらゆるタグを管理するデータベースです。これがあるおかげで後程紹介する詳細な集計を実現できています。
![スクリーンショット 2024-11-26 7.55.02.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/71a4d31f-9139-fb00-ba87-3c50a7764b95.png)
今後の目標と、１週間の目標を管理するページです。使い方は後程説明します。

## 使い方

### 1. タグの設定

TagLibrary というタグを作成するデータベースに自分のタスクを管理する上で必要だと思うタグを追加していきます。
分割の粒度は個人の好みの問題なのであまり気にせず欲しいと思ったタグをどんどん追加していらなくなったら消していきましょう！

### 2. 目標の設定

ここでいう目標とは 3 種類あります

<dl>
    <dt>１週間の目標</dt>
    <dd><b>WeekTask</b>ページの今週の目標部分のデータベースに追加します（最後の写真の右側部分）<br />
    直接ここに追加してもいいですし、今後の目標のデータベースの<b>追加ボタン</b>を押すことで今後の目標のうち今週達成したい目標を今週の目標に移すことができます。<br />
    ここには<b>1週間後にできるようになっていたいこと</b>を考えて追加します。<br />（例：マークダウン形式の書き方を理解する）
    <details>
        <summary>動作の様子</summary>

![addWeek.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/3dd0461c-75bb-7720-9380-92dc19d3cc64.gif)

</details>
</dd>
<dt>長期目標</dt>
<dd>ポモドーロタイマー下の<b>長期目標追加</b>ボタンを押して追加します。<br />
ここには試験の予定や、○ 月 △ 日までに〇〇資格を取りたいなど、<b>中・長期の目標</b>を追加します。<br />（例：今年中に Qiita で記事を 10 本書く）
<details>
<summary>動作の様子</summary>

![addLong.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/ae7b87aa-39ca-3fda-fbf9-412a42c080f4.gif)</details>

</dd>

<dt>今後の目標</dt>
<dd><b>WeekTask</b>ページの今後の目標部分のデータベースに追加します（最後の写真の左側部分）<br />
ここには生活していて、<b>今後達成したい目標</b>ができた時に追加します。<br />（例：Qiita で記事を書けるようになる
<details>
<summary>動作の様子</summary>

![addFuture.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/0be2f1ea-cc3b-8856-0a1d-20ffc15c9634.gif)

</details>
</dd>
</dl>

これらは毎日更新するようなものではなく、１週間や 1 ヶ月に一度目標を整理する時に追加します

### 3. タスクの追加

タスクは以下の 5 つの方法で追加することができます。

1. タスクの作成と実行を同時に行う
2. `新規タスク作成`ボタンでタスクを作成する
3. テンプレートからタスクを作成する
4. １週間の目標からタスクを追加する
5. 長期目標からタスクを追加する
   ![addTask.jpeg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/82347b01-6845-1bb1-e10c-4c919e2c7a8d.jpeg)

それぞれの方法でタスクを追加すると、`今日`ビューの今日以降のタスク一覧にタスクが追加されます

### 4. タスクの開始と終了

タスクの開始方法は 2 種類あります

1. タスクの追加と同時に開始をする方法
2. タスクのプロパティである`開始`ボタンを押す方法

タスクの終了方法も 2 種類あります

1. `TIMER`欄の`BREAK`ボタンを押す方法
2. タスクのプロパティである`完了`もしくは`中断`ボタンを押す方法

以下が動作例になります
![startTask.gif](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/371529/7500fc71-19b3-cc7e-6176-30760d794f76.gif)

- `BREAK`ボタンを押すと自動的に`break`というタスクが作成され、休憩時間を計測し始めます
- `完了`ボタンを押すと、進行状況が`進行中`から`完了`に変化し、タスクの実施時間が記録されます
- `中断`ボタンを押すと、進行状況が`進行中`から`完了`に変化し、タスクの実施時間が記録され、`No`プロパティがインクリメントされたコピーが作成されます

状況に応じてそれぞれのボタンを使い分けてください

以上が基本的な使い方になります<br />
集計や TagLibrary などの他の部分に関してはタスク計測ツールとはあまり関係ないので今回は割愛します<br />

## おわり

今回は初めての Qiita 記事投稿ということもあり、すごく無駄な説明や逆に言葉足らずになってしまった部分が多々あるなーと思っています<br >
もしも興味を持っていただけたら今回説明したテンプレートを Web で公開しておくのでみなさんの Notion ライフの足しにしてもらえたら幸いです<br />
記事を読んでいただきありがとうございました！<br/>

今回紹介したテンプレートの URL：[最強のタスク管理ツール](https://pastoral-dream-ed6.notion.site/Task_Template-1424b664efd980f8a648c624ae92fbc6?pvs=4)

### 参考

【保存版】Notion リレーション・ロールアップ機能を 30 分でマスター

https://www.youtube.com/watch?v=AasG9Eak8Tw&t=1500s&pp=ygUGTm90aW9u

これで仕事が楽になる！Notion でタスク漏れゼロを実現した方法

https://www.youtube.com/watch?v=wd9GToczc9g&pp=ygUWbm90aW9uIOOCv-OCueOCr-euoeeQhg%3D%3D
