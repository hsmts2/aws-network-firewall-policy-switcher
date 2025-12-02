## Table of Contents

- [Solution Overview](#solution-overview)
- [Architecture Diagram](#architecture-diagram)
- [Background](#background)
  - [Challenge](#challenge)
  - [Solution Approach](#solution-approach)
- [Mode Definitions](#mode-definitions)



## Solution Overview
(ここに本文...)

## Architecture Diagram
(ここに本文...)

# aws-network-firewall-policy-switcher

AWS Network Firewall のファイアウォールポリシーを、運用フェーズに合わせて動的に切り替えるための Lambda ツールです。
平常時の「通常 (Normal) モード」と、作業時の「メンテナンス (Maintenance) モード」を安全かつ即座に切り替える運用を実現します。

## Background

### Challenge
**課題: 高セキュリティ環境下での RHEL 9 アップデート失敗**

本環境はプライベートサブネットで構成されており、AWS Network Firewall によってアウトバウンド通信を厳格に制限（FQDN ホワイトリスト方式）しています。通常時は、業務に必要な最小限のドメインへの通信しか許可していません。

しかし、RHEL 9 (Red Hat Enterprise Linux) のパッケージ管理 (`dnf/yum`) は、Red Hat CDN (Content Delivery Network) を使用します。CDN はアクセス先の IP アドレスが頻繁に変動し、かつ参照するミラーサイトのドメインも多岐にわたるため、**通常の厳格なファイアウォールポリシーのままでは、正規のアップデート通信がブロックされてしまい、OS の更新ができない**という課題が発生しました。

### Solution Approach
**解決のアプローチ**

すべての CDN ドメインを常時許可リストに追加し続ける運用は、管理コストが高く、セキュリティホールになるリスクもありました。
そこで、「セキュリティレベルを下げることなく、運用作業を遂行する」ために、**OS アップデート作業時のみ一時的にファイアウォールポリシーを「メンテナンスモード（広範囲な通信を許可するルール）」へ切り替える** 運用フローを設計しました。

本ツールは、このポリシー切り替え作業におけるオペレーションミスを防ぎ、チャットボットやジョブから安全かつ迅速にモード変更を行うために開発されました。

## Mode Definitions

| モード名 | キーワード | 説明 |
| :--- | :--- | :--- |
| **Normal Mode** | `normal` | **(Default)** 平常時の運用状態です。正規の厳格なフィルタリングルールが適用されます。 |
| **Maintenance Mode** | `maintenance` | アップデート/メンテナンス作業用です。OS更新に必要なCDN通信の許可や、ログ調査用のファイアウォールポリシーが適用されます。 |

