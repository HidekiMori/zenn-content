---
title: "プラグインなしで構築する、動的ドロップダウンを備えたn8nのマルチステップフォーム"
emoji: "🔽"
type: "tech"
topics: ["n8n", "automation", "javascript", "tutorial"]
published: true
published_at: 2026-06-03 22:00
canonical_url: "https://dev.to/hidekimori/build-a-multi-step-n8n-form-with-dynamic-dropdowns-no-plugin-needed-1pm4"
---

![](https://static.zenn.studio/user-upload/09389ecaa4d9-20260524.png)
n8nでマルチステップのフォームを作りたいとします。各ステップのドロップダウンの選択肢は、ユーザーが前のステップで選んだ内容やアップロードしたファイルに依存します。「n8n dynamic dropdown」で検索すると、固定の `fieldOptions` の例や、時折「Codeノードを使ってあとは祈る」といったような情報が見つかるでしょう。

私が運営しているAIドキュメント処理API「[LDX hub](https://gw.portal.ldxhub.io)」の配布用ワークフローを構築していたときも、全く同じ問題に直面しました。私が欲しかったのは、ユーザーが5つのサービスの中から1つを選び、その後エンジン選択、ファイルアップロード、出力形式の選択へと進んでいく単一のワークフローでした。そして、すべてのドロップダウンは前のステップから計算される必要がありました。

結論から言えば、n8nはすでにこの機能をサポートしています。その機能は、Formノードの1つのトグルの裏側に隠されているだけなのです。

この記事では、そのパターンと実際に動くコード、3つの注意点、解きほぐされたリファレンス・ワークフローへのリンクを紹介します。

---

## すべてを変える1つのトグル

n8nのFormノードには **Define Form（フォームの定義）** という設定があります。デフォルトは **Using Fields Below（下のフィールドを使用）** であり、一般的なフォームビルダーのようにUI上でフィールドを追加していきます。しかし、もう一つの選択肢があります。**Using JSON（JSONを使用）** です。

JSONモードでは、フィールド配列全体がn8nの「式（Expression）」になります。あなたはJavaScriptの配列（各要素がフィールド定義を表す）を返すように記述し、その式の中では、これまでのステップのあらゆるデータを参照することができます。

テクニックとしては、それがすべてです。以下で説明するのは、そのスイッチを切り替えた後に「何ができるか」を示すだけのものです。

![Define FormパラメータがUsing JSONに切り替えられ、ドロップダウンのオプションが表示されている様子](https://static.zenn.studio/user-upload/0b404d9124ab-20260524.png)

これから私たちが構築するのは、ワークフローの一つの分岐における以下のチェーンです。

![ExtractDoc path: Form Trigger → HTTP Request → Form (Engine) → Set → Form (File) → Set → Form (Output) → Code → LDX hub Run → Form Ending](https://static.zenn.studio/user-upload/1cf4aa2eb86e-20260524.png)

まずForm TriggerでAPIキーとユーザーが利用したいサービスを収集します。次にHTTP RequestでLDX hubから利用可能なエンジンを取得します。その後、Form / Set / Form / Set / Form という一連のステップによって、前のステップの内容に基づいてユーザーの選択肢を漸進的に絞り込んでいきます。LDX hubノードでジョブを実行し、最後のFormノードで結果を表示します。

---

## 最小限の例：API呼び出しからのドロップダウン

最もシンプルな動的ドロップダウンは、「リストを返すエンドポイントを呼び出し、そのリストを選択肢としてレンダリングする」というものです。

LDX hubの `/extractdoc/engines` エンドポイントは、利用可能な抽出エンジンを認証なしで返します。n8nからこれを呼び出します。

```
HTTP Requestノード
  URL: https://gw.ldxhub.io/extractdoc/engines
```

現在の実際のレスポンスは以下のようになっています。

```json
{
  "data": [
    {
      "id": "ki/extract",
      "display_name": "KI Extract",
      "provider": "ki",
      "description": "Extracts plain text from documents in reading order...",
      "supported_conversions": [
        { "from": "pdf",  "to": "text"  },
        { "from": "pdf",  "to": "jsonl" },
        { "from": "docx", "to": "text"  },
        { "from": "docx", "to": "jsonl" },
        { "from": "xlsx", "to": "text"  },
        { "from": "xlsx", "to": "jsonl" },
        { "from": "pptx", "to": "text"  },
        { "from": "pptx", "to": "jsonl" }
      ]
    }
  ]
}
```

現在は1つのエンジンしかありません。もしプロバイダーが明日エンジンを追加すれば、それは自動的にレスポンスに現れ——これからお見せするように——ドロップダウンにも現れます。

HTTP Requestに続くFormノードは以下のようになります。

```
Formノード
  Define Form: Using JSON
  JSON Output:
    {{ [
      {
        fieldLabel: "Engine",
        fieldName: "engine",
        fieldType: "dropdown",
        fieldOptions: {
          values: $json.data.map(e => ({ option: e.id }))
        },
        requiredField: true
      }
    ] }}
```

この式は、1つのフィールドを持つ配列を返します。フィールドの種類はドロップダウンです。その選択肢（options）は、HTTPレスポンスの `data` 配列をn8nが期待する形（`{ option: "ki/extract" }`）にマッピングすることで計算されます。

![JSON Outputの式エディタ：中央のペインにエンジンのドロップダウン定義、左側に前のノードのデータツリー、右側に結果のプレビュー](https://static.zenn.studio/user-upload/86bcbedc8b70-20260524.png)

フォームがレンダリングされると、ユーザーには唯一の選択肢として `ki/extract` が表示されます。このリストは、先週あなたがハードコードしたものではなく、APIがたった今返してきたものを反映しています。これが、このやり方を採用する「将来を見据えた（future-proofing）」最大の理由です。

---

## もう一段階深く：ユーザーの選択によるフィルタリング

静的なドロップダウンは簡単です。興味深いのは、あるドロップダウンの選択肢が、ユーザーが前のドロップダウンで何を選んだかに依存するケースです。

ユーザーがエンジンを選んだ後、私はそのエンジンがサポートしている入力ファイル形式を計算したいと考えました。その情報はエンジンの `supported_conversions` 配列の中にあります。Setノードでこの導出を行います。

```
Setノード
  Assignments:
    name:  from_options
    type:  array
    value: {{ $('ExtractDoc: Get Engines').item.json.data
              .find(e => e.id === $json.engine)
              .supported_conversions
              .map(c => c.from)
              .filter((v, i, a) => a.indexOf(v) === i) }}
```

`ki/extract` の場合、これは `["pdf", "docx", "xlsx", "pptx"]` を生成します。

この式には3つの注目すべき点があります。

1. **`$('ExtractDoc: Get Engines').item.json`** — これはステップ間参照のためのロングフォーム（長い記述形式）です。ショートフォームである `$json` は、直前のステップのデータしか見ることができません。間に挟まったFormノードを越えて過去のステップに遡るには、このロングフォームが必要です。これは最大の落とし穴（gotcha）であり、後で詳しく説明します。

2. **`$json.engine`** — これは直前のステップのデータであり、ユーザーがたった今選択したエンジンです。

3. **`.filter((v, i, a) => a.indexOf(v) === i)`** — ユニーク化（重複排除）です。エンジンは同じ入力フォーマットを複数の変換ペア（`pdf → text`、`pdf → jsonl`など）にわたって提示するため、ドロップダウンに「pdf」が4回も繰り返されるのは避けたいからです。

次のFormノードは、受け付けるファイルの種類を「選択されたエンジンが実際にサポートしているもの」に制限しつつ、ファイルアップロードフィールドをレンダリングします。

```
Formノード
  Define Form: Using JSON
  JSON Output:
    {{ [
      {
        fieldLabel: "Source File",
        fieldName: "file",
        fieldType: "file",
        acceptFileTypes: "." + $json.from_options.join(",."),
        requiredField: true
      }
    ] }}
```

`acceptFileTypes` は、ブラウザの `<input accept="...">` 属性になります。`from_options = ["pdf", "docx", "xlsx", "pptx"]` の場合、結果の値は `.pdf,.docx,.xlsx,.pptx` となり、ユーザーのファイルピッカーにはこれら4つの種類だけが表示されます。

---

## さらにもう一段階：アップロードされたファイルによる出力のフィルタリング

出力形式のドロップダウンは、エンジンが「一般的に」何をサポートしているかではなく、ユーザーが実際にアップロードしたファイル、つまり「この特定の入力」を何に変換できるかに依存します。

ここで、`$binary` を通じてアップロードされたファイルのメタデータにアクセスします。

```
Setノード
  Assignments:
    name:  to_options
    type:  array
    value: {{ (() => {
      const map = { jpg: 'jpeg', tif: 'tiff' };
      const raw = ($binary.file.fileExtension || '').toLowerCase();
      const from = map[raw] || raw;
      return $('ExtractDoc: Get Engines').item.json.data
        .find(e => e.id === $('ExtractDoc: Select Engine').item.json.engine)
        .supported_conversions
        .filter(c => c.from === from)
        .map(c => c.to)
        .filter((v, i, a) => a.indexOf(v) === i);
    })() }}
```

この式に関するいくつかの注意点：

- **IIFE（即時実行関数）ラッパー** `(() => { ... })()` — ローカル変数（`map`, `raw`, `from`）を使用し、明示的な戻り値を返すために使用しています。n8nの式はJavaScriptであり、有効なJSであれば `{{ }}` の中で動きます。

- **拡張子のマッピング** — `$binary.file.fileExtension` は、ユーザーのファイル名の拡張子をそのまま返します。APIは時折正規化された形式（例えば `jpg` ではなく `jpeg`）を使用するため、フィルターが一致を見逃さないように小さなマップで境界を正規化しています。

- **2つのロングフォーム参照** — エンジンリストを取得するための `$('ExtractDoc: Get Engines')` と、ユーザーの選択を取得するための `$('ExtractDoc: Select Engine')` です。どこでもロングフォームを使います。

PDFをアップロードして `ki/extract` を選択したユーザーの場合、これは `["text", "jsonl"]` に解決されます。これで、出力ドロップダウンは自明なものになります。

```
Formノード
  Define Form: Using JSON
  JSON Output:
    {{ [
      {
        fieldLabel: "Output Format",
        fieldName: "output_format",
        fieldType: "dropdown",
        fieldOptions: {
          values: $json.to_options.map(t => ({ option: t }))
        },
        requiredField: true
      }
    ] }}
```

動的なドロップダウンが表示される前の、レンダリングされたフォームの最初のページでユーザーが見るものは以下の通りです。

![レンダリングされたフォームの最初のページ：APIキー、APIホスト、サービスのドロップダウン。たった今構築したエンジン、ファイル、出力形式のドロップダウンは、次のページ以降に表示される](https://static.zenn.studio/user-upload/3b02e9e5f3a3-20260524.png)

---

## 注意すべき3つのこと

### 1. ショートフォーム `$json` はフォームを越えられない

`$json` は直前のノードの出力を参照します。2つのノードの間にFormノードが挟まった瞬間、`$json` はそのFormノードより前のデータを見ることができなくなります。

解決策は、ロングフォーム `$('Node Name').item.json.field` を使うことです。これなら間にいくつのノードが挟まっていようと、ワークフローのどこからでも機能します。

これをデフォルトの書き方にしてください。ショートフォームで動く場合でもロングフォームを使っておけば、後でFormノードを挿入したくなったときに3つの式を書き直す羽目に陥らずに済みます。

### 2. バイナリデータは明示的にパススルーする必要がある

Formノードはデータを `json` チャネルで返します。ユーザーがアップロードしたファイルは独立した `binary` チャネルに置かれますが、これはSetノード、Switchノード、あるいは後続のFormノードを自動的には伝播（プロパゲート）しません。バイナリデータを実際に消費するノードまで運ぶには、Codeノードを挿入する必要があります。

```javascript
return $input.all().map(item => ({
  json: item.json,
  binary: $('ExtractDoc: Upload File').item.binary
}));
```

これは、何か有用なことを行うCodeノードとしては最小のものです。そのままコピーして使ってください。

### 3. テンプレートとしてエクスポートする際は、`webhookId` フィールドを空にする

Form TriggerとFormノードは、それぞれエクスポートされたJSON内に `webhookId`（UUID）を保持しています。n8nはインポート時にこれを保持し、再生成しません。つまり、あなたのIDがワークフローと一緒にインポーターのインスタンスへと旅してしまうのです。

これはn8nのインポート処理における既知の鋭い刃です（[issue #18683](https://github.com/n8n-io/n8n/issues/18683)）。共有可能なテンプレートの慣例は、公開前にすべてのIDを剥ぎ取ることです。すべての `webhookId` の値を `""` に設定してください。インポートしたインスタンスは、最初の保存時に新しいIDを割り当てます。

これを忘れると、最もよく起こる障害モードは「サイレントなWebhookのハイジャック」です。元のワークフローに向けられた呼び出しがインポートされた側にルーティングされたり、あるいはその逆が起こったりします。

以下のリファレンス・ワークフローは、すべての `webhookId` を空にした状態で提供されています。

---

## なぜこれが重要なのか

デフォルトのn8nテンプレートは、「単一の構成」のワークフローです。あなたは自分の正確なユースケースのためにそれを構築し、保存し、それはあなたのために機能します。それをインポートした他の誰かは、あなたの構成に完全に合わせるか、JSONを手動で編集するかのどちらかになります。

動的ドロップダウンはそれを変えます。同じワークフローが、ユーザー（彼らのファイル、彼らが選んだエンジン、彼らが利用可能な出力形式）に適応するようになるのです。テンプレートは「個人的な成果物」から「小さなアプリケーション」へと変わります。

LDX hubにおいて、これは、私が一切のハードコーディングを行うことなく、1つのワークフローで5つの異なるAIサービスとそれらの有効な構成すべてを公開できることを意味しました。プロバイダーが明日6つ目のエンジンを追加しても、ワークフローは変更されません。「Get Engines」の呼び出しがもう1つ多くのエントリを返し、ドロップダウンがもう1つ多くの選択肢を表示し、フローの残りの部分がそれを処理するだけです。

それが、このテクニックを知っておく価値がある理由です。動的ドロップダウンそのものが面白いからではなく、それが「設定可能なワークフロー」を出荷するためのn8nスタックの一部だからです。

---

## 完全なリファレンス・ワークフロー

5つのサービス、このパターンで駆動するすべてのドロップダウン、さらに動的認証情報（これは別の記事で）を含む完全なデモは、npmパッケージの [n8n-nodes-ldxhub](https://www.npmjs.com/package/n8n-nodes-ldxhub) に `examples/all-services-demo.json` として同梱されています。これをあなたのn8nインスタンスにインポートし、5つのサービスパスのいずれかを調べることで、プロダクションレベルでのこのパターンを確認できます。

もしLDX hubに対してこれを実行してみたい場合は、[gw.portal.ldxhub.io](https://gw.portal.ldxhub.io) で無料のAPIキーを取得してください（月間25,000クレジット、クレジットカード不要）。単にパターンだけが欲しい場合でも、JSONは寛容なライセンスで提供されており、ドロップダウンのロジックはあらゆるAPIに移植可能です。

---

## 終わりに

ドロップダウンのパターンは、「自分のn8nテンプレートを任意のユーザーのために機能させるにはどうすればいいか」という難解に見える問題を、「エンドポイントを呼び出し、そのレスポンスを選択肢としてレンダリングする」というほとんど退屈なものへと変えてくれます。私が長年戦ってきた設定可能性（configurability）に関する複雑さの多くは、適切な境界線を引く場所を見つけた瞬間に、同じように溶解していきました。

面白かったのは、式（Expression）を書いたことではありません。Formノードの設定の中に、そのトグルがずっとそこにあったという発見です。このテクニックが斬新に見えるのは、単にドキュメント化が不足しているからに過ぎません。

もしあなたが、自分自身の正確なセットアップでしか動かないn8nテンプレートを作ってきたなら、設定のノブの1つを、API呼び出しによって駆動するドロップダウンへと移してみてください。そこから、このテクニックが実を結び始めます。