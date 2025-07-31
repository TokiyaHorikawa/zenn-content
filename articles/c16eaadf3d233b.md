---
title: "「UIとContainerを分けるのはもう古い？」Hooks時代に再定義された責務分離の実践"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript", "frontend", "hooks", "architecture"]
published: false
publication_name: globis
---

## 「UI と Container を分けるのはもう古い」って本当？

React 界隈で一度は聞いたことがあるはず。「Presentational/Container パターンはもう古い」という声。

これ、2019 年に React Hooks の提唱者でもある Dan Abramov 自身が[「もう分けなくていい」とコメント](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)したことがきっかけなんです。

> "I don't suggest splitting your components like this anymore."
> (もうこのようにコンポーネントを分割することは推奨しません)

確かに Hooks の登場で、関数コンポーネント内で状態管理ができるようになった。わざわざ Container と UI を分ける必要はない...

**でも、テストを書こうとしたときに気づくんです。**

例えば、ユーザー編集モーダルの VRT（Visual Regression Test）を追加したいとき。純粋に「見た目が崩れてないか」をテストしたいだけなのに：

```tsx
// これをVRTしたいだけなのに...
<UserEditModal isOpen={true} userId="123" />
```

実際はこのコンポーネント内で、ユーザー情報の取得、フォームバリデーション、更新 API 呼び出し、楽観的更新など複数の振る舞いが動いている。

結果として：

- **VRT したいだけ**なのに、API サーバーのモック、認証状態のセットアップが必要
- **見た目の確認**をしたいのに、ビジネスロジックのテストデータ準備で時間を取られる
- 子コンポーネントで起きる副作用を全部制御しないと、テストが不安定になる

「UI 層が見た目に集中できない」の実害がここにあったんです。

## 現実：入れ子地獄で見た目に集中できない UI 層

## 解決策：UI/Container + Composition + Hooks の組み合わせ

### 4 つの要素による責務分離

### 実装例とルール

## この構成で得られる 3 つの効果

## まとめ：責務分離 + Composition は今でも強力
