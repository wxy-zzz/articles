---
title: "お気に入りの GitHub Repository Settings"
emoji: "♥️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GitHub"]
published: true
published_at: 2024-09-24 12:00
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

- `New ruleset` から `New branch ruleset`
    <!-- Ruleset Name: `Default` とかでも -->
    - Enforcement status: `Active`
    <!-- Bypass list: `Add bypass` から `Repository admin` (`For pull requests only` にしておく) -->
    - Targets
        - `Add target` から `Include default branch`
    - Rules
        - Branch rules
            <!-- Restrict creations --><!-- Restrict updates --><!-- Restrict deletions --><!-- Require linear history: squash merge を使うので細かいことを気にしなくてよさそう --><!-- Require deployments to succeed: deployment を使わない想定 --><!-- Require signed commits: 厳しすぎる気がする -->
            - [x] Require a pull request before merging
                - Required approvals: `1` 以上の数字に。
                - [x] Dismiss stale pull request approvals when new commits are pushed
                      Approve 後に変更されると困っちゃうので有効化。
                <!-- Require review from Code Owners: 厳しすぎる気がする -->
                - [x] Require approval of the most recent reviewable push
                      最新の push はレビューされていて欲しいので有効化。 
                <!-- Require conversation resolution before merging: 厳しすぎる気がする -->
            - [ ] Require status checks to pass
                  CI テストを作ってから設定。
            <!-- Block force pushes --><!-- Require code scanning results: 後で調べる -->

<!-- Actions - General: 後で調べる --><!-- Actions - Runners --><!-- Webhooks --><!-- Environments --><!-- Codespaces --><!-- Pages --><!-- Code security: 後で調べる --><!-- Deploy keys --><!-- Secrets and variables - Actions: 後から設定 --><!-- Secrets and variables - Codespaces --><!-- Secrets and variables - Dependabot --><!-- GitHub Apps --><!-- Email notifications -->
