---
title: "そこに理由はあるんか？〜コードを書く前に問いたい「なぜ？」の話〜"
emoji: "🤔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["思考法", "開発手法", "チーム開発"]
published: false
---

## 導入：某CMから始まった問い

某CMの「そこに愛はあるんか？」が、最近頭の中で「理由」に変換される。

最近の開発を振り返ると、コードを書くことが価値の本質ではないと感じることが増えた。本当に必要なのは「なぜこのコードを書くのか？」という問いなのかもしれない。

## 課題：実装が"思考停止"になりがち問題

実務では「仕様をそのまま実装する」場面が多い。気づけば、自分も含めて指示通り動くだけになっていることがある。

その結果、なぜその実装が有効だったのかを学べず、改善もできない状態に陥ってしまう。

## 「なぜ？」の力：コードを思考に変える

「なぜこれを作るのか？」「なぜこの方法を選ぶのか？」

この問いがあると：
- 実装に意図が生まれる
- 学びが残る
- 次回に活かせる判断基準が蓄積される

**コードを書くとは、「なぜ？」への答えを実行すること**なのかもしれない。

## 3つの「なぜ？」：コードを書く前に自分に問うこと

| 問い | 解説 | 例 |
| --- | --- | --- |
| ① なぜやる？ | 目的・背景 | KPI改善？CS工数削減？UX向上？ |
| ② なぜこの方法？ | 手段の理由 | バリデーション強化で早期離脱防止など |
| ③ なぜそう思う？ | 根拠・前提 | 「多くの離脱は入力ミスが原因」と考える理由は？ |

## コード例（仮説なし vs 仮説あり）

### React.js の例：ユーザー管理画面の設計

**🟥 思考停止コピペコード**

「管理画面はこんな感じでしょ」的な思考停止実装。全ての機能を一つのコンポーネントに詰め込む。

```javascript
function UserManagement() {
  // useState地獄の始まり...
  const [users, setUsers] = useState([]);
  const [searchTerm, setSearchTerm] = useState('');
  const [statusFilter, setStatusFilter] = useState('all');
  const [isLoading, setIsLoading] = useState(false);
  const [selectedUsers, setSelectedUsers] = useState([]);
  const [showModal, setShowModal] = useState(false);
  // まだまだ続く...

  // 巨大なfetchロジック
  const fetchUsers = async () => {
    // ローディング、API呼び出し、エラーハンドリング...
    // 50行のコード
  };

  // 複雑な一括操作
  const handleBulkAction = async (action) => {
    // 楽観的更新、エラーハンドリング...
    // 100行のコード
  };

  return (
    <div>
      {/* 検索、フィルター、テーブル、モーダル... */}
      {/* 200行の巨大JSX */}
    </div>
  );
}
```

**✅ 「なぜ？」ありコード（思考プロセスが見える）**

「なぜこの設計にしたのか？」が明確な実装：

```javascript
// なぜこう分けたのか？
// → 「巨大コンポーネントはテストしにくい」
// → 「機能ごとに分ければ再利用できる」
// → 「バグが起きたとき、原因特定が簡単」
```

**UserManagement.jsx（メインコンテナ）**
```javascript
// Container: 状態管理とコーディネートのみに専念
function UserManagement() {
  // 思考：「データ取得ロジックをカスタムフックに隠蔽することで、
  // このコンポーネントは純粋にUIのコーディネートに集中できる」
  const {
    users,
    loading,
    error,
    searchUsers,
    bulkUpdateUsers,
    deleteUser
  } = useUserManagement();
  
  // 思考：「選択状態は複数子コンポーネントで共有されるため、
  // 最小共通祖先（ここ）で管理するのが自然」
  const { selectedUserIds, toggleSelection, clearSelection } = useSelection();

  return (
    <div>
      <UserSearchAndFilter onSearch={searchUsers} />
      <UserBulkActions 
        selectedUserIds={selectedUserIds}
        onBulkAction={bulkUpdateUsers}
        onComplete={clearSelection}
      />
      <UserTable 
        users={users}
        loading={loading}
        selectedUserIds={selectedUserIds}
        onSelectionChange={toggleSelection}
        onDelete={deleteUser}
      />
    </div>
  );
}
```

**UserSearchAndFilter.jsx（検索・フィルタ）**
```javascript
// 検索・フィルタを独立したコンポーネントに
function UserSearchAndFilter({ onSearch }) {
  // 思考：「検索ロジック（debounce、URLクエリ同期）を
  // カスタムフックに隠蔽することで、このコンポーネントは
  // UIの見た目と基本的なイベントハンドリングに集中できる」
  const { filters, updateFilter, debouncedSearch } = useFilters(onSearch);
  
  // 思考：「商品管理、注文管理でも同じ検索UIが使えるよう、
  // ユーザー固有の情報に依存しない汎用的な作りにする」
  return (
    <div>
      <input 
        placeholder="検索..." 
        onChange={e => updateFilter('search', e.target.value)}
      />
      <select onChange={e => updateFilter('status', e.target.value)}>
        <option value="all">全て</option>
        <option value="active">有効</option>
      </select>
    </div>
  );
}
```

**UserTable.jsx（テーブル表示）**
```javascript
// テーブル表示を独立したコンポーネントに
function UserTable({ users, loading, selectedUserIds, onSelectionChange, onDelete }) {
  // 思考：「大量データの表示最適化（仮想化、memo化）は
  // このコンポーネント内で完結させる。
  // 親コンポーネントは表示方法の詳細を知らなくて良い」
  
  if (loading) return <div>読み込み中...</div>;
  
  // 思考：「VirtualizedTableという抽象化により、
  // パフォーマンス改善を他の機能に影響なく行える」
  return (
    <VirtualizedTable
      data={users}
      renderRow={(user) => (
        <UserTableRow 
          key={user.id}
          user={user}
          isSelected={selectedUserIds.includes(user.id)}
          onSelectionChange={onSelectionChange}
          onDelete={onDelete}
        />
      )}
    />
  );
}
```

**hooks/useUserManagement.js（ビジネスロジック）**
```javascript
// カスタムフック: ビジネスロジックを完全に隠蔽
function useUserManagement() {
  // 思考：「API呼び出し、キャッシュ戦略、楽観的更新、
  // エラーハンドリングなどの複雑なロジックを
  // UIコンポーネントから完全に隠蔽する。
  // こうすることで、APIが変わってもUI側は無影響」
  
  // 思考：「このフックだけを独立してテストすることで、
  // ビジネスロジックの正当性を検証できる」
  
  // SWR、React Query、Zustandなど好きなライブラリで実装
  // ...省略
  
  return { users, loading, error, searchUsers, bulkUpdateUsers, deleteUser };
}
```

**hooks/useSelection.js（選択状態管理）**
```javascript
// カスタムフック: 横断的関心事の分離
function useSelection() {
  // 思考：「選択状態の管理は多くの管理画面で共通する機能。
  // 汎用的なフックとして切り出すことで、
  // 商品管理、注文管理でも同じロジックを再利用できる」
  
  // ...省略
  return { selectedUserIds, toggleSelection, clearSelection };
}
```


### 違いは何か？

**思考停止コード**は真似て動かすことが目標だが、**「なぜ？」ありコード**は「なぜその選択をしたのか」が明確。

- どうしてそう書くのが良いか → 明確な理由がある
- 他の選択肢は無かったのか → 検討した跡が残る  
- 優先順位は何なのか → トレードオフが見える
- 思考の基準は何か → チームで共有できる判断軸がある

## 現場での実践：「なぜ？」をチームで共有するには

- PRテンプレに「なぜこの方法を選んだか」の項目を入れる
- スプリント計画時に「今回の開発で検証したいことは？」を共有する
- 実装が終わったあとも、「狙い通りだったか？」をふりかえる場をつくる

## 結論：「なぜ？」があるから学べるし、次がある

コードを書いた成果とは「通った」「動いた」ではなく、「何を学べたか」である。

そしてそれは、「なぜ？」を問わないと得られない。

**「そこに理由はあるんか？」と自分に問うことは、開発者が思考を止めないための最初の一歩**になるのではないだろうか。

## 締め：今日書いたコードに問いを投げてみて

今日書いたコードを、明日読む自分がどう思うか？

「ちゃんと理由を持って書いてるな」と思えるなら、それはもう戦略的な開発と言えるかもしれない。
