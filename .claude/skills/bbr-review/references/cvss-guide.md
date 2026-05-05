# CVSS 評価ガイド（バグバウンティ向け）

Hacker101 の CVSS フレームワークを基に、バグバウンティで実際にスコアを判断するための実務ガイドをまとめる。レビュー時に CVSS の妥当性を判断する根拠としてここを参照する。

## サポートされている CVSS バージョン（2025-26 時点）

| プラットフォーム | サポート |
|---|---|
| HackerOne | CVSS 3.0（カスタム実装）/ 3.1 / 4.0 すべて、複数併用可 |
| Bugcrowd | CVSS 3.1 / 4.0、VRT と連携（CVSS 入力で技術 Severity が事前入力される） |
| Intigriti | CVSS 3.x / 4.x（v1.3 トリアージスタンダード準拠） |
| YesWeHack | CVSS 3.1（標準） |

**重要**：HackerOne の severity は 5 段階（None / Low / Medium / High / Critical）。Bugcrowd は VRT と連携して P1〜P5 で表現される（CVSS スコアはリサーチャー側に非表示の場合あり）。

CVSS 公式：[https://www.first.org/cvss/v3-1/](https://www.first.org/cvss/v3-1/) / [v4.0](https://www.first.org/cvss/v4-0/)
CVSS Calculator：[https://www.first.org/cvss/calculator/3.1](https://www.first.org/cvss/calculator/3.1)

## Severity Cap（最大severity上限）

HackerOne では「アセット関連レポートの場合、**環境メトリクスと最大 severity 上限**を考慮」と公式が明示している。プログラムごとに「このアセットの最大 Severity は Medium」のような上限が設定されることがある。

**実務上の意味**：
- ハッカーが Critical 主張しても、プログラム側 Cap で Medium に下がる
- 報酬計算は「Cap 後の Severity」基準
- レビュー時、対象アセットの Severity Cap が分かれば併記する

**Bugcrowd の場合**：CVSS から技術 Severity が自動マッピングされ、VRT 範囲（P1〜P5）に押し込められる。Cap は VRT 経由で適用される。

---

## CVSS v3.1 ベース 8 メトリクス

ベクトル文字列の例：`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N`

### Attack Vector (AV)
攻撃の到達経路。

| 値 | 意味 | 典型例 |
|----|------|--------|
| `N` (Network) | インターネット越しに攻撃可能 | Web アプリの XSS、SQLi |
| `A` (Adjacent) | 同一物理/論理ネットワーク（LAN、Bluetooth、隣接サブネット） | ARP スプーフィング、社内専用 API |
| `L` (Local) | ローカル OS アカウント/シェル必要 | DLL ハイジャック、ローカル特権昇格 |
| `P` (Physical) | 物理アクセス必要 | USB 経由攻撃、Cold-boot 攻撃 |

**バグバウンティでよく間違える点**：
- モバイルアプリの IPC 経由バグは `L`（同一端末上のアプリ）。
- Bluetooth / NFC 攻撃は `A`。

### Attack Complexity (AC)
攻撃成功に必要な「攻撃者がコントロールできない」条件。

| 値 | 意味 | 典型例 |
|----|------|--------|
| `L` (Low) | 特殊条件不要、常に成立 | Reflected XSS、典型的な SQLi |
| `H` (High) | レース条件、推測不能な ID、特定設定が必要 | TOCTOU、暗号鍵推測、UUID 列挙 |

**注意**：「ユーザーがログインしている必要がある」は `AC` ではなく `PR` で表現する。
**注意**：UUID（推測不可）IDOR は `AC:H`、連続 ID（推測可）IDOR は `AC:L`。

### Privileges Required (PR)
攻撃者自身が持つ必要のある権限。

| 値 | 意味 |
|----|------|
| `N` (None) | 認証不要 |
| `L` (Low) | 一般ユーザー権限（自己登録可能なアカウント等） |
| `H` (High) | 管理者・特権ユーザー |

**バグバウンティでよく間違える点**：
- 自己登録できるアプリで「ユーザー登録が必要」は `PR:L`。
- 招待制 SaaS でアカウントが必要なら `PR:L`（自由登録できないため）。
- 管理パネル内の脆弱性は `PR:H`。

### User Interaction (UI)
攻撃成立に「攻撃者以外のユーザー」のアクションが必要か。

| 値 | 意味 | 典型例 |
|----|------|--------|
| `N` (None) | 不要 | サーバ側 SQLi、SSRF |
| `R` (Required) | 必要 | Reflected XSS（リンククリック）、CSRF（リンクアクセス） |

**注意**：自己 XSS は実質的には UI:R + 攻撃者自身しか被害を受けないので CVSS では Severity が下がる。多くのプログラムで OOS。

### Scope (S)
脆弱性の影響が「脆弱なコンポーネント」を超えて「他のセキュリティ機構/コンポーネント」に及ぶか。

| 値 | 意味 |
|----|------|
| `U` (Unchanged) | 影響は脆弱なコンポーネントの権限内 |
| `C` (Changed) | 別のセキュリティ境界・テナント・サービスに波及 |

**バグバウンティでよく間違える点**：
- Stored XSS（自分のプロフィール）→ 他人のブラウザで実行：**`S:C`**（自分の DOM 権限が他ユーザーのセッションを侵害）。
- SSRF で AWS メタデータから IAM クレデンシャル奪取 → AWS リソース侵害：**`S:C`**。
- 同一テナント内 IDOR：通常 `S:U`。
- マルチテナント SaaS で他テナントのデータが見える IDOR：**`S:C`**。

### Confidentiality (C) / Integrity (I) / Availability (A)
影響度。

| 値 | 意味 |
|----|------|
| `H` (High) | 全データ侵害、システム全体改変、サービス完全停止 |
| `L` (Low) | 一部データ、限定的改変、性能劣化 |
| `N` (None) | 影響なし |

**よく間違える点**：
- Reflected XSS：DOM アクセス可能 → C:L、I:L、A:N（DoS にはならない）。
- 漏洩する情報の機密度で判断：パスワード/トークン漏洩なら `C:H`、メアドだけなら `C:L`。

---

## 典型脆弱性のベクトル例（Hacker101 + 実務調整）

### Stored XSS（一般ユーザー → 他ユーザー）
```
CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N → 5.4 (Medium)
```
- `PR:L`：投稿に登録が必要
- `UI:R`：被害者がページを見る必要
- `S:C`：自分のセッション境界を超えて他ユーザーに影響
- `C:L/I:L`：DOM アクセスとフォーム改ざん
- 被害者が**通常の閲覧フロー**でほぼ自動的に踏むケース（タイムライン・ダッシュボード等）は `UI:N` を主張でき、その場合 6.4 (Medium) になる

### Stored XSS（管理者 → 一般ユーザー、認証 panel）
```
CVSS:3.1/AV:N/AC:L/PR:H/UI:R/S:C/C:L/I:L/A:N → 4.8 (Medium)
```
- `PR:H` で大幅にスコアが下がる

### Reflected XSS（認証不要、URL クリック）
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N → 6.1 (Medium)
```
- `S:C` を取れるかが論点。多くのトリアージは S:C を採用するが、プログラム側が S:U で減点する場合もある。

### IDOR — 連番 ID で PII 読み取り/改ざん
```
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N → 8.1 (High)
```
- `AC:L`（連番なので推測可）
- `C:H/I:H`（PII の読み出しと改ざん）

### IDOR — UUID で PII 読み取りのみ（参照リーク経由）
```
CVSS:3.1/AV:N/AC:H/PR:N/UI:N/S:U/C:H/I:N/A:N → 5.9 (Medium)
```
- `AC:H`（UUID は推測不能、何らかのリーク経路が必要）
- `PR:N`：UUID リーク経路（公開ページの埋め込みリンク・キャッシュ etc.）を経由するため認証不要
- 認証必須ルートからのみ到達可能なら `PR:L` で 5.3 (Medium)

### SSRF（Full Response、AWS メタデータ漏洩 → IAM 漏洩）
```
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H → 9.9 (Critical)
```
- `S:C`（クラウド境界越え）
- `C:H/I:H/A:H`（IAM クレデンシャルで RCE 級）
- 認証不要なら `PR:N` で 10.0（最大）

### SSRF（Blind、内部スキャンのみ）
```
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:N → 4.3 (Medium)
```
- 純粋な Blind SSRF（C/I/A すべて N）は CVSS スコアが 0 になり Severity が出せないため、**内部到達点で何が読み取り得るか**を `C:L/I:L/A:L` のいずれかで主張する。
- 「DNS resolve しか確認できていない」段階なら CVSS では Severity を主張せず、「内部 IP 到達の経路が確立した PoC」を提示してから報告する。

### SQL Injection（認証不要、Boolean ベース、データ抽出可）
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N → 7.5 (High)
```

### SQL Injection（UNION ベース、書き込み可、DBA 権限）
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H → 9.8 (Critical)
```

### RCE（認証不要）
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H → 9.8 (Critical)
```
- `S:C` を主張できる場合（コンテナ脱出など）はさらに上。

### CSRF（重要操作、トークン保護なし）
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:H/A:N → 6.5 (Medium)
```
- 「メアド変更でアカウント乗っ取り」なら `I:H`、`C:H/I:H` まで主張できる場合あり。

### Open Redirect（純粋な、フィッシング誘導）
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:N/I:L/A:N → 4.3 (Medium)
```
- 多くのプログラムで Low / Informative。チェーン化（OAuth トークン窃取等）できれば High。

### Authentication Bypass（管理者ログイン可能）
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H → 9.8 (Critical)
```

### XXE（Out-of-Band、ファイル読み取り）
```
CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N → 6.5 (Medium)
```
- ファイル読み取り範囲が `/etc/passwd` だけか機密ファイルまで届くかで C:L/H を切り替え。

---

## CVSS 評価でレビュー時にチェックすべきポイント

1. **ベクトル文字列が CVSS:3.1 のフォーマットで完結しているか**（`CVSS:3.1/...` の形式）
2. **Score が calculator の出力と一致するか**（不一致は減点）
3. **AV/AC/PR/UI/S が脆弱性の実態と整合しているか**（自己 XSS で `PR:N/UI:N` のような明らかな不整合）
4. **C/I/A が漏洩データ・改ざん範囲と整合しているか**（メアド漏洩で `C:H` は過大）
5. **Severity ラベルが計算スコアに対応しているか**
   - 0.1-3.9 = Low
   - 4.0-6.9 = Medium
   - 7.0-8.9 = High
   - 9.0-10.0 = Critical

---

## レビューでの提示例

```markdown
**CVSS 推定**: 8.1 High `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N`

**ユーザー記載との差分**:
- ユーザー記載：`Critical (9.8)` `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H`
- 訂正点：
  - `PR:N` → `PR:L`：認証セッションが必要なため
  - `A:H` → `A:N`：可用性への影響なし（DoS ではない）
  - 結果：Critical → High

**Severity Cap 注意**：プログラムによってはこのアセットの最大 Severity
が Medium / High に Cap されている可能性あり。Bounty 額計算には Cap 後の
値が使われる。プログラム規約を再確認のこと。

最終的なスコアは対象プログラムのトリアージ判断に委ねるが、
上記が CVSS v3.1 仕様に沿った妥当な値。
```

## CVSS 4.0 への移行に関する注意

- CVSS 4.0 は「Threat」「Environmental」「Supplemental」メトリクスが追加され、
  Base スコアの計算ロジックも変更されている
- CVSS 3.1 から 4.0 への自動変換は**不可**（Bugcrowd 公式：手動確認が必須）
- HackerOne では併用可能だが、プログラムが指定する版を使うのが無難
- 現時点（2025-26）ではまだ多くのプログラムが 3.1 をデフォルトとしている

レビュー時、ユーザーがどの版を使うべきかをプログラム規約から確認できない場合、
**CVSS 3.1 を推奨**するのが安全。
