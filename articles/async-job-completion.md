---
title: "ジョブがいつ終わるかは誰にもわかりません。それでも、私は正確に報告したいと考えています。"
emoji: "⏳"
type: "tech"
topics: ["api", "architecture", "backend", "devops"]
published: true
published_at: 2026-05-04 22:00
canonical_url: https://dev.to/hidekimori/nobody-knows-when-a-job-will-finish-id-still-like-to-report-it-accurately-26nn
---

![](https://static.zenn.studio/user-upload/bae47fb3e26a-20260502.png)
ほとんどの非同期APIは、一つのこと、つまりジョブを開始することだけを約束します。それらは `202 Accepted` を返し、ジョブIDを渡して、そこで契約は終了します。残りはあなたの問題です。

私は違うことをします。一つの約束をします。

> ジョブが完了したら、正確にお伝えします。それまでは、再試行を続けます。

それが、私がこれまで出荷してきたすべてのものに対する契約のすべてです。小さく聞こえるかもしれませんが、実際には、それが私が実際に行っている唯一のことです。

---

## 私のシステムのすべてのジョブが共有する形

あなたは私に仕事を渡します。

あなたは待ちます。

私はできる限り再試行します。

完了したら報告します。

それだけです。スキャンされたPDFのOCR、長いドキュメントからの構造化抽出、あるいはXLIFFファイルの翻訳の洗練など、仕事の内容が何であれ、その形は同じです。あなたが私に入力を与え、画面を見守る必要はありません。報告すべき確かな結果が得られた時に、私は戻ってきます。

実際にこれを提供しようとするまでは、当たり前のことのように聞こえます。

---

## なぜ「開始」は「完了」よりも簡単なのか

`202 Accepted`を返すのは簡単です。難しい部分は、その直後から始まります。

実際のジョブでは、次のような事態に直面します。

- 時折503を返すベンダーAPI。理由はなく、ただ時々発生します。
- コアダンプするネイティブバイナリ。2回連続で発生したかと思えば、1週間は何事もなかったりします。
- ゾンビ化するサブプロセス。クラッシュしたわけでも、終了したわけでもなく、ただ機能不全（defunct）の状態。OSがまだそれらを保持しています。
- どこかの何かが書き込んでそのまま忘れてしまった、古いデバッグファイルでいっぱいになるディスク。

もし「開始しました、これがジョブIDです、頑張ってください」とだけ返してそれをAPIと呼ぶなら、あなたは上記のすべてをユーザーに丸投げしていることになります。

私はそれをしたくありません。だから、その作業を内部に引き戻すのです。

---

## コードではどのようになるか

特定のベンダー名は挙げません。重要なのはそこではないからです。重要なのはその形状です。以下のコードは簡略化されたスケッチです。本番環境のバージョンでは、さらに多くのこと（PDFライブラリのバージョンの癖、最初のエンジンがインプットを拒否したときのフォールバックエンジン、デモモードのページ制限、そして「再試行」「スキップ」「停止」を意味するベンダー固有の長いエラーコードのリストなど）を処理しています。生き残るのはその形状です。

私の変換サービスの一つの内部構造のスケッチがこちらです：

```java
public JobResult runJob(Input input) throws Exception {
    for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
        Process child = new ProcessBuilder(
                "java", "-cp", classpath, EngineMain.class.getName())
            .redirectErrorStream(true)
            .start();
        passInputToStdin(child, input);

        long started = System.currentTimeMillis();
        while (child.isAlive()) {
            if (System.currentTimeMillis() - started > MAX_RUNTIME_MS) {
                child.destroyForcibly();
            }
            if (isDefunct(child)) {
                reap(child);
                break;
            }
            sweepStaleCoreFiles(workDir, MAX_CORE_AGE_MS);
            Thread.sleep(POLL_INTERVAL_MS);
        }

        ChildOutcome outcome = readOutcome(child);
        if (outcome.isTransientError()) continue; // retry
        if (outcome.isIrrelevantError()) {
            log.info("irrelevant error, treating as success: {}", outcome);
            return outcome.toSuccessResult();
        }
        if (outcome.hasResult()) return outcome.toResult();
    }
    return JobResult.failedAfterRetries(MAX_RETRIES);
}
```

そこには注目に値する点がいくつかあります。

**`new ProcessBuilder("java", ..., EngineMain.class.getName())`**。 「ライブラリ関数を呼び出す」のでも「SDKを使う」のでもありません。文字通り、別のプロセスから`main`を再実行しています。その理由は、基盤となるエンジンがネイティブの状態では信頼性に欠けるため、プロセスレベルの分離が必要だからです。もしエンジンがクラッシュしても、子プロセスだけが終了します。

**`if (isDefunct(child)) { reap(child); break; }`**。 ネイティブバイナリは常に正常に終了するとは限りません。クラッシュしているわけでもなく、動作しているわけでもない、つまり「固まっている」状態になることがあります。親プロセスはそれに気づき、判断し、クリーンアップしなければなりません。

**`sweepStaleCoreFiles(workDir, MAX_CORE_AGE_MS)`**。 子プロセスが激しくクラッシュすると、OSはコアダンプファイルを出力します。そのファイルは巨大です。掃除をしなければディスクがいっぱいになります。これに巧妙な解決策はありません。ただ掃除するのみです。

**`outcome.isTransientError()` → `continue`**。 ベンダーのエラーの中には、発生したり消えたりするものがあります。解決策は、待ってから再試行することです。再試行しなければ、ユーザーには「失敗」と表示されます。再試行すれば、ユーザーには「少し時間がかかった」と表示されます。私は後者を選びます。

**`outcome.isIrrelevantError()` → ログを記録して成功を返す**。 これは人々を驚かせる部分です。ユースケースによっては、一部のエラーは実際にはエラーではありません。それらはエンジンが発するノイズです。どれがどれであるかを知るには何年もかかりますし、それが実際の製品の大部分を占めています。

これらはどれも洗練されたものではありません。アーキテクチャ図に現れることもありません。すべては「ジョブが送信された」から「ジョブが完了し、結果がここにある」までの隙間に存在しています。

その隙間こそが、私の仕事です。

---

## 私が諦めたこと

低レイテンシは約束しません。できません。私が待っている対象は予測不可能だからです。

ジョブが常に成功することも約束しません。入力が本当に壊れていることもあります。その時は、ふりをするのではなく、正確にその旨を報告します。

部分的な結果のストリーミングも約束しません。安定して返せるものが手に入るまで、ユーザーをループの外に置いておきます。その代償は待ち時間ですが、利点はノイズを見せずに済むことです。

これらのトレードオフは洗練されたものではありません。ただ一貫しているだけです。

---

## 私はこれを設計したのではない。生き残ったのです。

振り返ってみれば、私がこれまでに構築したすべてのジョブ形式のAPIはこのように動作してきました。ある日座り込んで契約を決めたわけではありません。いつもここに辿り着いてしまうのです。

APIが「開始」とだけ言って関心を失うようなものをリリースしようとするたびに、ユーザーは何が起きたのかと尋ねに戻ってきました。だから私は関心を持つようになりました。一時的なエラーをすべてユーザーに表示しようとするたびに、ユーザーは不安になりました。だから私はそれらを吸収するようになりました。クリーンアップをスキップしてジョブを高速化しようとするたびに、ディスクがいっぱいになりました。だから私は掃除を始めるようになったのです。

長年これを繰り返した結果、残ったのはたった一つのルールです。

> 仕事が完了したら、正確に報告します。それまでは、再試行を続けます。

それがあなたのシステムにとって適切な契約かどうかは、正直なところ分かりません。ただ、私がこれまでに生き残ると確信できた唯一のものです。

---

*このシリーズの記事:*
- *[アコーディオン・パターン：私が一つの巨大なLLMプロンプトを書くのをやめた理由](https://zenn.dev/hidekimori/articles/accordion-pattern-llm)*

*この記事の英語版: [Nobody knows when a job will finish. I'd still like to report it accurately.](https://dev.to/hidekimori/nobody-knows-when-a-job-will-finish-id-still-like-to-report-it-accurately-26nn) (dev.to)*