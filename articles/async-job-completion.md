---
title: "ジョブがいつ終わるかは誰にも分からない。それでも正確に報告したい。"
emoji: "⏳"
type: "tech"
topics: ["api", "architecture", "backend", "devops"]
published: true
published_at: 2026-05-04 22:00
canonical_url: https://dev.to/hidekimori/nobody-knows-when-a-job-will-finish-id-still-like-to-report-it-accurately-26nn
---

![](https://static.zenn.studio/user-upload/bae47fb3e26a-20260502.png)
ほとんどの非同期 API は、ひとつのことだけを約束します。「あなたのジョブを開始する」こと。`202 Accepted` を返してジョブ ID を渡したら、契約はそこで終わり。残りはあなたの問題です。

私は少し違うやり方をしています。約束はひとつだけ。

> ジョブが終わったら、正確にお知らせします。それまで、こちらは可能な限りリトライを続けます。

これが、私が今までに作ってきたすべてのものに対する契約のすべてです。小さく聞こえますね。実際のところ、私が本当にやっているのはこれだけです。

---

## 私のシステムのジョブが共有している形

仕事を渡してもらいます。

待ってもらいます。

こちらで可能な限りリトライします。

終わったら報告します。

これだけです。スキャン PDF の OCR でも、長いドキュメントからの構造化抽出でも、XLIFF ファイルの翻訳改善でも — 形は同じです。入力をくれたら、画面を見続ける必要はありません。誠実に報告できるものができたら、こちらから戻ります。

当たり前に聞こえます — 実際にこれを届けようとするまでは。

---

## 「開始した」が「終わった」より簡単な理由

`202 Accepted` を返すのは簡単です。難しいのはその直後から始まります。

実際のジョブはこんなことに当たります。

- ベンダー API が時々 503 を返す。理由はない。たまにそうなる。
- ネイティブバイナリがコアダンプする。2 回連続で吐いた後、1 週間調子よく動く。
- サブプロセスがゾンビ化する。クラッシュもしてない、終わってもいない。Defunct。OS はまだ握ったまま。
- どこかで何かが書いて忘れたデバッグファイルでディスクが埋まる。

「開始しました、ジョブ ID これです、頑張って」と返して、これを API と呼んでしまうと、上記のすべてをユーザーに丸投げすることになります。

私はそれをやりたくありません。なので、仕事を内側に取り戻します。

---

## コードで見るとこうなる

ベンダー名は出しません。重要じゃないので。重要なのは形です。下のコードは簡略化されたスケッチです — 本番版はもっと多くのことを扱っています(PDF ライブラリのバージョン差異、最初のエンジンが入力を拒否したときのフォールバックエンジン、デモモードでのページ数制限、そして「リトライ」「スキップ」「停止」を意味するベンダー固有のエラーコードの長いリスト)。生き残るのは形の方です。

私の変換サービスのひとつの内側を、スケッチで示します。

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

中で指差したい箇所がいくつかあります。

**`new ProcessBuilder("java", ..., EngineMain.class.getName())`。** 「ライブラリ関数を呼ぶ」のではない。「SDK を使う」のでもない。文字通り、別プロセスから `main` に再突入させます。理由は、下回りのエンジンがネイティブの形ではあまりに不安定で、プロセスレベルの隔離を欲しがるからです。死ぬときは子プロセスだけが死ぬ。

**`if (isDefunct(child)) { reap(child); break; }`。** ネイティブバイナリは綺麗に終了するとは限りません。クラッシュもしていないし、走ってもいない — 詰まっています。親プロセスが気づいて、判断して、片付ける必要があります。

**`sweepStaleCoreFiles(workDir, MAX_CORE_AGE_MS)`。** 子プロセスが激しくクラッシュすると、OS はコアファイルを吐きます。これは巨大です。掃除しないと、ディスクが埋まります。賢い解決策はありません。掃除します。

**`outcome.isTransientError()` → `continue`。** 一時的に出てくるベンダーエラーがあります。直し方は、待ってからもう一度やる。やらなければユーザーには `failed` が見えます。やればユーザーには「ちょっと長くかかった」が見える。私は後者を選びます。

**`outcome.isIrrelevantError()` → ログに残して成功扱い。** これは人を驚かせる部分です。エラーじゃないエラーがあります。エンジンが吐いているだけのノイズ。どれがどれかを知るには年単位かかり、それが実際のプロダクトのほとんどです。

このどれもエレガントではありません。アーキテクチャ図にも出てきません。すべては「ジョブが投入された」と「ジョブが終わって、結果はこれです」の間の隙間に住んでいます。

その隙間が、私のやっていることです。

---

## 諦めたもの

低レイテンシは約束しません。できません。私が待っているものは予測できないので。

ジョブが必ず成功するとも約束しません。入力が本当に壊れていることもあります。そういうときは、ふりをするのではなく、正確に報告します。

ストリーミングで部分結果を返すのも約束しません。ユーザーをループから外して、安定したものを返せるまで内側で持っておきます。コストはユーザーが待つことです。利益はユーザーがノイズを見ないことです。

これらのトレードオフは洗練されてはいません。ただ、一貫しています。

---

## 私はこれを設計したわけではない。生き残った形です。

振り返ると、私が今まで作ってきたジョブ型 API はすべてこういう動きをしてきました。ある日座って「契約はこうしよう」と決めたわけではありません。気がつくと、いつもここにたどり着いていました。

`started` と返してそれ以降気にしないものを出荷しようとするたびに、ユーザーが「どうなった?」と戻ってきました。だから気にし始めました。一時的なエラーをすべてユーザーに見せようとするたびに、ユーザーは怖がりました。だから吸収するようになりました。後始末を省いて速くしようとするたびに、ディスクが埋まりました。だから掃除するようになりました。

これだけ年月をかけて残ったのは、ひとつのルールです。

> ジョブが終わったら、正確にお知らせします。それまで、こちらは可能な限りリトライを続けます。

これがあなたのシステムにとって正しい契約かどうかは、本当に分かりません。私が見つけた、生き残るたったひとつの契約というだけです。

---

*このシリーズの記事:*
- *[アコーディオンパターン: ひとつの巨大な LLM プロンプトを書くのをやめた話](https://zenn.dev/hidekimori/articles/accordion-pattern-llm)*

*この記事の英語版: [Nobody knows when a job will finish. I'd still like to report it accurately.](https://dev.to/hidekimori/nobody-knows-when-a-job-will-finish-id-still-like-to-report-it-accurately-26nn) (dev.to)*