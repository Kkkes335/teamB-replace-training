# ALB作成手順書（HTTPS化・ACM証明書発行込み）

## 1. 作業概要
- 目的：公開Webの入口として Internet-facing ALB を作成し、HTTPS(443)でアクセス可能にする
- 想定構成：
  - Internet
  - IGW
  - ALB（Public Subnet × 2）
  - Web/APサーバ（Private Subnet × 2）
- 通信方式（推奨）：
  - 外部 → ALB：HTTPS / 443
  - HTTP / 80 は HTTPS / 443 へリダイレクト
  - ALB → Web/AP：HTTP / 80（ALBでTLS終端）
- 前提：
  - VPC作成済み
  - 別AZにPublic Subnetが2つある
  - Web/AP用EC2はPrivate Subnetにある
  - DNSはNSDで管理している

---

## 2. 事前確認

### 2.1 VPC / Subnet / Route
- VPCが作成済みであること
- 別AZに Public Subnet が2つ存在すること
- それぞれの Public Subnet の Route Table に `0.0.0.0/0 -> Internet Gateway` が設定されていること

### 2.2 Web/APサーバ
- Web/APサーバが Private Subnet に存在すること
- アプリケーションが HTTP(80) で待ち受けていること
- ALB からの疎通を許可する SG が付与されていること

### 2.3 セキュリティグループ
#### ALB用SG
- Inbound
  - TCP 443 from `0.0.0.0/0`
  - TCP 80 from `0.0.0.0/0`（HTTP→HTTPSリダイレクトを使う場合）
- Outbound
  - All traffic（研修ではこれで可）

#### Web/AP用SG
- Inbound
  - TCP 80 from ALB用SG
  - SSH 22 from 踏み台用SG（必要な場合のみ）
- Outbound
  - 必要に応じて許可（研修では All traffic でも可）

---

## 3. ACMで証明書を発行する（HTTPS化の前提）

### 3.1 ACMコンソールを開く
- AWSコンソール → Certificate Manager（ACM）
- リージョンが ALB を作成するリージョンと同じであることを確認する
  - 例：`us-west-2`

### 3.2 パブリック証明書のリクエスト
- `Request a certificate` を押下
- `Request a public certificate` を選択
- ドメイン名を入力
  - 例：`www.teamb2.entrycl.net`
- 必要に応じて SAN（追加ドメイン）を設定
  - 例：`teamb2.entrycl.net`

### 3.3 検証方法を選択
- `DNS validation` を選択

### 3.4 DNS検証レコードを確認
- ACM が表示する CNAME レコード（Record name / Record value）を確認する

### 3.5 NSDに検証用CNAMEを追加
外部DNS（NSD）のゾーンファイルに、ACMが指定した CNAME を追加する

例（実際の値はACM画面のものをそのまま使う）：
```dns
_acm-validation-token.www.teamb2.entrycl.net. IN CNAME _xxxx.acm-validations.aws.
```

### 3.6 ゾーン反映
- zone の serial を更新する
- 構文チェック
```bash
sudo /usr/local/sbin/nsd-checkzone teamb2.entrycl.net /etc/nsd/teamb2.entrycl.net.zone
```
- 反映
```bash
sudo /usr/local/sbin/nsd-control reload teamb2.entrycl.net
```
または
```bash
sudo systemctl restart nsd
```

### 3.7 証明書発行完了を待つ
- ACMコンソールでステータスが `Issued` になることを確認する

---

## 4. Target Groupを作成する

### 4.1 Target Group作成
- EC2コンソール → Target Groups → Create target group

### 4.2 基本設定
- Target type：`Instances`
- Name：例 `tg-web-80`
- Protocol：`HTTP`
- Port：`80`
- VPC：対象VPC

### 4.3 Health Check
- Protocol：`HTTP`
- Path：`/`
  - アプリにヘルスチェック専用URLがある場合は `/health` などに変更

### 4.4 ターゲット登録
- Private Subnet 内の Web/APサーバを選択
- `Include as pending below`
- `Create target group`

---

## 5. ALBを作成する

### 5.1 ALB作成開始
- EC2コンソール → Load Balancers → Create load balancer
- `Application Load Balancer` を選択

### 5.2 基本設定
- Name：例 `CL-TeamE-test0`
- Scheme：`Internet-facing`
- IP address type：`IPv4`
- VPC：対象VPC

### 5.3 Subnet選択
- **AZ1の Public Subnet**
- **AZ2の Public Subnet**

> 注意：ALBに設定するのは Public Subnet。  
> Web/APが Private Subnet にいる構成でも、ALB 自体は Public Subnet に置く。

### 5.4 Security Group設定
- 事前に作成した ALB用SG を選択

---

## 6. HTTPS Listenerを作成する

### 6.1 443 Listener
- Protocol：`HTTPS`
- Port：`443`

### 6.2 証明書選択
- `From ACM`
- 手順3で発行済みの証明書を選択

### 6.3 Default Action
- 作成済みの Target Group（例：`tg-web-80`）へ転送

---

## 7. HTTP(80) → HTTPS(443) リダイレクトを設定する（推奨）

### 7.1 80 Listener追加
- Protocol：`HTTP`
- Port：`80`

### 7.2 Action
- `Redirect to URL`
- Protocol：`HTTPS`
- Port：`443`
- Status code：`HTTP_301`

---

## 8. 作成後の確認

### 8.1 ALB状態確認
- Load Balancer の Status が `active`

### 8.2 Target Group状態確認
- 登録した EC2 が `healthy`

### 8.3 ALB DNS名で疎通確認
- ALB詳細画面の `DNS name` を確認
- ブラウザで `https://<ALB_DNS_NAME>` にアクセス
- 正常に表示されること

### 8.4 HTTP→HTTPS確認
- `http://<ALB_DNS_NAME>` にアクセス
- 自動で HTTPS にリダイレクトされること

---

## 9. DNS（NSD）の www を ALBへ向ける

### 9.1 ゾーンファイル修正
`www` を ALB の DNS名へ CNAME で向ける

例：
```dns
www IN CNAME CL-TeamE-test0-2044806691.us-west-2.elb.amazonaws.com.
```

> 注意：
> - 末尾の `.` を付ける
> - ALB の IP を A レコードで固定しない

### 9.2 serial更新・反映
- serial を +1
- 構文チェック
```bash
sudo /usr/local/sbin/nsd-checkzone teamb2.entrycl.net /etc/nsd/teamb2.entrycl.net.zone
```
- 反映
```bash
sudo /usr/local/sbin/nsd-control reload teamb2.entrycl.net
```
または
```bash
sudo systemctl restart nsd
```

### 9.3 DNS確認
```bash
dig @127.0.0.1 www.teamb2.entrycl.net
```

---

## 10. 完了条件
以下を満たせば作業完了とする

- ACM証明書が `Issued`
- ALBが `active`
- Target Groupが `healthy`
- `https://www.teamb2.entrycl.net` で画面表示できる
- `http://www.teamb2.entrycl.net` で HTTPS にリダイレクトされる
