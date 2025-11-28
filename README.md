# aws-network-firewall-policy-switcher

AWS Network Firewall のファイアウォールポリシーを、運用フェーズに合わせて動的に切り替えるための Lambda ツールです。
平常時の「通常 (Normal) モード」と、作業時の「メンテナンス (Maintenance) モード」を安全かつ即座に切り替える運用を実現します。


## 開発の背景 (Background)

### 課題: 高セキュリティ環境下での RHEL 9 アップデート失敗
本環境はプライベートサブネットで構成されており、AWS Network Firewall によってアウトバウンド通信を厳格に制限（FQDN ホワイトリスト方式）しています。通常時は、業務に必要な最小限のドメインへの通信しか許可していません。

しかし、RHEL 9 (Red Hat Enterprise Linux) のパッケージ管理 (`dnf/yum`) は、Red Hat CDN (Content Delivery Network) を使用します。CDN はアクセス先の IP アドレスが頻繁に変動し、かつ参照するミラーサイトのドメインも多岐にわたるため、**通常の厳格なファイアウォールポリシーのままでは、正規のアップデート通信がブロックされてしまい、OS の更新ができない**という課題が発生しました。

### 解決のアプローチ
すべての CDN ドメインを常時許可リストに追加し続ける運用は、管理コストが高く、セキュリティホールになるリスクもありました。
そこで、「セキュリティレベルを下げることなく、運用作業を遂行する」ために、**OS アップデート作業時のみ一時的にファイアウォールポリシーを「メンテナンスモード（広範囲な通信を許可するルール）」へ切り替える** 運用フローを設計しました。

本ツールは、このポリシー切り替え作業におけるオペレーションミスを防ぎ、チャットボットやジョブから安全かつ迅速にモード変更を行うために開発されました。

## モードの定義 (Modes)

| モード名 | キーワード | 説明 |
| :--- | :--- | :--- |
| **通常モード** | `normal` | **(Default)** 平常時の運用状態です。正規の厳格なフィルタリングルールが適用されます。 |
| **メンテモード** | `maintenance` | アップデート/メンテナンス作業用です。OS更新に必要なCDN通信の許可や、ログ調査用のファイアウォールポリシーが適用されます。 |

## アーキテクチャ (Architecture)

```mermaid
graph TD
    Op[Operator / Tool] -->|Invoke with JSON| Lambda[Policy Swapper]
    Lambda -->|AssociateFirewallPolicy| ANF[AWS Network Firewall]
    
    subgraph Policy States [ファイアウォールポリシーの状態]
        Normal[通常ファイアウォールポリシー<br/>(厳格なホワイトリスト)]
        Maint[メンテナンス用ファイアウォールポリシー<br/>(RHEL CDN許可/監査)]
    end
    
    ANF -.-> Normal
    ANF -.-> Maint

使い方 (Usage)Lambda 関数を以下のペイロード（JSON）で実行します。1. メンテナンスモードに入るOSアップデートなどの作業開始時に実行します。JSON{
  "mode": "maintenance"
}
2. 通常モードに戻す作業終了時に実行します。※ mode を省略した場合も自動的に normal になります（安全設計）。JSON{
  "mode": "normal"
}
環境変数 (Environment Variables)この関数を動作させるために、以下の環境変数を設定してください。変数名説明FIREWALL_ARN対象となる Network Firewall の ARNNORMAL_POLICY_ARN通常時（Normal）に使用するファイアウォールポリシーの ARNMAINTENANCE_POLICY_ARNメンテナンス時（Maintenance）に使用するファイアウォールポリシーの ARN必要な権限 (IAM Permissions)この Lambda 関数には、以下の権限を持つ IAM ロールを付与してください。JSON{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "network-firewall:AssociateFirewallPolicy"
            ],
            "Resource": "arn:aws:network-firewall:region:account-id:firewall/my-firewall"
        }
    ]
}

---

