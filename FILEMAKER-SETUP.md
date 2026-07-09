# 退職願 — iPad（FileMaker Go）セットアップ

FileMaker Pro の印刷レイアウト `P｜退職願` は Sideways（`-fm-tategaki`）で縦書きを実現していますが、**FileMaker Go は Sideways 非対応**のため iPad では文字が横向きに崩れます。

本フォルダの `taishoku-negai.html` を Web Viewer で表示し、CSS `writing-mode: vertical-rl` で正しい縦書きを再現します。**PC / FileMaker Pro では従来の `P｜退職願` レイアウト印刷をそのまま使います。**

## ホスト環境について（重要）

`InformationBoard` は **FileMaker Server でホスト**されています。iPad の FileMaker Go はサーバー上のファイルに接続するだけなので、**`file:///` や各端末の `Get(DocumentsPath)` は使えません**（HTML は iPad ローカルに存在しません）。

Web Viewer の HTML は **HTTPS で配信される 1 か所**に置き、全クライアントが同じ URL を参照する構成にします。

```
iPad (FileMaker Go) ──接続──▶ FileMaker Server (InformationBoard)
        │
        └── Web Viewer ──HTTPS GET──▶ GitHub Pages（taishoku-negai.html）
```

## 1. HTML の公開 URL（GitHub Pages）

本番 URL:

```
https://happycare-dev.github.io/taishokunegai/taishoku-negai.html
```

`main` ブランチへ push すると GitHub Pages から自動配信されます（リポジトリ: `happycare-dev/taishokunegai`）。

### 更新の流れ

1. ローカルで `taishoku-negai.html` を編集
2. `git commit` → `git push origin main`
3. 数分後、上記 URL に反映される

### 動作確認

ブラウザで開けること（サンプルデータ付き）:

```
https://happycare-dev.github.io/taishokunegai/taishoku-negai.html
```

データ付きの確認例:

```
https://happycare-dev.github.io/taishokunegai/taishoku-negai.html?resignDate=2026/06/25&currentDate=2026/06/25&employeeName=永井　みゆき&companyName=株式会社CLOVERワークス&repTitle=代表取締役&repName=野口　潔
```

### FileMaker 内のベース URL

`IF` テーブルにグローバルフィールドを追加することを推奨します。

| フィールド例 | 値 |
|-------------|-----|
| `IF::g_TaishokuHtmlUrl` | `https://happycare-dev.github.io/taishokunegai/taishoku-negai.html` |

環境ごとに URL を変える必要がなければ、この固定値で全クライアント共通です。

## 2. FileMaker 側の変更概要

| 項目 | 内容 |
|------|------|
| 既存レイアウト `P｜退職願` | **変更しない**（PC 印刷用） |
| 新規レイアウト `P｜退職願｜iPad` | Web Viewer 1 つのみ（全画面） |
| スクリプト `P｜退職願` | `isiOS` 分岐内で Web Viewer を設定（別スクリプト不要） |

## 3. 新規レイアウト `P｜退職願｜iPad`

- テーブル: `入社登録_EmployeeM`（既存 `P｜退職願` と同じ）
- レイアウトサイズ: 画面いっぱい（例: 768 × 1024）
- オブジェクト: **Web Viewer** 1 つ（オブジェクト名: `wvTaishoku`）
- URL はスクリプトで `Set Web Viewer` により設定（下記）

## 4. スクリプト `P｜退職願｜URL構築`（新規）

JSON のエスケープは **`JSONSetElement`** を使います（カスタム関数不要、FileMaker 18+）。

退職願 HTML に渡す URL を組み立てます。戻り値を `$url` に格納します。

```filemaker
# 前提: 入社登録_EmployeeM レコード上で実行（既存 P｜退職願 と同じコンテキスト）

# ホスト環境用: HTTPS のベース URL（末尾スラッシュなし）
Set Variable [ $base ; Value:
  Case (
    not IsEmpty ( IF::g_TaishokuHtmlUrl ) ; IF::g_TaishokuHtmlUrl ;
    "https://happycare-dev.github.io/taishokunegai/taishoku-negai.html"
  )
]

Set Variable [ $json ; Value:
  JSONSetElement ( "" ;
    [ "resignDate"   ; GetAsText ( IF::g_Date_1 ) ; JSONString ] ;
    [ "currentDate"  ; GetAsText ( Get ( CurrentDate ) ) ; JSONString ] ;
    [ "employeeName" ; 入社登録_EmployeeM::氏名 ; JSONString ] ;
    [ "companyName"  ; 入社登録_EmployeeM_BranchM_CompanyM::会社名 ; JSONString ] ;
    [ "repTitle"     ; 入社登録_EmployeeM_BranchM_CompanyM::代表者役職名 ; JSONString ] ;
    [ "repName"      ; 入社登録_EmployeeM_BranchM_CompanyM::代表者氏名 ; JSONString ]
  )
]

# JSON → UTF-8 バイナリ（コンテナ）→ Base64（1 引数）
Set Variable [ $data ; Value: Base64Encode ( TextEncode ( $json ; "UTF-8" ; 1 ) ) ]

# Base64Encode は 76 文字ごとに改行を入れる → URL が壊れるので除去
Set Variable [ $data ; Value: Substitute ( $data ; [ ¶ ; "" ] ; [ Char ( 10 ) ; "" ] ; [ Char ( 13 ) ; "" ] ) ]

# 代替: 改行なし Base64（2 引数のみ）
# Set Variable [ $data ; Value: Base64EncodeRFC ( 4648 ; $json ) ]

# TextEncode が使えない場合の代替（クエリパラメータ直渡し・Base64 不要）:
# Set Variable [ $url ; Value:
#   $base & "?" &
#   "resignDate=" & GetAsText ( IF::g_Date_1 ) & "&" &
#   "currentDate=" & GetAsText ( Get ( CurrentDate ) ) & "&" &
#   "employeeName=" & 入社登録_EmployeeM::氏名 & "&" &
#   "companyName=" & 入社登録_EmployeeM_BranchM_CompanyM::会社名 & "&" &
#   "repTitle=" & 入社登録_EmployeeM_BranchM_CompanyM::代表者役職名 & "&" &
#   "repName=" & 入社登録_EmployeeM_BranchM_CompanyM::代表者氏名 & "&" &
#   "print=1"
# ]
# ↑ この場合は下の $url 行をコメントアウト

Set Variable [ $url ; Value: $base & "?data=" & $data & "&print=1" ]

Exit Script [ Result: $url ]
```

**親スクリプト `P｜退職願` 側:**

```filemaker
Perform Script [ "P｜退職願｜URL構築" ]
Set Variable [ $url ; Value: Get ( ScriptResult ) ]
Set Web Viewer [ Object Name: "wvTaishoku" ; URL: $url ]
```

`Get ( ScriptResult )` で戻り値を受け取るのが確実です。サブスクリプト単体でテストした場合、**スクリプト終了後**は `$url` などの `$` 変数は消えます（Data Viewer で空に見えるのは正常）。終了前に確認するか、`Get ( ScriptResult )` を使ってください。

`IF::g_TaishokuHtmlUrl` を使わない場合のデフォルトは GitHub Pages URL です。

**フィールド対応（既存レイアウト `P｜退職願` のマージフィールドと同じ）:**

| HTML キー | FileMaker フィールド |
|-----------|---------------------|
| `resignDate` | `IF::g_Date_1` |
| `currentDate` | `Get(CurrentDate)` |
| `employeeName` | `入社登録_EmployeeM::氏名` |
| `companyName` | `入社登録_EmployeeM_BranchM_CompanyM::会社名` |
| `repTitle` | `入社登録_EmployeeM_BranchM_CompanyM::代表者役職名` |
| `repName` | `入社登録_EmployeeM_BranchM_CompanyM::代表者氏名` |

日付は `GetAsText` で渡せば、HTML 側で `2026/06/25` 形式を漢数字（二〇二六年 六月二十五日）に変換します。FileMaker 側で既に漢数字整形済みの文字列を渡す場合はそのまま表示されます。

## 5. `P｜退職願` の iOS 分岐（既存スクリプトをそのまま使う）

**別スクリプト `P｜退職願｜iPad` は不要です。** すでに `P｜退職願` 内で `If [ isiOS ]` 分岐があるなら、そこに Web Viewer 処理を足すだけで足ります。

追加が必要なのは次の 2 点だけです。

| 作るもの | 必須？ |
|---------|--------|
| レイアウト `P｜退職願｜iPad`（Web Viewer `wvTaishoku`） | **はい** |
| サブスクリプト `P｜退職願｜URL構築` | 推奨（URL 組み立てを分離） |
| スクリプト `P｜退職願｜iPad` | **不要**（`P｜退職願` に統合済み） |

### `P｜退職願` の iOS 側（こう直す）

```filemaker
# 2026.07 iPadの対応
If [ isiOS ]
	Go to Related Record [
		Show only related records ;
		From table: "入社登録_EmployeeM" ;
		Using layout: "P｜退職願｜iPad" (入社登録_EmployeeM)
	]
	Perform Script [ "P｜退職願｜URL構築" ]
	Set Variable [ $url ; Value: Get ( ScriptResult ) ]
	Set Web Viewer [ Object Name: "wvTaishoku" ; URL: $url ]
	Pause [ Indefinitely ]
	Close Window [ Current Window ]
Else
	Go to Related Record [
		Show only related records ;
		From table: "入社登録_EmployeeM" ;
		Using layout: "P｜退職願" (入社登録_EmployeeM)
	]
	Print Setup [ Restore ; With dialog: Off ]
	Adjust Window [ Resize to Fit ]
	Perform Script [ "設定｜プリンタ指定 ｜たて" ]
	Enter Browse Mode [ Pause: Off ]
	Close Window [ Current Window ]
End If
```

**ポイント:** iOS 側では `Print Setup` / プリンタ指定を**実行しない**。PC 側だけ従来どおり印刷。

`P｜退職願｜URL構築` も作りたくなければ、同じ `$base` / `$json` / `$url` の計算を `If [ isiOS ]` ブロック内に直接書いても構いません。

## 6. 動作確認

### ローカル（レイアウト調整用）

```bash
open /Users/anh/Documents/Kaizen/Taishoku/taishoku-negai.html
```

サンプルデータが入った状態で縦書きを確認できます。

### GitHub Pages（本番と同じ経路）

```
https://happycare-dev.github.io/taishokunegai/taishoku-negai.html?resignDate=2026/06/25&currentDate=2026/06/25&employeeName=永井　みゆき&companyName=株式会社CLOVERワークス&repTitle=代表取締役&repName=野口　潔
```

### FileMaker Go（iPad）での確認

1. サーバーにホストされた `InformationBoard` に接続
2. 退職願作成画面から「退職願を印刷」
3. Go 分岐により Web Viewer が HTTPS URL を読み込むこと
4. 縦書きが PC 印刷と同等に見えること

## 7. iPad での印刷

FileMaker Go の Web Viewer では **`print=1` の自動印刷は iOS の制限で動かない**ことが多いです。**ユーザーのタップ**が必要です。

### 方法 A: HTML の「印刷」ボタン（推奨）

画面下に **印刷** ボタンが表示されます。タップ → iOS の印刷シート → AirPrint / PDF 保存。

GitHub Pages に最新 HTML を反映後、そのまま使えます。

### 方法 B: FileMaker レイアウトにボタンを置く

`P｜退職願｜iPad` レイアウトに **印刷** ボタンを追加:

```filemaker
Perform JavaScript in Web Viewer [
  Object Name: "wvTaishoku" ;
  Function Name: "printLetter" ;
  Parameters: ""
]
```

Web Viewer の下に配置すると押しやすいです。

### 方法 C: 印刷後に閉じる

`Pause [ Indefinitely ]` の代わりに、レイアウトに **閉じる** ボタン:

```filemaker
Close Window [ Current Window ]
```

### 印刷設定（iOS / ブラウザ）

- 用紙: **A4 縦**
- **ヘッダーとフッター: オフ**（URL・日付が入ると本文がはみ出します）
- 倍率: 100%（はみ出す場合は「ページに合わせる」）

## 8. トラブルシューティング

| 症状 | 対処 |
|------|------|
| Web Viewer が真っ白 | HTTPS URL が iPad から到達可能か（ブラウザで同 URL を開く）。証明書エラーがないか |
| `file://` が使われている | ホストファイルでは不可。`FILEMAKER-SETUP.md` の HTTPS 構成に切り替える |
| `$data` に改行が入る | `Substitute` で `¶` を除去してから `$url` を組み立て |
| `$url` が空 / null | サブスクリプト終了後は `$` 変数が消える。親で `Get ( ScriptResult )` を使う。サブスクリプト末尾に `Exit Script [ Result: $url ]` |
| `Base64Encode` は 1 引数のみ | `TextEncode` でコンテナにしてから `Base64Encode` |
| `TextEncode` は 3 引数 | 3 番目は `1`（改行をそのまま） |
| URL が長すぎてデータが切れる | `?data=` を `#` フラグメントに変更、またはフィールドを減らす |
| 縦書きが崩れる | FileMaker レイアウトではなく **HTML** が表示されているか確認（Go 分岐） |
| 印刷ダイアログが出ない | `print=1` パラメータ、または `printLetter()` を手動実行 |
| 開発と本番で URL が違う | `IF::g_TaishokuHtmlUrl` で上書き（通常は GitHub Pages 固定で不要） |
| HTML を更新したのに反映されない | `git push` 後、GitHub Pages の反映に数分かかることがある |
