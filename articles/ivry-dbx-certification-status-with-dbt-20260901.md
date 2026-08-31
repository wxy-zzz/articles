---
title: "dbt で認定済みタグを管理し、Databricks 上のデータの信頼性を可視化する"
emoji: "✅️"
type: "tech"
topics: ["Databricks", "dbt", "IVRy",]
published: true
published_at: 2026-09-01 12:00
publication_name: "ivry"
---

株式会社 IVRy のアナリティクスエンジニアの [wada](https://note.com/wxy_zzz/n/nd1e905d15842) です。
小ネタ記事ですが、[Databricks](https://www.databricks.com/jp) 上でデータの信頼性を示すための認定済みタグ（`Certification status system tag`）を dbt で管理してみた話です。

## 課題背景

[Data + AI Summit 2026](https://www.databricks.com/dataaisummit) での発表で、Databricks の AI 機能は [Genie ファミリー](https://docs.databricks.com/aws/ja/genie/)として再編されました。これまで、例えば個別のダッシュボードなどに対応していた AI 機能の Genie Space は Genie Agent となりました。Genie One は Genie Agent を含めたワークスペース内のオブジェクトに網羅的にアクセス可能な、Databricks の入口となるような AI 機能になります。モバイルアプリも公開され、今後 Databricks は Genie One を起点としてより簡単にデータ活用が可能なデータ基盤へ進化していくでしょう。

一方で、Genie One を十分に活用するためには、それを想定した対応が必要になります。例えば、ダッシュボードに対応する形で存在する Genie Space はそのダッシュボードに登録されたデータにしかアクセスできないため、ダッシュボードで取り扱うデータ、コンテキストに閉じられますが、Genie One はそうではなく、全てのオブジェクトを参照した回答が可能です。記事執筆時点での Databricks では、SQL エディタが使えるユーザは自由にダッシュボードを作ることが可能なため、様々な人が作ったオブジェクトがその参照先の一つになってしまいます。

この状態ではどれが信頼度が高いデータでどれがそうではないかの判別が難しいため（勿論、Genie はアクセスの頻度やデータの作成者などを考慮してくれるようですが）、Genie One には、野良的に生まれたオブジェクトよりも優先的に使って欲しいデータを指定したくなります。

## 「認定済みタグ」とは

Databricks では、データアセットに対して認定済みタグ/非推奨タグをつけることができます（[ドキュメント](https://docs.databricks.com/aws/ja/data-governance/unity-catalog/certify-deprecate-data)）。

![](https://docs.databricks.com/aws/ja/assets/images/certified-tag-a4a7c7e9541215b1e10de1898bee158c.png)
*認定済みタグ（[公式ドキュメント](https://docs.databricks.com/aws/ja/data-governance/unity-catalog/certify-deprecate-data)から引用）*

![](https://docs.databricks.com/aws/ja/assets/images/deprecated-tag-0cb9e54fc18b8af264227001799b3247.png)
*非推奨タグ（[公式ドキュメント](https://docs.databricks.com/aws/ja/data-governance/unity-catalog/certify-deprecate-data)から引用）*

このタグがあることで、社内の Databricks の利用者に対してデータアセットの信頼性を示せるだけでなく、Databricks の AI 機能である [Genie One](https://docs.databricks.com/aws/ja/genie-one/) に対しても優先して使うべきデータアセットであることを伝えることができます（[リリースノート](https://docs.databricks.com/aws/en/ai-bi/release-notes/2026#genie-one-enhancements-9)）。

## タグを dbt で管理したい

認定済みタグを含めた管理タグは割り当てを自動で行うこともできるようで（[ドキュメント](https://docs.databricks.com/aws/ja/admin/governed-tags/automate-tag-assignment), 2026-09-01 時点ではベータ版機能）、最終的には自動化された運用にはなっていきそうですが、運用を始める 1 歩目としては人間がタグの有無を認識した状態で始めたいです。

IVRy では、Databricks 上にある分析用データの生成パイプラインを [dbt Core](https://github.com/dbt-labs/dbt-core) を使って構築しているので、ここで一緒に管理したいです。dbt で管理できれば、認定済み/非推奨のタグをつけるにも GitHub で人間のレビューを通すこともできて良さそうです。

## dbt のデータモデルに認定済みタグをつける

### 権限設定

認定済みタグを付与するには以下の権限が必要です。

- `system.certification_status` に対する `ASSIGN` 権限
- 対象オブジェクトに対する `APPLY TAG` 権限

`system.certification_status` に触れる権限はアカウントレベルの権限なので、少し注意が必要です。どのオブジェクトに認定済みタグを付与可能かどうかは `APPLY TAG` の権限で絞れるので、それを使って任意のオブジェクトを認定しないように調整する必要があります。dbt 実行用のサービスプリンシパルにだけ権限を渡すことで、認定されているものは全てレビューを介している、と統制を組むことも可能です。

### dbt 側の記述

[dbt-databricks adapter](https://github.com/databricks/dbt-databricks) では databricks tag を付与することが可能なので、素直に記述すればそのまま反映されます。ありがたい！

```sql
{{ config(
    materialized="metric_view",
    alias="hogehoge",
    databricks_tags={'system.certification_status': 'certified'},
) }}
```

IVRy では [metric view](https://docs.databricks.com/aws/ja/uc-semantics/metric-views/) を分析用データの基本として使いたいので、優先的に使って欲しい metric view に認定済みタグをつけています。

## まとめ

Databricks の認定済みタグを dbt で管理する方法についてまとめました。

課題として、現時点での dbt-databricks アダプターの databricks_tags は付与はできるが削除ができない（dbt 側でタグ設定した後でその設定を削除しても、Databricks 上の実体からはタグが外れない）、というものがあるため、厳密に dbt 管理を続ける場合は post-hook などで対応する必要がありそうです。

## 最後に

IVRy のデータチームでは、各種ポジションを募集中です。

https://ivry.jp/lp-article/data-team/

私と同じアナリティクスエンジニアのポジションも募集中ですので、もし気になることがあれば是非カジュアル面談でお話しさせていただけると嬉しいです。

https://herp.careers/v1/ivry/NuVTXpVGqk1g
