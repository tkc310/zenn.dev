---
title: "MeltypeとGoogle日本語入力をショートカットキーで切り替える"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["karabiner", "mac", "ime", "meltype", "googlejapaneseinput"]
published: true
---

## 3行サマリー


- 話題のMeltypeとGoogle日本語入力IMEを併用したかった
- Karabiner-Elementsを利用してショートカットキーを設定した (JISキーボード)
- ショートカットキーをコマンドで追加できるようにした


## やりたいこと

[Meltype](https://github.com/yksr-melt/Meltype)はローマ字入力を自動で日本語と英語の単語に変換してくれてタイポなどもよしなに調整してくれるIMEです。
Xやyahooニュースで話題になっていました。

https://x.com/yksr_melt/status/2107061826911703272

https://news.yahoo.co.jp/articles/79b7238bb97fdbe54119cf34d77f652e284c36b1

しかし、使い慣れていないだけだと思いますが、細かい変換などを制御したいケースで普段使っているGoogle日本語入力IMEと切り替えたいシーンが出てきました。
そのため、下記のようにmacのJISキーボードでGoogle日本語入力とMeltypeを切り替えられるようにしました。

| キー | 切り替え先 |
| --- | --- |
| 英数 | Google 日本語入力の英数 |
| かな | Google 日本語入力のひらがな |
| cmd + 英数 | Meltype |

なお、Meltype自体に入力中に細かく制御する方法はあるようです。
https://github.com/yksr-melt/Meltype/blob/main/docs/USAGE.md


## 前提

- Karabiner-Elements をインストール済みであること
- Google 日本語入力と Meltype を入力ソースに追加済みであること

## 入力ソースの ID を調べる

`select_input_source` で切り替え先を指定するには、入力ソースの ID が必要です。
Karabiner-EventViewer の Variables タブを開いた状態で入力ソースを切り替えると、現在の `input_source_id` が表示されるためこれを利用しました。

私の環境での ID は次のとおりでした。

| 入力ソース | input_source_id |
| --- | --- |
| Google 日本語入力（英数） | `com.google.inputmethod.Japanese.Roman` |
| Google 日本語入力（ひらがな） | `com.google.inputmethod.Japanese.base` |
| Meltype | `io.github.yksr-melt.inputmethod.Meltype.Japanese` |

## ルール

Complex Modifications の「Add your own rule」に次の JSON を貼り付けます。

```json
{
  "description": "英数/かな=Google, cmd+英数=Meltype",
  "manipulators": [
    {
      "type": "basic",
      "from": { "key_code": "japanese_eisuu", "modifiers": { "mandatory": ["command"], "optional": ["caps_lock"] } },
      "to": [{ "select_input_source": { "input_source_id": "io.github.yksr-melt.inputmethod.Meltype.Japanese" } }]
    },
    {
      "type": "basic",
      "from": { "key_code": "japanese_eisuu", "modifiers": { "optional": ["caps_lock", "shift"] } },
      "to": [{ "select_input_source": { "input_source_id": "com.google.inputmethod.Japanese.Roman" } }]
    },
    {
      "type": "basic",
      "from": { "key_code": "japanese_kana", "modifiers": { "optional": ["caps_lock", "shift"] } },
      "to": [{ "select_input_source": { "input_source_id": "com.google.inputmethod.Japanese.base" } }]
    }
  ]
}
```

Karabiner-Elements は manipulators を上から順に評価し、最初に一致したものを使います。
cmd + 英数のルールを英数単体のルールより先に書いておくと、cmd を押しているときは Meltype が優先されます。

## GUI を使わずに追加する

前述のjsonをGUIからではなくコマンドで追加できるようにしました。

次のコマンドを実行すると `~/.config/karabiner/karabiner.json` の選択中のプロファイルに上のルールを追加します。

```bash
curl -fsSL https://raw.githubusercontent.com/tkc310/settings/refs/heads/main/meltype/meltype-karabiner.sh | bash
```

:::message
スクリプトの中身が心配な場合は実行前に下記からコードを確認してください。
https://github.com/tkc310/settings/blob/main/meltype/meltype-karabiner.sh
:::

## 適用例

設定を適用すると、Karabiner-Elements の Complex Modifications に次のようにルールが追加されます。

![Karabiner-Elements にルールを適用した画面](/images/meltype-karabiner/meltype.png)
