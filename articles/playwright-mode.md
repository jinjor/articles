---
title: Playwright の各 mode の挙動を正確に理解する
emoji: "🍡"
type: "tech"
topics: ["Playwright"]
publication_name: knowledgework
published: false
---

こんにちは。ナレッジワークの [torii](https://twitter.com/jinjor) です。

E2E テストでお世話になっている Playwright の挙動についてさくっと解説します！

# わかりにくい mode の挙動

Playwright では `describe` がどのように動作するかを決める `mode` を指定することができます。

| mode     |                         |
| :------- | :---------------------- |
| default  | デフォルトの挙動        |
| serial   | `test` を直列に実行する |
| parallel | `test` を並列に実行する |

```ts
test.describe('なんらかのテスト', () => {
  test.describe.configure({
    mode: 'default',
    retries: 3,
  })
  ...
})
```

筆者は最近までこれらの挙動をよく理解しておらず勘で書いていましたが、理解不足ゆえの想定外の挙動が頻発したためちゃんと調べることにしました。

# default, serial, parallel の挙動

それでは早速、各 mode の違いを見ていきます。まずは次の串団子を見てください。

![凡例](/images/playwright-mode/legend.png)

色が付いている処理は実行されることを示しています。

以降の説明では、次の前提で 3 つの mode の挙動の違いを見ていきます。

前提:

- 3 つのテストを実行する
- 2 つ目のテストが必ず失敗する
- 失敗したら一度だけリトライする（retries: 1）

## default

![mode: default](/images/playwright-mode/mode-default.png)

まずは default の挙動です。

worker が 3 つ書かれていますが、並列実行されているわけではなく、上から順番に実行されています。

失敗したテストが再実行されていますが、失敗した `test2` だけではなく `beforeAll` や `beforeEach` も実行されていることに注意が必要です。
`beforeAll` で環境をセットアップすることを想定していると、これは意外な挙動かもしれません。

## serial

![mode: serial](/images/playwright-mode/mode-serial.png)

次に serial の挙動です。

default とは違い、失敗したテストよりも後ろのテストは実行されません。また、テストが失敗すると、その前のテストから全てやり直しになります。

全てのテストが前のテストに依存している前提で考えると、納得の挙動ですね。

## parallel

![mode: parallel](/images/playwright-mode/mode-parallel.png)

最後に parallel の挙動です。

workers （同時実行可能な worker の数）を多くできる場合、すべてのテストを並列に実行することができます。workers が少ない場合、ひとつの worker が複数の test を実行することがあります。workers が 1 の場合は default と同じ挙動になります。

また、ここでも `beforeAll` が worker の数だけ実行されていることに注意が必要です。`beforeAll` を最初に一回だけ実行して、 `test` だけを並列に実行ということはできません（hack は可能かもしれませんが）。

# その他の挙動

## mode 無指定は default と同じではない

mode を指定しない場合は明示的に default を指定した時と同じ挙動になるかと思いきや、 fullyParallel を指定した時の挙動に違いがあります。fullyParallel によって test が並列に変わるのは mode 無指定の場合のみです。

| mode     | fullyParallel |                 |
| :------- | :------------ | :-------------- |
| 無指定   | false         | default の挙動  |
| 無指定   | true          | parallel の挙動 |
| default  | false / true  | default の挙動  |
| serial   | false / true  | serial の挙動   |
| parallel | false / true  | parallel の挙動 |

fullyParallel を使う場合は気をつけましょう。いっそのこと fullyParallel は使わずに全て明示的に指定するのもアリだと思います。

## describe のネストの制約

describe はネストさせることができますが、複数の mode を混在させる時は組み合わせに制約があります。

| parent \ child | default | serial | parallel |
| :------------- | :------ | :----- | :------- |
| default        | ok      | ok     | error    |
| serial         | ok      | ok     | error    |
| parallel       | ok      | ok     | ok       |

parallel の親は parallel でなくてはなりません。「一部分だけを並行に実行するのは無理」と覚えておきましょう。
同様に、トップレベルに parallel とそれ以外の mode を混在させることもできません。

## beforeAll, afterAll, beforeEach, afterEach で失敗した時の挙動

場所を取るので折りたたみますが、 test 以外で落ちた場合の挙動も調べてみました。

:::details beforeAll, afterAll, beforeEach, afterEach で失敗した時の挙動

mode はすべて default です。

### beforeAll で失敗

![fail at beforeAll](/images/playwright-mode/fail-before-all.png)

### afterAll で失敗

![fail at afterAll](/images/playwright-mode/fail-after-all.png)

何故 test3 が再実行されるのかは分かりません。

### beforeEach で失敗

![fail at beforeEach](/images/playwright-mode/fail-before-each.png)

### afterEach で失敗

![fail at afterEach](/images/playwright-mode/fail-after-each.png)

:::

# おわりに

筆者はこの動きをよく理解しておらず、失敗するたびに `beforeAll` が実行されていることに気付いたときは驚きました。
またある時は、 parallel 指定した afterAll の中で「リソースは既に削除されています」という旨のエラーが出ていたのにも悩まされましたが、調べてみると afterAll が何度も実行されていたからでした。

そして、 Playwright のドキュメントを読んでもなかなか理解できず、手当たり次第に実験を繰り返してようやく理解できました。
分かってしまえばそんなに難しくないですね！（もっと早くやればよかったと後悔しています）
