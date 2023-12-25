---
title: フロントエンドのテスト基盤を Jest から Vitest に移行した話
emoji: "⚡️"
type: "tech"
topics: ["Vitest"]
# publication_name: knowledgework
published: false
---

こんにちは。ナレッジワークの [torii](https://twitter.com/jinjor) です。
7 月にフロントエンドエンジニアとして入社してもうすぐ半年、そろそろ技術記事の一つも書きたいなと思っていたところに、ちょうどいいネタを見つけたので投稿してみます！

# Jest から Vitest に移行してみた

早速やったことですが、フロントエンドのテストフレームワークを [Jest](https://jestjs.io/) から [Vitest](https://vitest.dev/) に移行しました。理由としては、Jest が CJS を前提として動作しており、ESM 前提のモジュールを動かすのに一手間も二手間もかかるからです。

ナレッジワークのフロントエンドは Next.js を採用していることもあり、テストフレームワークに Jest を採用していました。関数単位のテストや UI コンポーネントのテストを書く分には問題なかったのですが、それより上層（ページなど）になるとたちまち ESM 互換性の問題を踏み、テストを書こうにも腰の重い状態が続いていました。そんな時、とあるページでバグが頻発した対策として重点的にテストを書くことになり、この際シュッと Vitest に移行してしまおうということになりました。Vitest は最近 v1.0.0 がリリースされたこともあり、移行するには絶好のタイミングです。

# 移行作業の詳細

ここからは、移行の際に行った具体的な作業を書いていきます。Jest からの移行方法は公式ドキュメントにも[記載](https://vitest.dev/guide/migration.html#migrating-from-jest)があるので基本的にはその通りにやればできるのですが、プロジェクト固有の事情などで公式には書かれていない部分もあり少しつまづく部分もありました。

## 依存の更新

色々右往左往した結果、最終的に追加・削除されたパッケージ次のようになりました。

```diff
- @storybook/jest
- @types/jest
- jest
- jest-css-modules
- jest-environment-jsdom
+ @next/env
+ @storybook/test
+ @vitejs/plugin-react-swc
+ jsdom
+ node-fetch
+ vite-tsconfig-paths
+ vitest
```

直接の依存の数だけ見ると増えていますが、 pnpm-lock.yml の差分を見ると `682 additions / 1,109 deletions` という結果なのでだいぶ身軽になったと思います。

## 設定ファイルの更新

Jest の設定を Vitest 用に書き換えました。
コメントを省くと、ざっと以下のようになりました。

```typescript:vitest.config.mts
import nextEnv from "@next/env";
import react from "@vitejs/plugin-react-swc";
import tsconfigPaths from "vite-tsconfig-paths";
import { defineConfig } from "vitest/config";

nextEnv.loadEnvConfig(process.cwd());

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    globals: true,
    root: "src",
    environment: "jsdom",
    setupFiles: ["./vitest.setup.mts"],
    testTransformMode: {
      ssr: ["**/*"],
    },
    reporters: ["default", "hanging-process"],
  },
});
```

特筆すべき点は以下になります。

- `plugin-react` の代わりに `plugin-react-swc` を使うことでトランスパイルを速くしています
- 元々設定に使っていた `next/jest` をやめたため、`@next/env` を使って自前で環境変数をセットする必要が生じました
- `vite-tsconfig-paths` を使って、インポートのエイリアス設定（`'@/...'`）を tsconfig.json から読み込んでいます
- jsdom より happy-dom の方が高速という記事をいくつかみましたが、一部の動作に不具合があったため見送りました
- `testTransformMode` は Vitest 0.33 -> 0.34 で起きたパフォーマンス劣化（[Issue](https://github.com/vitest-dev/vitest/issues/3967)）の対策です
- `reporters` の `"hanging-process"` は後述するハング対策の時にデバッグのために追加しました

## セットアップファイルの更新

ここは長くなるため、ファイルの中身をいくつかに分割して説明していきます。

```typescript:vitest.setup.mts
import "@testing-library/jest-dom/vitest";
import { loadEnvConfig } from "@next/env";
import { configure } from "@testing-library/react";
import fetch, { Request, Response } from "node-fetch";

import { mswServer } from "@/libs/msw/mswServer";

loadEnvConfig(process.cwd());

configure({
  testIdAttribute: "data-test",
});
```

- `"@testing-library/jest-dom"` の代わりに `"@testing-library/jest-dom/vitest"` を使います
- 設定ファイルと同じく、`@next/env` を使って環境変数を読み込んでいます
- `testIdAttribute` は `getByTestId()` で要素を取得する時の命名規則の設定です

```typescript:vitest.setup.mts
/** @see https://github.com/jsdom/jsdom/issues/1695 */
Element.prototype.scrollIntoView = vi.fn();
Element.prototype.scrollBy = vi.fn().mockReturnValue(undefined);

/** @see https://jestjs.io/docs/manual-mocks#mocking-methods-which-are-not-implemented-in-jsdom */
beforeAll(() => {
  Object.defineProperty(window, "ResizeObserver", {
    writable: true,
    value: vi.fn().mockImplementation(() => ({
      disconnect: vi.fn(),
      observe: vi.fn(),
      unobserve: vi.fn(),
    })),
  });
  // 他にも
});
```

jsdom で定義されていないメソッドやグローバルな値を差し込んでいます。ここは Vitest に移行する前からあったコードをそのまま利用しています。

```typescript:vitest.setup.mts
// undici fetch が hang するので node-fetch に差し替える
// https://github.com/vitest-dev/vitest/issues/3077
// https://github.com/nodejs/undici/issues/2026
Object.assign(window, { fetch, Response, Request });
```

テスト終了後にプロセスが終了しない問題がしばしば起こったため、その対策をしています。[Issue](https://github.com/vitest-dev/vitest/issues/3077) によると、Node.js の fetch 実装 (undich) に問題がありそうとのことで、 `node-fetch` に差し替えています。

```typescript:vitest.setup.mts
// Setup MSW
beforeAll(() => mswServer.listen());
afterEach(() => mswServer.resetHandlers());
afterAll(() => mswServer.close());
```

[MSW (Mock Service Worker)](https://github.com/mswjs/msw) の設定です。API からのレスポンスを差し替えられます。便利ですね。

## 関数の更新

Vitest は Jest との互換性をかなり重視した作りになっており、ほとんどの関数を単純な置換で移行することができました。

- `jest.spyOn()` -> `vi.spyOn()`
- `jest.fn()` -> `vi.fn()`
- `jest.mock()` -> `vi.mock()`
- `jest.useFakeTimers()` -> `vi.useFakeTimers()`
- ...

少しだけ違うところは、`jest.requireActual` の代わりに `vi.importActual` を使い、非同期に動的インポートするあたりでしょうか。しかし、大した差分はなく機械的に置き換えられました。

```diff typescript
- jest.mock('@/libs/react', () => ({
-   ...jest.requireActual('@/libs/react'),
-   useState: jest.fn(),
- }))
+ vi.mock('@/libs/react', async () => ({
+   ...(await vi.importActual('@/libs/react')),
+   useState: vi.fn(),
+ }))
```

## scaffolding を更新

ナレッジワークのフロントエンドでは [scaffdog](https://github.com/scaffdog/scaffdog) というライブラリを愛用してしているため、こちらもあわせて書き換えました。（ちなみに今月 scaffdog 作者の [wadamaru](https://twitter.com/wadackel) がジョインしました 🎉 心強いです！）

## storybook/jest を storybook/jest に移行

時を同じくして [Storybook 7.6 がリリース](https://storybook.js.org/blog/storybook-7-6/)されました。
このリリースにより、`@storybook/test` がそれまでの Jest から Vitest を使うように変更されたため、こちらも一緒に移行しました。

# トレードオフ

Vitest への移行に際して、以下に挙げるようにいくつかのトレードオフがありました。

## トランスパイラの違い

Vitest は Vite プロジェクトでの使用を想定していますが、 Next.js のプロジェクトでも問題なく使用できます。しかし Next.js と Vite ではコードのトランスパイル結果が違うことには留意する必要があるでしょう。Next.js はトランスパイラとして babel を使用し、 Vite はトランスパイラとして esbuild を使用するため、厳密に本番で使用されるコードをテストすることができません。

しかし、今回は微妙な差異を気にするよりもそもそテストが書けないという問題を解決するメリットが大きかったため、無視することにしました。

## 速度

ローカル実行は Jest と Vitest で体感できるほどの差がありませんでしたが、 CI (GitHub Actions) 上での実行は約 1.5 倍の時間がかかるようになってしまいました。もちろん実行環境やコードベースなどの条件によるので一概に Vitest の方が遅いとは言えませんが、ちょっと残念な結果ですね。

しかし、現状の CI のボトルネックがそこではない（※）ことや、まだ CI の最適化の余地があること、今後 ESM 前提のライブラリが増えていくことなどを勘案した結果、全体の判断としては Vitest に乗っておいた方が良いだろうという結論に至りました。

※ VRT = Visual Reguression Test の方が遅い

## その他

`vi.mock()` を使ってモジュールから export される関数をモックする時、ちょっとした注意点があるようです。この点については公式ドキュメントにも[記載](https://vitest.dev/guide/mocking.html#mocking-pitfalls)があります。

例として、次のように同じモジュールで export されている `foo()` をモックに差し替えることはできますが、 `foobar()` の中で使われている `foo()` を差し替えることはできません。

```typescript
export function foo() {
  return "foo";
}
export function foobar() {
  return `${foo()}bar`;
}
```

Jest ではこの問題は起こりません。Jest と Vitest で挙動の差異は、それぞれが内部で使っているトランスパイラが生成するコードの差異によって生じているようです（[参考](https://github.com/vitest-dev/vitest/issues/1329#issuecomment-1129708816)）。
今回の移行ではこの挙動の違いによるトラブルは起きなかったのですが、知らないと思わぬハマり方をしそうなので注意するに越したことはないでしょう。

# まとめ

Jest から Vitest へ無事に移行することができました。移行にかかった期間は調査やレビューを含めて足掛け 3 日ほどです。

ナレッジワークのフロントエンドエンジニアは、ギルドというコミュニティ活動の中で技術的な知見を共有したり雑談・相談を積極的に行っており、今回の移行も「やりたいね、じゃあやりましょう」ですぐに始まり、レビューも沢山もらうことができました。（特に今回は設定周りで [yukita](https://twitter.com/_otofu_square_) の助けを大いに借りました。ありがとうございます！）
現在[エンジニア積極採用中](https://kwork.studio/recruit-engineer)ですので、気になった方は是非カジュアル面談でお会いしましょう！
