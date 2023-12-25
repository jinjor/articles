---
title: Jest から Vitest に移行してみた
emoji: "🚀"
type: "tech"
topics: ["Vitest"]
publication_name: knowledgework
published: false
---

# Jest から Vitest に移行してみた

こんにちは。torii@ナレッジワークです。
7 月にフロントエンドエンジニアとして入社してもうすぐ半年、そろそろ技術記事の一つも書きたいなと思っていたところに、ちょうどいいネタを見つけたので投稿してみます。

やったこととしてはタイトルの通りで、フロントエンドのテストフレームワークを Jest から Vitest に移行しました。理由としては、Jest が CJS を前提として動作しており、ESM 前提のモジュールを動かすのに一手間も二手間もかかるからです。
ナレッジワークのフロントエンドでは、関数単位のテストや UI コンポーネントのテストに Jest を使ってきましたが、それより上のレイヤーになるとたちまちこの ESM の問題を踏み、テストの数が激減していました。そんな時、とあるページでバグが頻発した対策として重点的にテストを書くことになり、この際シュッと Vitest に移行してしまおうということになりました。

# やったこと

## 依存の更新

移行中には色々右往左往しましたが、最終的には次のようになりました。

依存から削除されたパッケージ:

- @storybook/jest
- @types/jest
- jest
- jest-css-modules
- jest-environment-jsdom

依存に追加されたパッケージ:

- @next/env
- @storybook/test
- @vitejs/plugin-react-swc
- jsdom
- node-fetch
- vite-tsconfig-paths
- vitest

直接の依存の数だけ見ると増えていますが、pnpm-lock.yml は `682 additions / 1,109 deletions` という結果なのでだいぶ身軽になったと思います。

## 設定ファイルの更新

Jest の設定を Vitest 用に書き換えました。
コメントを省くと、ざっと以下のようになりました。

vitest.config.mts

```typescript
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

ここは長くなるため、分割して説明してきます。

vitest.setup.mts

```typescript
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
- `getByTestId()` で要素を取得する時の命名規則の設定です

```typescript
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

jsdom で定義されていない値を差し込んでいきます。
ここは Vitest に移行する前からあったコードをそのまま利用しています。

```typescript
// undici fetch が hang するので node-fetch に差し替える
// https://github.com/vitest-dev/vitest/issues/3077
// https://github.com/nodejs/undici/issues/2026
Object.assign(window, { fetch, Response, Request });
```

テスト終了後にプロセスが終了しない問題がしばしば起こったため、その対策です。
[Issue](https://github.com/vitest-dev/vitest/issues/3077) によると、Node.js の fetch 実装 (undich) に問題がありそうとのことで、 `node-fetch` に差し替えています。

```typescript
// Setup MSW
beforeAll(() => mswServer.listen());
afterEach(() => mswServer.resetHandlers());
afterAll(() => mswServer.close());
```

[MSW (Mock Service Worker)](https://github.com/mswjs/msw) の設定です。
API へのリクエストを差し替えられます。便利ですね。

## 関数の更新

Vitest は Jest との互換性をかなり重視した作りになっており、ほとんどの関数を単純な置換で移行することができました。

- `jest.spyOn()` -> `vi.spyOn()`
- `jest.fn()` -> `vi.fn()`
- `jest.mock()` -> `vi.mock()`
- `jest.useFakeTimers()` -> `vi.useFakeTimers()`
- ...

少しだけ違うところは、`jest.requireActual` の代わりに `vi.importActual` を使い、非同期に動的インポートするあたりでしょうか。しかし、大した差分はなく機械的に置き換えられました。

```diff
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

ナレッジワークのフロントエンドでは [scaffdog](https://github.com/scaffdog/scaffdog) というライブラリを愛用してしているため（先日、なんと作者がジョインしています！🎉）、こちらも合わせて書き換えました。

# 速度比較

ローカル実行は Jest と Vitest で体感できるほどの差がありませんでしたが、 CI (GitHub Actions) 上での実行は約 1.5 倍の時間がかかるようになってしまいました。もちろん実行環境やコードベースなどの条件によるので一概に Vitest の方が遅いとは言えませんが、ちょっと残念な結果ですね。

しかし、現状の CI のボトルネックがそこではない（※）ことや、まだ CI の最適化の余地があること、今後 ESM 前提のライブラリが増えていくことなどを勘案した結果、全体としては Vitest に乗っておいた方が良いだろうという判断で OK をもらいました。

※ VRT = Visual Reguression Test の方が遅い

# その他、気づいたこと
