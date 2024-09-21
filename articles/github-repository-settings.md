---
title: "お気に入りの GitHub Repository Settings"
emoji: "♥️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub"]
published: false
---
設定しては忘れ、設定しては忘れ、を繰り返しているので備忘録として。

# General

<!-- General --><!-- - Repository name --><!-- - Template repository --><!-- - Require contributors to sign off on web-based commits: Web からの操作だけを制限しなくても良さそう -->

## Default branch

手癖でタイプしてしまうので base branch は `main` に。
<!-- Social preview: 外部露出するようになるまで不要 -->

## Features

必要になるまでは off にしておく。

- [ ] Wikis
- [ ] Issues
<!-- Sponsorships: 元々 off -->
- [ ] Preserve this repository
<!-- Discussions: 元々 off -->
- [ ] Projects

## Pull Requests

Pull Request の利用を前提とすると、Pull Request 単位の粒度の変更にしか興味がないことがほとんどなので、squash merge だけを有効にしておくのが好き。
個別の commit の commit message に悩みすぎなくてよくなるのも良い。
が、commit message が適当になりやすいので `Default commit message` は Pull Request のタイトルだけを使うようにしておく（Default は commit タイトル or Pull Request タイトル + commit 一覧（[ref](https://docs.github.com/ja/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/configuring-commit-squashing-for-pull-requests)））

- [ ] Allow merge commits
- [x] Allow squash merging
    - Default commit message: `Pull request title`
- [ ] Allow rebase merging

最新の CI テストをパスするかを確認したいので、branch の更新がしやすくしておきたい。

- [x] Always suggest updating pull request branches

<!-- Allow auto-merge Loading: マージタイミングをコントロールしたいことはありそう -->

マージ後の branch 削除は自動でやってくれて OK.

- [x] Automatically delete head branches

<!-- Archives --><!-- - Include Git LFS objects in archives --><!-- Pushes --><!-- - Limit how many branches and tags can be updated in a single push --><!-- Danger Zone --><!-- - Change repository visibility --><!-- - Disable branch protection rules --><!-- - Transfer ownership --><!-- - Archive this repository --><!-- - Delete this repository --><!-- Collaborators --><!-- Moderation options - Interaction limits -->

# Moderation options - Code review limits

意図的に off にしておく理由もないので、初期設定としては有効にしておく。

- [x] Limit to users explicitly granted read or higher access

<!-- Branches --><!-- Tags -->

# Rules - Rulesets

# Actions - General
<!-- Actions - Runners --><!-- Webhooks --><!-- Environments --><!-- Codespaces --><!-- Pages -->
# Code security
<!-- Deploy keys -->
# Secrets and variables - Actions
<!-- Secrets and variables - Codespaces -->
# Secrets and variables - Dependabot
<!-- GitHub Apps --><!-- Email notifications -->
