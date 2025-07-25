---
title: "そこに仮説はあるんか？〜コードを書く前に立てたい「問い」と、クリティカル・シンキングの話〜"
emoji: "🤔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["思考法", "開発手法", "チーム開発"]
published: false
---

## 導入：某CMから始まった問い

某CMの「そこに愛はあるんか？」が、最近頭の中で「仮説」に変換される。

最近の開発を振り返ると、コードを書くことが価値の本質ではないと感じることが増えた。もしかすると本当に必要なのは「なぜこのコードを書くのか？」という問いと、その裏の仮説なのかもしれない。

## 課題：実装が"思考停止"になりがち問題

実務では「仕様をそのまま実装する」場面が多い。気づけば、自分も含めて指示通り動くだけになっていることがある。

その結果、なぜその実装が有効だったのかを学べず、改善もできない状態に陥ってしまうことがあるのではないだろうか。

## 仮説の力：コードを思考に変える

仮説を立てるとは、「どうすれば目的を達成できるか？」の仮の答えを持つこと。

仮説があると：
- 実装が検証可能になる
- 学びが残る
- 次回に活かせる知見が蓄積される

これこそが、クリティカル・シンキングの中核の一つだと思う。

## クリティカル・シンキングとの接続

クリティカル・シンキングとは「目的は何か？前提は正しいか？他に選択肢はあるか？」を問い直すこと。

コードを書く前の仮説立ては、これらをすべて内包している。

つまり、**コードを書くとは、仮説を通じて戦略的な判断を実行すること**なのかもしれない。

## 3つの問い：コードを書く前に自分に問うこと

| 問い | 解説 | 例（軽め） |
| --- | --- | --- |
| ① なぜやる？ | 目的・背景 | KPI改善？CS工数削減？UX向上？ |
| ② どういう狙い？ | 手段の意図 | バリデーション強化で早期離脱防止など |
| ③ 何を仮定した？ | 検証視点 | 「多くの離脱は入力ミスが原因」と仮定、など |

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
  const [sortBy, setSortBy] = useState('name');
  const [isLoading, setIsLoading] = useState(false);
  const [selectedUsers, setSelectedUsers] = useState([]);
  const [showModal, setShowModal] = useState(false);
  const [editingUser, setEditingUser] = useState(null);
  const [formData, setFormData] = useState({});
  const [currentPage, setCurrentPage] = useState(1);
  const [error, setError] = useState(null);
  const [totalPages, setTotalPages] = useState(1);
  const [isDeleting, setIsDeleting] = useState(false);
  // まだまだ続く...

  // 巨大なfetchロジック
  const fetchUsers = async () => {
    setIsLoading(true);
    setError(null);
    try {
      const response = await fetch(`/api/users?search=${searchTerm}&status=${statusFilter}&sort=${sortBy}&page=${currentPage}`);
      const data = await response.json();
      setUsers(data.users);
      setTotalPages(data.totalPages);
    } catch (err) {
      setError(err.message);
    }
    setIsLoading(false);
  };

  // 複雑な一括操作
  const handleBulkAction = async (action) => {
    setIsDeleting(true);
    // 楽観的更新、エラーハンドリング、ローディング状態管理...
    // 100行のコード...
    setIsDeleting(false);
  };

  // フィルタリングとソート
  const filteredUsers = users.filter(user => 
    user.name.toLowerCase().includes(searchTerm.toLowerCase()) && 
    (statusFilter === 'all' || user.status === statusFilter)
  ).sort((a, b) => {
    // 複雑なソートロジック...
    return a[sortBy].localeCompare(b[sortBy]);
  });

  return (
    <div>
      {/* 検索とフィルター */}
      <div>
        <input 
          value={searchTerm} 
          onChange={e => setSearchTerm(e.target.value)}
          placeholder="ユーザー検索..."
        />
        <select value={statusFilter} onChange={e => setStatusFilter(e.target.value)}>
          <option value="all">全て</option>
          <option value="active">有効</option>
          <option value="inactive">無効</option>
        </select>
        <button 
          onClick={() => handleBulkAction('activate')}
          disabled={selectedUsers.length === 0 || isDeleting}
        >
          一括有効化
        </button>
      </div>

      {/* 巨大なテーブル */}
      <table>
        <thead>
          <tr>
            <th><input type="checkbox" /* 全選択の複雑なロジック */ /></th>
            <th onClick={() => setSortBy('name')}>名前 {sortBy === 'name' && '↓'}</th>
            <th onClick={() => setSortBy('email')}>メール</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          {isLoading ? (
            <tr><td colSpan={4}>読み込み中...</td></tr>
          ) : (
            filteredUsers.map(user => (
              <tr key={user.id}>
                <td><input type="checkbox" checked={selectedUsers.includes(user.id)} /></td>
                <td>{user.name}</td>
                <td>{user.email}</td>
                <td>
                  <button onClick={() => { setEditingUser(user); setShowModal(true); }}>編集</button>
                  <button onClick={() => deleteUser(user.id)}>削除</button>
                </td>
              </tr>
            ))
          )}
        </tbody>
      </table>

      {/* さらに巨大なモーダル */}
      {showModal && (
        <div className="modal">
          <form onSubmit={handleSubmit}>
            <input 
              value={formData.name || ''} 
              onChange={e => setFormData({...formData, name: e.target.value})} 
            />
            <input 
              value={formData.email || ''} 
              onChange={e => setFormData({...formData, email: e.target.value})} 
            />
            {/* 他にも大量のフィールド... */}
            <button type="submit">保存</button>
            <button onClick={() => setShowModal(false)}>キャンセル</button>
          </form>
        </div>
      )}
    </div>
  );
}
```

**✅ 仮説ありコード（深い思考プロセス）**

複数の観点から検討して設計：

```javascript
// Composition思考プロセス：
// 1. 責務分離：「検索とテーブル表示は別の関心事では？」
// 2. 状態の境界：「選択状態は誰が管理すべき？」
// 3. 再利用性：「この検索コンポーネント、商品管理でも使えそう」
// 4. テスタビリティ：「この巨大コンポーネント、どうテストする？」
// 5. パフォーマンス：「1000行のテーブル、全部再レンダリングしてる？」

// 仮説：「Compositionで機能を分離すると、各コンポーネントが
// "ひとつのことだけをうまくやる"ようになり、
// バグの原因特定、パフォーマンス最適化、機能追加が
// 他の部分に影響せずに行える」
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

**仮説なしコード**は真似て動かすことが目標だが、**仮説ありコード**は「なぜその選択をしたのか」が明確。

- どうしてそう書くのが良いか → 明確な理由がある
- 他の選択肢は無かったのか → 検討した跡が残る  
- 優先順位は何なのか → トレードオフが見える
- 思考の基準は何か → チームで共有できる判断軸がある

## 現場での実践：仮説をチームで共有するには

- PRテンプレに「仮説」の項目を入れる
- スプリント計画時に「今回の開発で検証したいことは？」を共有する
- 実装が終わったあとも、「仮説通りだったか？」をふりかえる場をつくる

## 結論：仮説があるから学べるし、次がある

コードを書いた成果とは「通った」「動いた」ではなく、「何を学べたか」である。

そしてそれは、仮説を立てないと得られない。

**「そこに仮説はあるんか？」と自分に問うことは、開発者が思考を止めないための最初の一歩**になるのではないだろうか。

## 締め：今日書いたコードに問いを投げてみて

今日書いたコードを、明日読む自分がどう思うか？

「ちゃんと仮説を持って書いてるな」と思えるなら、それはもう戦略的な開発と言えるかもしれない。
