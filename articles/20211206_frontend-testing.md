---
title: "フロントエンドにおけるTDD"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

## フロントエンドでのテスト戦略

## TDD とは

## フロントエンドにおける TDD

## React Testing Library

## サンプル

## メモ [temp]

### フロントエンドでのテストとは

Kent はまず自身のコードベースからテストを以下の様に分類した。

ここでの各用語の意味は以下の様になっている

- E2E テスト
  - 可能な限り（できれば一切）モックを使用せずに、ソフトウェアの振る舞いを検証する
- インテグレーションテスト
  - 複数のコンポーネントを連携させてテストを行う
- 単体テスト
  - 依存関係のないコンポーネントやモックされたコンポーネントを使用してテストを行う

https://twitter.com/kentcdodds/status/960723172591992832

> なぜテスティングトロフィーが考え出されたのか。

テスティングトロフィーはそのまま使用するのではなく、自身のプロジェクトの状況やコードベースのコードをもとに分類していく必要がある。

そして、RTL の指針にも従って記述していく必要がある。つまり、`ユーザーがソフトウェアを使用するようにテストを行う` ことである。

### どのようなテストを書くべき？

テストを書く際に重要なことは、テストを書くことでプロジェクトにバグがないことをどの程度確信できるのかということ。

TypeScript と ESlint を使用することで、静的型つきツールは高い信頼性を誇っているが、ビジネスロジックにバグがないわけではない。

テストカバレッジ 100％は無駄でり、テストする必要のないものもテストするために時間を費やす必要が発生する。

テストの種類にはそれぞれの項目でトレードオフが存在している。E2E により近いテストでは、本物に近いソフトウェアをテストすることができるため、信頼度は高くなる。これはレガシーコード改善ガイドでも示されていることである。

Martin Fawler もテストピラミッドに対して、前提を置いており、高レベルのテストが高速で信頼性があり、変更容易性がある老婆愛には低レベルのテストは不要であると宣言している。

RTL や MSW はこれを実現している。

より多くの結合テストを記述するためには、あまり多くのものをモックすることを避けることである。何かをモックするということは、自分がモックしているものとモックされているものとの間の統合の信頼性を失ったしまうことになる。

モックはあくまでも開発者の期待を記述しているだけであり、実際のソフトウェアの挙動をテストしているわけではない点に注意が必要である。

### MSW

https://kentcdodds.com/blog/stop-mocking-fetch

MSW は全てのリクエストを横取りするモックサーバーを作成し、実際のサーバーと同じように動作する。

実装的には、データベースとしての json ファイルや、faker などのライブラリを使用してレスポンスデータを作成していることが多い。

```ts
import { rest } from "msw"; // msw supports graphql too!
import * as users from "./users";

const handlers = [
  rest.get("/login", async (req, res, ctx) => {
    const user = await users.login(JSON.parse(req.body));
    return res(ctx.json({ user }));
  }),
  rest.post("/checkout", async (req, res, ctx) => {
    const user = await users.login(JSON.parse(req.body));
    const isAuthorized = user.authorize(req.headers.Authorization);
    if (!isAuthorized) {
      return res(ctx.status(401), ctx.json({ message: "Not authorized" }));
    }
    const shoppingCart = JSON.parse(req.body);
    // do whatever other things you need to do with this shopping cart
    return res(ctx.json({ success: true }));
  }),
];

export { handlers };
```

```ts
// test/server.js
import { rest } from "msw";
import { setupServer } from "msw/node";
import { handlers } from "./server-handlers";

const server = setupServer(...handlers);
export { server, rest };
```

```ts
// test/setup-env.js
// add this to your setupFilesAfterEnv config in jest so it's imported for every test file
import { server } from "./server.js";

beforeAll(() => server.listen());
// if you need to add a handler after calling setupServer for some specific test
// this will remove that handler for the rest of them
// (which is important for test isolation):
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

```ts
// __tests__/checkout.js
import * as React from "react";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

test('clicking "confirm" submits payment', async () => {
  const shoppingCart = buildShoppingCart();
  render(<Checkout shoppingCart={shoppingCart} />);

  userEvent.click(screen.getByRole("button", { name: /confirm/i }));

  expect(await screen.findByText(/success/i)).toBeInTheDocument();
});
```

懸念点としては、1 つのファイルとして配置してしまうため、コロケーションの利点がうしなられてしまう可能性がること。

テストしたい場合はテストしたいものだけをコロケーションしたい。なのでハッピーパスはセットファイルに用意して、エッジケースは以下の様にモックすればいい。

```ts
// __tests__/checkout.js
import * as React from "react";
import { server, rest } from "test/server";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

// happy path test, no special server stuff
test('clicking "confirm" submits payment', async () => {
  const shoppingCart = buildShoppingCart();
  render(<Checkout shoppingCart={shoppingCart} />);

  userEvent.click(screen.getByRole("button", { name: /confirm/i }));

  expect(await screen.findByText(/success/i)).toBeInTheDocument();
});

// edge/error case, special server stuff
// note that the afterEach(() => server.resetHandlers()) we have in our
// setup file will ensure that the special handler is removed for other tests
test("shows server error if the request fails", async () => {
  const testErrorMessage = "THIS IS A TEST FAILURE";
  server.use(
    rest.post("/checkout", async (req, res, ctx) => {
      return res(ctx.status(500), ctx.json({ message: testErrorMessage }));
    })
  );
  const shoppingCart = buildShoppingCart();
  render(<Checkout shoppingCart={shoppingCart} />);

  userEvent.click(screen.getByRole("button", { name: /confirm/i }));

  expect(await screen.findByRole("alert")).toHaveTextContent(testErrorMessage);
});
```

### Colocation

https://kentcdodds.com/blog/colocation

メンテナンスしやすいコードを書いていきたい。

そのために責務を意識したフォルダ構成にしたい。つまり変更が発生した場合に、変更するフォルダが 1 箇所で済むような形式にできれば、保守性も向上し、使いやすくもなる。

### Static vs Unit vs Integration vs E2E Testing for Frontend Apps

https://kentcdodds.com/blog/static-vs-unit-vs-integration-vs-e2e-tests

### Testing Implementation Details

### Avoid the Test User

### Should I write a test or fix a bug?

### How to know what to test

### https://zenn.dev/seya/scraps/6f930e359d6a7c

## 参考資料

- [The Testing Trophy and Testing Classifications](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications)
- [Write tests. Not too many. Mostly integration.](https://kentcdodds.com/blog/write-tests)
- [フロントエンドで TDD を実践する（理論編）](https://qiita.com/taneba/items/48db2ad9cf10ad644908)
- [フロントエンドで TDD を実践する（react-testing-library を使った実践編）](https://qiita.com/taneba/items/b21f5fee17eb593b30c8)
- [今日からはじめる React のテスト実践](https://zenn.dev/t_keshi/articles/react-test-practice)
- [React Testing Library の使い方](https://qiita.com/ossan-engineer/items/4757d7457fafd44d2d2f)
- [Common mistakes with React Testing Library](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
