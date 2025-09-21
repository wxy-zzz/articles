---
title: "AI でデータ分析！Databricks Genie の活用事例"
emoji: "🧞‍♂️"
type: "tech"
topics: ["Databricks", "Genie", "IVRy", "AIブログリレー", "zennfes2025free"]
published: true
published_at: 2025-09-24 12:00
publication_name: "ivry"
---

本記事は [#IVRy_AIブログリレー](https://x.com/hashtag/IVRy_AI%E3%83%96%E3%83%AD%E3%82%B0%E3%83%AA%E3%83%AC%E3%83%BC?src=hashtag_click&f=live) の 9 月 24 日（15 日目）の記事です。
記事一覧は「[IVRy AIブログリレー全記事まとめ](https://note.com/ivry/n/na4f8be95a13b)」をご覧ください。

----

## はじめに

株式会社IVRy のアナリティクスエンジニアの [wada](https://note.com/wxy_zzz/n/nd1e905d15842) です。
IVRy では 2025 年 7 月に社内データ基盤を BigQuery から Databricks に移し、Databricks 上で新たに分析基盤を構築しています。本記事では、Databricks の AI/BI ツールである [Genie](https://www.databricks.com/jp/product/business-intelligence/ai-bi-genie) を分析活動に活用するために行っていることを紹介します。

### 忙しい人のための本記事まとめ

記事が長くなりそうなので、先に簡単にまとめます。

- Genie にダッシュボードのドリルダウンを任せているが、いい感じ
- 自然言語での分析を進めるためには、結局データモデリングが大事
- かっちりしたデータモデリングをいかに崩せるのかが今後の課題になる予感

また今回の記事の内容は、Findy さん主催のイベント「[AI時代の意思決定を支える 各社のデータ基盤Lunch Talk](https://findy.connpass.com/event/368980/)」でも詳しく紹介しようと思うので、ぜひ聞いてください！今回記事内にまとめきれていないこともできる限り紹介しようと思っています！

https://findy.connpass.com/event/368980/

----

本編再開。

## 背景情報

事例紹介に関連するツールなどの背景情報を紹介します。

### プロダクトについて

弊社プロダクト「[アイブリー](https://ivry.jp)」は電話業務を効率化する対話型音声 AI SaaS です。
プロダクトの詳細は割愛しますが、本記事では例としてプロダクトの売上情報を扱いたいので、基本的なビジネスモデルを背景情報として共有します。大まかに、アイブリーからお客様に対して発番した電話番号に対して
- 各種機能を使うためのサブスクリプション料金
- 各種機能を実際に使ったときの従量料金

の二種類の料金をいただく形態となっています。

なお、アイブリーは toB プロダクトであるため、全国的に展開している toC プロダクトや大量のトラフィックを取り扱う広告システムなどで扱うデータと比べると、データの規模自体はやや小さめです。アイブリーを使った電話の着信は [2025 年 6 月時点では累計 5,000 万件](https://ivry.jp/pr/u2x0671a8ymk/) で、取り扱うデータの種類（音声データなど）による違いはあるものの、単純なレコード数としてのデータのオーダー量は膨大というわけではなく、それを前提としたデータモデリングを行っています。

### Databricks

IVRy では [Databricks](https://www.databricks.com/jp) を社内データ基盤として採用し、2025 年 7 月から本格利用を開始しています。Databricks を含めたデータのエコシステム全体のアーキテクチャは [こちらのページ](https://findy-tools.io/companies/ivry/90/76) をご覧ください。

#### metric view

:::message
本記事執筆段階では、metric view は public preview な機能です
:::

[metric view](https://docs.databricks.com/aws/en/metric-views/) は Databricks の機能で、一言で言えば「セマンティックレイヤーを物理 View として実現するもの」です。以下のようなクエリで定義をします（SQL の中に yaml が出てくるの、若干不思議な感覚になりますw）

```sql
create view sample_metric_view (
    month     comment '利用月',
    phone_id  comment '電話番号に対応する ID',
    used_plan comment 'phone_id がその月に利用していた料金プラン名',
    revenue   comment '総収益',
    arpa      comment 'ARPA. phone_id 単位での平均収益額',
)
with metrics
language yaml
comment 'metric view のサンプルです'
as $$
version: 0.1
source: |-
  -- view の元になるクエリ
  select *
  from sample_revenues
    inner join sample_dimensions using (phone_id)
dimensions:
  - name: month
    expr: "`month`"
  - name: phone_id
    expr: "`phone_id`"
  - name: used_plan
    expr: "`used_plan`"
measures:
  - name: revenue
    expr: sum(revenue)
  - name: arpa
    expr: try_divide(sum(revenue), count(distinct phone_id))
$$;
```

このように定義して metric view を作ると、例えば

```sql
select
    month,
    measure(revenue) as revenue
from sample_metric_view
group by all
```

とすれば、月ごとの総収益が集計できますし、

```sql
select
    trunc(month, 'year') as year,
    measure(arpa) as arpa
from sample_metric_view
where used_plan = 'standard'
group by all
```

とすれば、スタンダードプランを利用しているお客様に限定した上で、年間の ARPA を計算することもできちゃいます。

事前に定義を決めた上で用途に応じた切り口で集計することが可能なので、様々なシーンで活躍しています。また、Databricks 内の [ダッシュボード機能](https://docs.databricks.com/aws/ja/dashboards/) のデータソースとして参照することができるので、アドホックな集計でも定常的に利用するダッシュボードでも同一の定義を参照できます。
何かと発生しがちな、定義の差異による集計結果のズレを解消するのに一役買っています。

#### Genie

![](https://www.databricks.com/sites/default/files/2025-06/1-conversational-user-experience.png)
*Databricks 公式サイトからイメージ画像の引用*

[Genie](https://www.databricks.com/jp/product/business-intelligence/ai-bi-genie) は Databricks に標準で搭載されている AI/BI ツールで、自然言語で Databricks 内のデータに対して問い合わせをすることができます。自然言語に対して SQL を生成し、その結果を返したり、ダッシュボード機能と同等のグラフを作成したりすることも可能です。また、[ダッシュボードに対しても Genie を使うことができ](https://docs.databricks.com/aws/ja/dashboards/genie-spaces)（※ 執筆時点では public preview）、ダッシュボード内の特定のグラフなどに対して自然言語で質問をすることも可能です。

## Databricks 移行に伴う期待

背景情報が長くなりましたが（Databricks の機能について書くの楽しい）、Databricks への移行を機に取り組みたかった改善点は主に「データ基盤上での社内メンバーの自走性向上」という点です。

Genie を使ってデータ分析・抽出ができるようになれば、データサークル（サークルは [IVRy における部署みたいなもの](https://note.com/ryogaskywalker/n/n773420326251)）に対して依頼を出す前に、ビジネスメンバーが自分自身で解決できるシーンが増えるかもしれません。

また、ダッシュボードの作成という観点でも工数の削減が期待できます。何度もやりとりをしてダッシュボードを作り込むのではなく、ざっくりとチャートを作った上で詳細は Genie に聞いて深堀りをする、という進め方ができると今よりも早く運用開始できる可能性があります。また Genie に指示を出して SQL を生成できるようになれば、ビジネスメンバーがダウンロードするためだけの raw data の表を貼り付ける、みたいな本来の用途から外れたダッシュボードの使い方も回避できる可能性があります。データ基盤の中で分析活動が完結するようになると、活動履歴から更にデータ基盤を改善することにも期待が持てます。

結果としては
- Genie を使ったデータ分析・抽出: △
    - 任意のデータに対して任意のクエリを処理するのは難しい
    - 整備されたデータに対してはほぼエラーはない
- ダッシュボードに対する Genie での操作: ○
    - ダッシュボードで表現できるであろうことは Genie で可能
    - SQL の生成も適切

という状態です。

## Genie の検証と結果

Genie でどれぐらいのことができるかは、インターンの三宅さんに協力してもらって進めました（三宅さんの記事は [こちら](https://zenn.dev/ivry/articles/9e65e9cea6c8c1)）

それを踏まえた私の所感は以下の通りです。
- Genie 賢い
    - テーブルやカラムのコメントや、データの中身そのものを考慮してくれている
        - 例えば「業種がサービス業であるもの」と指定すると、SQL を作る時には実際の値に合うように `industry = 'サービス業（他に分類されないもの）'` と解釈してくれるなど曖昧な指示に強く、ビジネスメンバーにも薦めやすい。
- 歴史的経緯などを含めた raw data の正確な定義の理解は難しい
    - 人間にも難しいので、AI が悪いわけではないが……
        - 人がハマりどころとして既知のものに Genie がハマると、Genie が悪いように見える
        - 人にとって未知のものは人がレビューできない（結果使わない判断になってしまう）

## Genie の現在の利用方針

検証結果を受けて、任意の raw data を入力として自然言語で何かを解決するのはかなりハードルが高いと判断しました。メタデータを充実させれば改善される可能性はありますが、使うかどうかわからないものについて整備を進めるのはコストパフォーマンスが非常に悪いので、データの入り口側（raw data 側）ではなく出口側（ダッシュボードなど）に Genie を導入する方が良いのではないかと考えました。

出口側の素になる metric view に適切なコメントをつけておくと、それに対する Genie の動作はかなり安定し、問い合わせに対しての間違いは十分に低減されます。その metric view をダッシュボードにも使うようにすれば、自ずとダッシュボードの Genie に対する問い合わせも安定することになります。当初の期待通り、基本的なチャートだけダッシュボードに置いておいて色々な観点で可視化し直すのを一旦 Genie に任せる運用が可能なので、少なくともダッシュボードを作成する側としては確認することが大幅に削減されたように思います。利用者側から見ると Genie が挟まる分不便さは増しているかもしれないので、そこは今後の課題として調整していく領域になります（が、ちょっとずつ違う条件の似たようなチャートを量産してメンテ不能な状態に陥るよりは今の方が遥かに健全だと個人的には思っています）

![](/images/ivry-ai-blog-relay-databricks-genie-20250924/drilldown.png)
*同じシェイプのチャートでも Genie に指示を出して可視化パターンを変えられる*

一方で、metric view に定義されていないことは全て回答不能になってしまうというデメリットはあります。ただ、Genie に対する問い合わせの履歴は確認することができるので、回答に失敗しているものやよく質問されているものに関しては、ある程度ピックアップして対応することが可能です。

![](/images/ivry-ai-blog-relay-databricks-genie-20250924/history.png)
*Genie に対する問い合わせの内容や、それに対するリアクションも確認できる*

## Genie 活用にあたっての学び

**結局データモデリング。**

metric view のような定義のかっちりとしたものに対して Genie はかなり安定して動作することはわかったので、IVRy の使い方における分析のボトルネックは metric view の品質になります。

例えば「背景情報」の項目でサンプルとして挙げた metric view は、アイブリーのビジネスモデルである「サブスクリプション料金」「従量料金」に分解することができません。

:::details 【再掲】サンプルとして掲載した metric view 定義
```sql
create view sample_metric_view (
    month     comment '利用月',
    phone_id  comment '電話番号に対応する ID',
    used_plan comment 'phone_id がその月に利用していた料金プラン名',
    revenue   comment '総収益',
    arpa      comment 'ARPA. phone_id 単位での平均収益額',
)
with metrics
language yaml
comment 'metric view のサンプルです'
as $$
version: 0.1
source: |-
  -- view の元になるクエリ
  select *
  from sample_revenues
    inner join sample_dimensions using (phone_id)
dimensions:
  - name: month
    expr: "`month`"
  - name: phone_id
    expr: "`phone_id`"
  - name: used_plan
    expr: "`used_plan`"
measures:
  - name: revenue
    expr: sum(revenue)
  - name: arpa
    expr: try_divide(sum(revenue), count(distinct phone_id))
$$;
```
:::

また、機能別の料金でまとめることもできませんし、契約法人単位での平均料金（ARPU）を集計することもできません。これは単に metric view に定義されていないから、という理由です。

これらに対応するためには、最初から必要な項目をバラッバラにしたデータモデルを事前に作っておく必要があります。例えば下記のようにファクトデータとディメンションデータを用意した上で、それらを結合した metric view を作れば、上記で挙げた「サブスクリプションと従量の分解」「機能別」「法人単位での平均収益額」などが集計可能になります。

ファクトテーブル（イメージ）

| month      | phone_id | used_plan | charge_type     | used_function | revenue | ... |
| :--        |      --: | :--       | :--             | :--           |     --: | --: |
| 2025-08-01 |        1 | standard  | サブスクリプション | プラン         |    2980 | ... |
| 2025-08-01 |        1 | standard  | サブスクリプション | AI 音声対話    |       0 | ... |
| 2025-08-01 |        1 | standard  | 従量課金         | IVR           |     600 | ... |
| 2025-08-01 |        1 | standard  | 従量課金         | AI 音声対話     |    300 | ... |
| ...        |      ... | ...       | ...             | ...           |     ... | ... |

ディメンションテーブル（イメージ / 簡略化のため、法人情報も連結済み）

| phone_id | company_name | size | industry              | use_started_at | ... |
|      --: | :--          | :--  | :--                   |            --: | --: |
|        1 | 株式会社IVRy  | SMB  | インターネットサービス業  |     2019-03-05 | ... |
|        2 | 株式会社IVRy  | SMB  | インターネットサービス業  |     2020-10-20 | ... |
|      ... | ...          | ...  | ...                   |            ... | ... |

アドホックな集計をする場合に、一旦全部バラバラにする、というプロセスを踏むことはあまりないので、アドホック分析ばかりしていると正確な回答ができる Genie の構築は遅れるのではないかと思います。社内の分析用途に耐えうる中間データモデルを整備する仕事は今後も続きそうです。

Genie を利用する際は対象にするテーブルを事前に指定する必要があるため、細かくデータを散在させるように整備するよりは一元的に管理する方が効率が良さそうです。以前 [IVRy で開催したイベント](https://ivry.connpass.com/event/346397/) で [Sansan株式会社 金髙さんが仰っていました](https://speakerdeck.com/sansan_randd/250325-ci-data?slide=19)が、まさに

> 「デカい」dimension は全てを制す

というのを実感しています。ディメンションだけでなく、指標も一元的に管理する方が良さそうです。

## 今後の課題

### Genie の真価を、さらに拓く

前述の通り、今の IVRy では metric view で定義をかっちりとさせて、間違いが起きにくい状態にした上で Genie を運用しています。

社内で運用しているダッシュボードの Genie に対する問い合わせを見ている感じだと、Genie はかなり良い感じ対応してくれており、metric view で定義されていること以上のことが本来はできそうな手応えがあります。定義されていないデータは返さない、という方針はこれはこれで間違っていないとは思うのですが、metric view がある程度出揃ってきたら、今度は「如何にして定義されていない質問を捌けるようにしていくか」が課題になりそうです。

ちなみに、現段階で具体的な方法は全く思い浮かんでいませんw

### その他の課題

#### 1 テーブルで表現しにくい指標への対応

収益に関するよくある指標として、収益額の昨月比や昨年比があるのですが、（少なくとも）これらの指標を metric view として綺麗にモデリングできていません。今は苦し紛れに、1 レコードに対応する数字を格納し（`revenue` カラムの隣に `revenue__prev_month` や `revenue__prev_year` を追加する）、その上で metric view で
```yaml
measures:
  - name: revenue__mom
    expr: try_divide(sum(revenue), sum(revenue__prev_month))
  - name: revenue__yoy
    expr: try_divide(sum(revenue), sum(revenue__prev_year))
```

みたいな定義をすることで暫定的に実現していますが、このやり方だと

```sql
select
    trunc(month, 'year') as year,
    measure(revenue__mom) as revenue__mom
from sample_metric_view
group by all
```

とクエリを書くことで、年単位の昨月比という謎の数字が作れてしまうんですよね……

うーん、データモデリング難しすぎる……

#### コード管理

実は metric view はまだ dbt に対応していないため、dbt に完結してデータモデルを管理するという状態に至れていません。人手による温かみのあるオペレーションで運用しています。[dbt Labs との調整が必要](https://github.com/databricks/dbt-databricks/issues/1106#issuecomment-3129173795)、とのことらしいので、首を長くして待っています！手動オペレーションで複雑なことをしたくないので、今は metric view は本当に最後の出口にしか使っていないのですが metric view を多段に組めればまた違ったデータモデリングができそうな気がしています。
[Snowflake](https://www.snowflake.com/ja/) とのコラボだけでなく Databricks も何卒お願いします！！！！

## おわりに

ここまでお読みいただき、ありがとうございました。

### まとめ

本記事では、IVRy においてデータ分析に Genie を活用するために行っていることを紹介させていただきました。改めて本記事のまとめです。

- ダッシュボードのドリルダウンは Genie で問題なく対応できており、有用
- 結局データモデリングが大事
- 厳密なデータモデリングと AI による柔軟性のバランスが今後の課題になりそう

今後も、社内のデータ分析活動をより推進するために分析基盤を整えていきます！

### お知らせ

IVRy では、イベントや最新ニュース、募集ポジションなどの情報をお届けするカジュアルなつながり「キャリア登録」をご用意しています。IVRy に少しでもご興味をお持ちいただいた方は、以下のリンクからぜひご登録ください。

https://ivry-jp.notion.site/209eea80adae800483a9d6b239281f1b

また、データサークルでは新卒・インターンも募集しています。詳細はこちらの記事をご覧ください。

https://note.com/ivry/n/nc80c02e6250f

実際にインターンでどんなことをしているのかは、インターン生の公開している記事をご覧ください。

- [自然言語でのデータ分析を当たり前に：IVRyのDatabricks Genie運用の現在地（インターン視点）](https://zenn.dev/ivry/articles/9e65e9cea6c8c1)
- [データウェアハウスでアプリ開発!? 新機能Databricks Appsで社内アプリを作った話](https://zenn.dev/ivry/articles/databricks-apps-devlog)
