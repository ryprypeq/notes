# Authelia 運用検討(caddy-security で気にしていた点との比較)

caddy-security の検証(`caddy-security-手順書.md`)で気にしていた点を、Authelia に当てはめて見直した記録。

- 作成日: 2026-11-22
- **重要な前提: Authelia は、この環境では動かしていない。** 以下はすべて**公式ドキュメント(web 検索の結果)と一般的な知識**にもとづく。caddy-security のように実機で確認した内容ではない。確認の度合いを項目ごとに示す。
  - **公式で確認**: 公式ドキュメントのページ(検索結果)に記載があった
  - **未確認**: 検索で見つからなかった、または私の知識のみ。採用前に公式ドキュメントか実機で確認が必要
- 想定: 社内オンプレ・外部非公開・外部 ID 基盤なし(caddy-security 手順書 12.2・12.3 と同じ)。

---

## 1. 結論(サマリ)

| 気にしていた点 | caddy-security(検証済み) | Authelia |
|---|---|---|
| TOTP の鍵の保存 | `users.json` に**平文**(7.7) | **暗号化して保存**(公式で確認) |
| TOTP の鍵を API で返すか | 登録後も API 応答に**含まれる** | 未確認(返さない作りと思われるが、確認していない) |
| 管理者が鍵を読めるか | `info user` で**平文で読める** | CLI の `totp export` で取得可能(暗号化の鍵が必要) |
| 他ユーザーの MFA 解除 | `users.json` 編集+`reload`、またはルール変更で回避 | **CLI で削除できる**(公式で確認) |
| 試行制限 | **送信元単位**(5回/5分、固定) | **ユーザー単位**(既定 3回/2分→5分ブロック、設定可)(公式で確認) |
| 管理者の権限モデル | 管理 API+ロール(`authp/admin`) | **OS 側の権限のみ**(CLI が設定と DB を直接操作)(公式の CLI で確認。管理 API の有無は未確認) |
| ログイン後のアプリ選択 | `ui link` でメニュー表示 | **未確認**(提供しないと思われる) |
| **メール(SMTP)** | メール不要で運用できた | **本人による MFA 登録・パスワード再設定に、メールが必要**(公式で確認) |

**一番の分かれ目は SMTP。** Authelia では、ユーザー本人が 2FA を登録したり、パスワードを再設定したりするときに、メールで届くワンタイムコードによる本人確認が必要になる。**社内に使える SMTP が無い場合、この自己登録・自己再設定が使えない**(§5)。

---

## 2. ユーザーとパスワード

| 項目 | 内容 | 確認 |
|---|---|---|
| ユーザーの保存先 | YAML ファイル(`users:` の下に `password`(ハッシュ)・`email`・`groups`・`disabled` など)、または LDAP | 公式で確認 |
| パスワードのハッシュ | Argon2(argon2id 推奨)が既定。パラメーターは設定で調整できる | 公式で確認 |
| ハッシュの生成 | `authelia crypto hash generate argon2`。`--random` で、ランダムなパスワードとハッシュを同時に作れる | 公式で確認 |
| 書き戻し | ユーザーがパスワードを再設定すると、Authelia が同じ形式でファイルに書き戻す(ファイルの拡張子・形式を勝手に変えない) | 公式で確認 |
| パスワード再設定・変更の無効化 | `authentication_backend.password_reset.disable` / `password_change.disable` の設定がある | 公式で確認(各設定の細かい範囲は未確認) |
| ユーザー単位の無効化 | ユーザーの項目に `disabled` | 公式で確認 |
| 管理画面 | 未確認(ファイル編集が基本と思われる) | 未確認 |

**caddy-security との違い(運用)**

- 初期パスワードは、**管理者がハッシュを作って YAML に書く**形になる。`--random` で作ったランダムなパスワードを本人に伝える運用なら、caddy-security の「生成パスワードを管理者の画面に出して本人に伝える」(手順書 4.3)と同じ構造になる。
- **初回ログイン時の変更強制の有無は未確認**。caddy-security では強制されなかった。
- ユーザー追加・削除が「ファイル編集」になる(管理 API・CLI がない場合)。複数管理者での同時編集や、ファイルの反映方法(再読込の要否)は**未確認**。

---

## 3. MFA の鍵の保護(最も気にしていた点)

| 項目 | 内容 | 確認 |
|---|---|---|
| 保存時の暗号化 | TOTP の鍵(およびキャッシュ・ワンタイムコード等)を暗号化して保存。AES-GCM(256bit)で、`storage.encryption_key` が必須 | 公式で確認 |
| 暗号化の対象 | 4.33.0 以降は、TOTP の鍵に加え、U2F の公開鍵の列も暗号化 | 公式で確認(リリースノート) |
| 暗号化の鍵の管理 | 環境変数 `AUTHELIA_STORAGE_ENCRYPTION_KEY_FILE`(ファイル)で渡せる。**ファイルでの受け渡しが推奨** | 公式で確認 |
| 鍵の変更 | `change-key` コマンドと、鍵の整合を確認する `check` コマンドがある | 公式で確認(手順の詳細は未確認) |
| 管理者による鍵の取得 | `authelia storage user totp export` がある(暗号化の鍵を含む最小限の設定が必要) | 公式で確認 |
| 登録後の API が鍵を返すか | 未確認 | **未確認(要実機検証)** |
| パスキー(WebAuthn) | CLI に `webauthn` の削除コマンドがある。パスキー対応そのものは、各機能の公式ページで要確認 | 一部確認 |

**caddy-security との比較**

- 保存の暗号化は、**Authelia の方が明確に良い**(caddy-security は平文)。
- ただし、**暗号化の鍵は別に守る必要がある**。DB(SQLite)と同じ場所に置くと意味が薄れる。失うと、全員の TOTP を再登録することになる。
- **OS の権限を持つ管理者は、`totp export` で鍵を取れる**ので、管理者に対する露出は残る(caddy-security の `info user` と同様の構造)。ただし、一般ユーザーの API 経由の露出は、**要実機検証**。

---

## 4. 管理者の権限と、他ユーザーの MFA の扱い

| 項目 | 内容 | 確認 |
|---|---|---|
| 管理 CLI | `authelia storage user ...` などが、**設定と DB を直接**操作する。リモートの管理 API・ロールではない | 公式の CLI で確認(管理 API の有無は未確認) |
| TOTP の削除 | `authelia storage user totp delete <ユーザー名>`。公式が「再登録させたいときの手段」として紹介している | 公式で確認 |
| パスキーの削除 | `authelia storage user webauthn delete <ユーザー名> --all` / `--description <名前>` / `--kid <鍵ID>` | 公式で確認 |
| 削除の効果 | 該当ユーザーは MFA が未登録に戻る。再登録に、メールのワンタイムコードが要る(§5) | 公式で確認 |

**caddy-security との違い**

- 「誰が管理者か」は、**OS 側の権限(Authelia の設定ファイル・DB・暗号化の鍵に触れること)**で決まる。caddy-security のように、管理者ロールで 403 を出し分ける仕組みは、見当たらなかった。
- したがって、**管理者の最小化 = Authelia のサーバーに入れる人・ファイルを読める人の最小化**になる。サーバーの SSH・バックアップの権限が、そのまま管理者権限。
- caddy-security の「ロールの付与・剥奪の即効性」(古いトークンが残る問題)に相当する論点は、**CLI が直接 DB を操作するため、設計上は別物**(セッションの失効は未確認)。

---

## 5. メール(SMTP)が必要か【分かれ目】

| 項目 | 内容 | 確認 |
|---|---|---|
| 通知方法 | **ちょうど1つ**の通知手段を設定する(SMTP または filesystem) | 公式で確認 |
| filesystem の位置づけ | **テスト・開発用**。通知を1つのファイルに書き出すだけで、ユーザーには届かない。公式は、**ユーザー自身によるパスワード再設定と、新しい 2FA デバイスの登録を妨げる**と説明している | 公式で確認 |
| 本人確認 | 重要な操作(2FA の登録など)の前に、**メールで届くワンタイムコード**で本人確認する。4.38 で、従来のリンク方式からコード方式に変わった | 公式で確認 |
| 登録の通知 | 2FA を登録すると、「Second factor registered」の通知が送られる | 公式で確認 |

**意味すること**

- **使える SMTP(社内メールリレーなど)がある**: Authelia の自己登録・自己再設定が使える。ユーザーが自分で TOTP を登録し、紛失時は管理者が `totp delete` → 本人がメールのコードで再登録、という運用になる。
- **使える SMTP が無い**: filesystem 通知では、ユーザーが自分で 2FA を登録・再設定できない。**管理者が通知ファイルからコードを取り出して本人に渡す**といった手作業になる可能性があるが、**公式の運用として成立するかは未確認**。管理者側で TOTP を生成して渡すコマンドの有無も、**未確認**(要確認)。
- caddy-security は、メール無しでも、自己登録・ログインが成立した(手順書 11 章)。**この点は、Authelia が不利**。

---

## 6. 試行制限(ブルートフォース対策)

| 項目 | caddy-security | Authelia |
|---|---|---|
| 単位 | **送信元アドレス** | **ユーザー(アカウント)** |
| 既定の回数・時間 | 5回失敗 → 5分ブロック(**固定**) | `max_retries: 3`(`find_time: 2m` 内に3回失敗 → `ban_time: 5m` ブロック) |
| 設定変更 | 不可 | **可能**(環境変数も用意されている)。`max_retries: 0` で無効化 |
| 弱点 | 同じ送信元(NAT・LB)の全員が巻き込まれる(手順書 12.2) | **ユーザー名が分かれば、誰でもそのアカウントを5分間ロックできる**(嫌がらせの DoS)。公式の説明は「ユーザーが BAN される」で、送信元単位の制限の有無は**未確認** |

- オンプレで、社内の多数の端末が1つの送信元に見える構成なら、**ユーザー単位の Authelia の方が、巻き込まれにくい**。
- 一方、社内の誰かが、他人のアカウントを意図的にロックする嫌がらせはできる。管理者アカウントも対象。

---

## 7. ログイン後のアプリ選択・アクセス制御

| 項目 | 内容 | 確認 |
|---|---|---|
| アクセス制御 | ルールを、サブドメイン・ユーザー・グループ・URI・メソッド・ネットワークで定義し、ルールごとに「1要素 / 2要素 / 拒否」を選べる | 公式で確認 |
| アプリの一覧メニュー | caddy-security のポータルにあった「Applications」一覧(`ui link`)に相当する機能は、**未確認**(提供しないと思われる) | 未確認 |
| Caddy との連携 | Caddy の `forward_auth` で、リクエストごとに Authelia に問い合わせる。Caddy 2.5.1 以降。既定のエンドポイントは `/api/authz/forward-auth`。`Remote-User` などのヘッダーをアプリに渡せる | 公式で確認 |
| 注意 | 公式ガイドは、「明示的な forward auth」の設定を誤ると**認証を回避される**おそれがあると警告し、**信頼するプロキシの設定を先に確認**するよう求めている | 公式で確認 |

- 「ログイン後に飛べるアプリを選べるか」は、Authelia では**メニューではなく、アクセス制御のルールで『行けるかどうか』を決める**形になる。アプリへのリンク集が必要なら、別途用意する(社内ポータルなど)。

---

## 8. 運用(バックアップ・鍵の差し替え)

**バックアップの対象(caddy-security の `users.json` 1つに対し、増える)**

1. ストレージ(SQLite など。TOTP・パスキー・設定・ログ)
2. ユーザーのファイル(YAML)
3. **ストレージの暗号化の鍵**(DB とは**別の場所**に保管する。失うと、全員が MFA を再登録することになる)
4. 設定ファイル、その他のシークレット(セッションなど。**詳細は未確認**)

**DB の要否**

- 専用の DB サーバーは必須ではない(SQLite のファイルでよい構成が一般的。公式の設定ページに SQLite と MySQL がある)。
- ストレージの設定自体は**必須**。
- セッションは既定ではメモリ上で、再起動で再ログインになると思われる(**未確認**)。複数台構成や永続化には Redis が要る。

**鍵・シークレットの差し替え**

- 暗号化の鍵の変更コマンドはある(`change-key`)。手順と、**セッション用シークレット等の差し替え**は、公式の設定ページで要確認。
- caddy-security の「署名鍵は2か所に書く・新しい鍵を先に書く」(手順書 12.1)のような落とし穴が、Authelia にあるかは**未確認**。

---

## 9. その他の前提

- **HTTPS と名前解決(DNS)**: パスキー(WebAuthn)やセッションの Cookie は、一般に HTTPS とホスト名(ドメイン)が前提になる。**社内 DNS・社内 CA の準備**が必要になる可能性がある(**未確認**)。
- **サービスが1つ増える**: Authelia の起動・監視・アップデートが必要。
- **mTLS との併用**: Authelia の前段で mTLS を要求する構成も、技術的には可能(Caddy 側)。caddy-security の mTLS 検証(クライアント証明書がなければ接続不可、管理 CLI は証明書を提示できない)とは別の論点になる(Authelia の CLI は、リモートの管理 API ではないため、この制約は当てはまらない見込み。**未確認**)。

---

## 10. 採用の判断基準

| 状況 | おすすめ |
|---|---|
| 社内に使える **SMTP がある**、HTTPS と内部 DNS を用意できる、MFA の鍵をきちんと保護したい、アプリが複数ある | **Authelia** が有力 |
| **SMTP が使えない**、アプリが1〜2つ、サービスを増やしたくない | **caddy-security の補強**(手順書 12.2)、または **mTLS + パスワード**(検証済み) |
| 管理者だけ強い第二要素にしたい | パスキー(両方で要実機検証) |

**採用前に確認したいこと(優先順)**

1. **社内に使える SMTP があるか。** 無いなら、Authelia の自己登録・自己再設定が使えないので、他の案が有利。
2. 登録後の API が、TOTP の鍵を返さないか(実機)。
3. 管理者が、メール無しで TOTP を発行・登録する手段(CLI)があるか。
4. 初回ログイン時のパスワード変更の強制の有無。
5. 社内 DNS・HTTPS の準備ができるか。
6. セッション・設定のバックアップと、シークレットの差し替え手順。

---

## 参考(公式ドキュメント)

- [Storage(暗号化の鍵)](https://authelia.com/configuration/storage/introduction)
- [Secrets](https://authelia.com/configuration/methods/secrets)
- [4.33.0 リリース(TOTP・U2F の暗号化)](https://www.github.com/authelia/authelia/releases/tag/v4.33.0)
- [File(ユーザーのファイル)](https://www.authelia.com/configuration/first-factor/file/)
- [Passwords](https://www.authelia.com/reference/guides/passwords/)
- [`authelia crypto hash generate argon2`](https://www.authelia.com/reference/cli/authelia/authelia_crypto_hash_generate_argon2/)
- [Regulation(試行制限)](https://www.authelia.com/configuration/security/regulation/)
- [`authelia storage user totp delete`](https://www.authelia.com/reference/cli/authelia/authelia_storage_user_totp_delete/)
- [`authelia storage user webauthn delete`](https://www.authelia.com/reference/cli/authelia/authelia_storage_user_webauthn_delete/)
- [`authelia storage user totp export`](https://www.authelia.com/reference/cli/authelia/authelia_storage_user_totp_export)
- [Notifications(通知)](https://www.authelia.com/configuration/notifications/introduction/)
- [4.38 リリースノート(ワンタイムコード)](https://www.authelia.com/blog/4.38-release-notes/)
- [Security measures](https://www.authelia.com/overview/security/measures/)
- [Caddy との連携](https://www.authelia.com/integration/proxies/caddy/)
- [Proxy Authorization](https://www.authelia.com/reference/guides/proxy-authorization/)
- [Caddy: `forward_auth`](https://caddyserver.com/docs/caddyfile/directives/forward_auth)
