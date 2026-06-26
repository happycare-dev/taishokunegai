# 退職願 — iPad（FileMaker Go）セットアップ

FileMaker Pro の印刷レイアウト `P｜退職願` は Sideways（`-fm-tategaki`）で縦書きを実現していますが、**FileMaker Go は Sideways 非対応**のため iPad では文字が横向きに崩れます。

本フォルダの `taishoku-negai.html` を Web Viewer で表示し、CSS `writing-mode: vertical-rl` で正しい縦書きを再現します。**PC / FileMaker Pro では従来の `P｜退職願` レイアウト印刷をそのまま使います。**

## ホスト環境について（重要）

`InformationBoard` は **FileMaker Server でホスト**されています。iPad の FileMaker Go はサーバー上のファイルに接続するだけなので、**`file:///` や各端末の `Get(DocumentsPath)` は使えません**（HTML は iPad ローカルに存在しません）。

Web Viewer の HTML は **HTTPS で配信される 1 か所**に置き、全クライアントが同じ URL を参照する構成にします。

```
iPad (FileMaker Go) ──接続──▶ FileMaker Server (InformationBoard)
        │
        └── Web Viewer ──HTTPS GET──▶ taishoku-negai.html（サーバーまたは Web サーバー）
```

## 1. HTML の配置（サーバー側）

`taishoku-negai.html` を HTTPS で公開できる場所にデプロイします。

### 方法 A: FileMaker Server 付属 Web サーバー（推奨）

FileMaker Server マシンの `HTTPServer/htdocs` に配置します。

| OS | 配置先（例） |
|----|-------------|
| macOS | `/Library/FileMaker Server/HTTPServer/htdocs/Taishoku/taishoku-negai.html` |
| Windows | `C:\Program Files\FileMaker\FileMaker Server\HTTPServer\htdocs\Taishoku\taishoku-negai.html` |

アクセス URL の例（開発環境）:

```
https://dev.happycare.host/Taishoku/taishoku-negai.html
```

※ 実際のホスト名・パス・SSL 設定はサーバー管理者に合わせてください。本番は `happycare.host` など別ホストになる場合があります。

### 方法 B: 既存の Web サーバー

nginx / Apache / CDN など、社内で既に使っている HTTPS 配信先に `taishoku-negai.html` を置いても構いません。

### デプロイ後の確認

ブラウザで HTML が開けることを確認してから FileMaker 側を設定します。

```bash
# 例（ホスト名は環境に合わせる）
curl -I https://dev.happycare.host/Taishoku/taishoku-negai.html
```

### ベース URL の管理（FileMaker 内）

環境ごとに URL が変わる場合、`IF` テーブルにグローバルフィールドを 1 つ追加することを推奨します。

| フィールド例 | 用途 |
|-------------|------|
| `IF::g_TaishokuHtmlUrl` | `https://dev.happycare.host/Taishoku/taishoku-negai.html` など |

開発 / 本番の切り替えは、既存の `Get(HostName)` 判定（`dev.happycare.host` など）と同様の `Case` で初期値をセットできます。

## 2. FileMaker 側の変更概要

| 項目 | 内容 |
|------|------|
| 既存レイアウト `P｜退職願` | **変更しない**（PC 印刷用） |
| 新規レイアウト `P｜退職願｜iPad` | Web Viewer 1 つのみ（全画面） |
| スクリプト `P｜退職願` | 起動時に Go / iPad なら Web Viewer 経路へ分岐 |

## 3. 新規レイアウト `P｜退職願｜iPad`

- テーブル: `入社登録_EmployeeM`（既存 `P｜退職願` と同じ）
- レイアウトサイズ: 画面いっぱい（例: 768 × 1024）
- オブジェクト: **Web Viewer** 1 つ（オブジェクト名: `wvTaishoku`）
- URL はスクリプトで `Set Web Viewer` により設定（下記）

## 4. カスタム関数（JSON エスケープ）

既存の `Substitute` ベースでも構いません。JSON を使う場合の例:

```filemaker
// 名前: FM_JSONEscape
// 引数: text
Substitute ( text ;
  [ "\"" ; "\\\"" ] ;
  [ "\\" ; "\\\\" ] ;
  [ ¶ ; "\\n" ] ;
  [ Char ( 9 ) ; "\\t" ]
)
```

## 5. スクリプト `P｜退職願｜URL構築`（新規）

退職願 HTML に渡す URL を組み立てます。戻り値を `$url` に格納します。

```filemaker
# 前提: 入社登録_EmployeeM レコード上で実行（既存 P｜退職願 と同じコンテキスト）

# ホスト環境用: HTTPS のベース URL（末尾スラッシュなし）
# IF::g_TaishokuHtmlUrl に格納するか、Case でホスト名から組み立てる
Set Variable [ $base ; Value:
  Case (
    not IsEmpty ( IF::g_TaishokuHtmlUrl ) ; IF::g_TaishokuHtmlUrl ;
    Get ( HostName ) = "dev.happycare.host" ;
      "https://dev.happycare.host/Taishoku/taishoku-negai.html" ;
    "https://happycare.host/Taishoku/taishoku-negai.html"   // 本番ホスト名に合わせて変更
  )
]

Set Variable [ $json ; Value:
  "{" &
  "\"resignDate\":\"" & FM_JSONEscape ( GetAsText ( IF::g_Date_1 ) ) & "\"," &
  "\"currentDate\":\"" & FM_JSONEscape ( GetAsText ( Get ( CurrentDate ) ) ) & "\"," &
  "\"employeeName\":\"" & FM_JSONEscape ( 入社登録_EmployeeM::氏名 ) & "\"," &
  "\"companyName\":\"" & FM_JSONEscape ( 入社登録_EmployeeM_BranchM_CompanyM::会社名 ) & "\"," &
  "\"repTitle\":\"" & FM_JSONEscape ( 入社登録_EmployeeM_BranchM_CompanyM::代表者役職名 ) & "\"," &
  "\"repName\":\"" & FM_JSONEscape ( 入社登録_EmployeeM_BranchM_CompanyM::代表者氏名 ) & "\"" &
  "}"
]

Set Variable [ $data ; Value: Base64EncodeRFC ( 4648 ; $json ; 1 ) ]

# クエリが長すぎる場合は # フラグメントに変更（?data= → #）
Set Variable [ $url ; Value: $base & "?data=" & $data & "&print=1" ]
```

`IF::g_TaishokuHtmlUrl` を使わない場合は、`$base` の `Case` に実際の本番ホスト名を追記してください。

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

## 6. スクリプト `P｜退職願｜iPad`（新規）

```filemaker
Allow User Abort [ Off ]
Freeze Window
Commit Records/Requests [ No dialog ]

New Window [
  Style: Card ;
  Name: "退職願を印刷" ;
  Using layout: "P｜退職願｜iPad" (入社登録_EmployeeM) ;
  Close: Yes ; Minimize: No ; Maximize: No ; Resize: No ;
  Menu Bar: No ; Dim parent window: Yes ; Toolbars: No
]

Go to Related Record [
  From table: "入社登録_EmployeeM" ;
  Using layout: "P｜退職願｜iPad" (入社登録_EmployeeM) ;
  Show only related records
]

Perform Script [ "P｜退職願｜URL構築" ]
Set Web Viewer [ Object Name: "wvTaishoku" ; URL: $url ]

# print=1 付き URL で自動的に印刷ダイアログが開きます。
# 自動印刷が不要な場合は URL から &print=1 を外してください。

Pause [ Indefinitely ]   # ユーザーが印刷完了後に閉じるまで（任意）
Close Window [ Current Window ]
```

## 7. 既存スクリプト `P｜退職願` への分岐追加

スクリプト先頭（`Commit Records` の後）に追加:

```filemaker
If [
  PatternCount ( Lower ( Get ( ApplicationVersion ) ) ; "go" ) > 0
  or Get ( Device ) = 4
]
  Perform Script [ "P｜退職願｜iPad" ]
  Exit Script [ Text Result: "" ]
End If

# 以下、既存の PC 用印刷処理（New Window → P｜退職願 → Print …）をそのまま残す
```

| 判定 | 意味 |
|------|------|
| `ApplicationVersion` に `Go` | FileMaker Go |
| `Get(Device) = 4` | iPad |

## 8. 動作確認

### ローカル（レイアウト調整用）

```bash
open /Users/anh/Documents/Kaizen/Taishoku/taishoku-negai.html
```

サンプルデータが入った状態で縦書きを確認できます。

### サーバー配置後（本番と同じ経路）

デプロイした HTTPS URL をブラウザで開き、クエリでデータを渡して確認します。

```
https://dev.happycare.host/Taishoku/taishoku-negai.html?resignDate=2026/06/25&currentDate=2026/06/25&employeeName=永井　みゆき&companyName=株式会社CLOVERワークス&repTitle=代表取締役&repName=野口　潔
```

### FileMaker Go（iPad）での確認

1. サーバーにホストされた `InformationBoard` に接続
2. 退職願作成画面から「退職願を印刷」
3. Go 分岐により Web Viewer が HTTPS URL を読み込むこと
4. 縦書きが PC 印刷と同等に見えること

## 9. 印刷について

- HTML は A4 縦 (`@page { size: A4 portrait }`) を想定しています。
- `?print=1` 付きで読み込むと、表示後に `window.print()` で iOS 印刷シートを開きます。
- 手動印刷: Web Viewer 表示中にスクリプトから  
  `Perform JavaScript in Web Viewer [ Object Name: "wvTaishoku" ; Function Name: "printLetter" ; Parameters: "" ]`

## 10. トラブルシューティング

| 症状 | 対処 |
|------|------|
| Web Viewer が真っ白 | HTTPS URL が iPad から到達可能か（ブラウザで同 URL を開く）。証明書エラーがないか |
| `file://` が使われている | ホストファイルでは不可。`FILEMAKER-SETUP.md` の HTTPS 構成に切り替える |
| 文字化け | JSON は `Base64EncodeRFC ( 4648 ; … ; 1 )`（UTF-8）を使用 |
| URL が長すぎてデータが切れる | `?data=` を `#` フラグメントに変更、またはフィールドを減らす |
| 縦書きが崩れる | FileMaker レイアウトではなく **HTML** が表示されているか確認（Go 分岐） |
| 印刷ダイアログが出ない | `print=1` パラメータ、または `printLetter()` を手動実行 |
| 開発と本番で URL が違う | `IF::g_TaishokuHtmlUrl` または `Get(HostName)` の `Case` で切り替え |
