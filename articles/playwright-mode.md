---
title: Playwright の並列処理（mode）の挙動を正確に理解する
emoji: "🍡"
type: "tech"
topics: ["Playwright"]
publication_name: knowledgework
published: true
---

こんにちは。ナレッジワークの [torii](https://twitter.com/jinjor) です！

E2E テストでお世話になっている [Playwright](https://playwright.dev/) の挙動についてさくっと解説します。筆者は最近まで並列処理や mode の挙動をよく理解しておらず勘で書いていましたが、理解不足ゆえの想定外の挙動が頻発したためちゃんと調べてみました。

## 3 種類の mode の挙動

Playwright では `describe` がどのようなフローで動作するかを決める `mode` を指定することができます。

https://playwright.dev/docs/test-parallel

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

ドキュメントの説明を読めばなんとなくわかりますが、リトライ時の挙動や beforeAll などの実行のされ方など細かいところがよく分かりません。そこで、それぞれのパターンの挙動を徹底的に調べて図解してみました。

## default, serial, parallel の挙動

まずは次の串団子（凡例）を見てください。

![凡例](/images/playwright-mode/legend.png)

色が付いている処理は実行されることを示しています。

以降の説明では、次の前提で 3 つの mode の挙動の違いを見ていきます。

**[前提]**

- 3 つのテストを実行する
- 2 つ目のテストが必ず失敗する
- 失敗したら一度だけリトライする（retries: 1）

### default

![mode: default](/images/playwright-mode/mode-default.png)

まずは default の挙動です。

worker が 3 つ書かれていますが、並列実行されているわけではなく、上から順番に実行されています。また、テストが失敗すると、失敗した `test2` から再実行されます。

ここで、失敗した `test2` だけではなく `beforeAll` や `beforeEach` も実行されていることに注意が必要です。`beforeAll` が１回のみの前提でテスト環境をセットアップしていると、思わぬ競合が発生します。

### serial

![mode: serial](/images/playwright-mode/mode-serial.png)

次に serial の挙動です。

default とは違い、テストが失敗すると最初のテストから全てやり直しになります。また、リトライしても最後まで失敗した場合、それ移行のテストは実行されません。

全てのテストが前のテストに依存している前提なので納得の挙動ではありますが、多少おかしくても手っ取り早く全てのテストを実行してほしい時は default の方が良さそうです。

### parallel

![mode: parallel](/images/playwright-mode/mode-parallel.png)

最後に parallel の挙動です。

workers （同時実行可能な worker の数）を多くできる場合、すべてのテストを並列に実行することができます。workers が少ない場合、ひとつの worker が複数の test を実行することがあります。workers が 1 の場合は default と同じ挙動になります。

また、`beforeAll` は worker の数だけ実行されるので、 `beforeAll` を１回実行した後に `test` だけを並列に実行ということはできないようです（hack は可能かもしれませんが）。

## その他の挙動

### mode 無指定は default と同じではない

mode を指定しない場合は明示的に default を指定した時と同じ挙動になるかと思いきや、 playwright.config.ts で [fullyParallel](https://playwright.dev/docs/api/class-testconfig#test-config-fully-parallel) を指定した時の挙動に違いがあります。fullyParallel によって test が並列に変わるのは mode 無指定の場合のみです。

| mode     | fullyParallel |                 |
| :------- | :------------ | :-------------- |
| 無指定   | false         | default の挙動  |
| 無指定   | true          | parallel の挙動 |
| default  | false / true  | default の挙動  |
| serial   | false / true  | serial の挙動   |
| parallel | false / true  | parallel の挙動 |

fullyParallel を使う場合は気をつけましょう。いっそのこと fullyParallel は使わずに全て明示的に指定するのもアリだと思います。

### beforeAll, afterAll, beforeEach, afterEach で失敗した時の挙動

場所を取るので折りたたみますが、 test 以外で落ちた場合の挙動も調べてみました。

:::details beforeAll, afterAll, beforeEach, afterEach で失敗した時の挙動

mode はすべて default です。

#### beforeAll で失敗

![fail at beforeAll](/images/playwright-mode/fail-before-all.png)

#### afterAll で失敗

![fail at afterAll](/images/playwright-mode/fail-after-all.png)

test3 が再実行されている理由はわかりません。

#### beforeEach で失敗

![fail at beforeEach](/images/playwright-mode/fail-before-each.png)

#### afterEach で失敗

![fail at afterEach](/images/playwright-mode/fail-after-each.png)

:::

### describe をネストした時の挙動

describe はネストさせることができますが、複数の mode を混在させる時は組み合わせに制約があります。

| parent \ child | default | serial | parallel  |
| :------------- | :------ | :----- | :-------- |
| default        | ok      | ok     | **error** |
| serial         | ok      | ok     | **error** |
| parallel       | ok      | ok     | ok        |

parallel の親は parallel でなくてはなりません。「一部分だけを並行に実行するのは無理」と覚えておきましょう。同様に、トップレベルに parallel とそれ以外の mode を混在させることもできません。

参考までに、ネストした時の挙動も調べてみました。

:::details describe をネストした時の挙動

子の 2 つ目のテストが必ず失敗する想定です（網掛けの部分が子の describe）。

#### 親: default, 子: default

![親: default, 子: default](/images/playwright-mode/nest-default-default.png)

#### 親: default, 子: serial

![親: default, 子: serial](/images/playwright-mode/nest-default-serial.png)

#### 親: serial, 子: default

![親: serial, 子: default](/images/playwright-mode/nest-serial-default.png)

子は default ですが、親の serial の影響を受けているのでしょうか。

#### 親: serial, 子: serial

![親: serial, 子: serial](/images/playwright-mode/nest-serial-serial.png)

#### 親: parallel, 子: default

![親: parallel, 子: default](/images/playwright-mode/nest-parallel-default.png)

#### 親: parallel, 子: serial

![親: parallel, 子: serial](/images/playwright-mode/nest-parallel-serial.png)

:::

## おわりに

分かってしまえばそんなに難しくないですね！もっと早く調べればよかったと後悔しています。

筆者は最初この動きをよく理解しておらず、２回目の実行で必ず落ちる `beforeAll` や parallel で実行すると必ずエラー出る `afterAll` を書いたりしていました。同じように想定外の挙動で困っている人のお役に立てれば幸いです。

ナレッジワークの E2E テストの活動に興味のある方は以下の記事も併せてどうぞ！

https://zenn.dev/knowledgework/articles/e2e-tenant-pool
