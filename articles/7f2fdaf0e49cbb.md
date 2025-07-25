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

### React.js の例：責務分離とコンポジション

**🟥 思考停止コピペコード**

「Todoアプリはこんな感じでしょ」的な思考停止実装。他のコードを真似て全ての責務を一つのコンポーネントに詰め込む。

```javascript
function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [newTodo, setNewTodo] = useState('');
  const [filter, setFilter] = useState('all');
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  const [editingId, setEditingId] = useState(null);
  const [editingText, setEditingText] = useState('');

  const addTodo = async () => {
    setIsLoading(true);
    try {
      const response = await fetch('/api/todos', {
        method: 'POST',
        body: JSON.stringify({ text: newTodo })
      });
      const todo = await response.json();
      setTodos([...todos, todo]);
      setNewTodo('');
    } catch (err) {
      setError(err.message);
    }
    setIsLoading(false);
  };

  const filteredTodos = todos.filter(todo => {
    if (filter === 'completed') return todo.completed;
    if (filter === 'active') return !todo.completed;
    return true;
  });

  return (
    <div>
      <input 
        value={newTodo} 
        onChange={e => setNewTodo(e.target.value)}
        placeholder="What needs to be done?"
      />
      <button onClick={addTodo} disabled={isLoading}>
        {isLoading ? 'Adding...' : 'Add Todo'}
      </button>
      {error && <div className="error">{error}</div>}
      
      <div>
        <button onClick={() => setFilter('all')}>All</button>
        <button onClick={() => setFilter('active')}>Active</button>
        <button onClick={() => setFilter('completed')}>Completed</button>
      </div>

      <ul>
        {filteredTodos.map(todo => (
          <li key={todo.id}>
            {editingId === todo.id ? (
              <>
                <input 
                  value={editingText} 
                  onChange={e => setEditingText(e.target.value)}
                />
                <button onClick={() => saveEdit(todo.id)}>Save</button>
              </>
            ) : (
              <>
                <span onClick={() => toggleTodo(todo.id)}>
                  {todo.completed ? '✓' : '○'} {todo.text}
                </span>
                <button onClick={() => startEdit(todo.id, todo.text)}>Edit</button>
                <button onClick={() => deleteTodo(todo.id)}>Delete</button>
              </>
            )}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

**✅ 仮説ありコード（深い思考プロセス）**

複数の観点から検討して設計：

```javascript
// 深い思考プロセス：
// 1. 責務分離：TodoAppが全部やってていい？presentation/container分けるべき？
// 2. テスタビリティ：この状態でロジックのテスト書ける？UIと分離できてる？
// 3. 再利用性：TodoItemの編集ロジック、他でも使えそうだけど取り出せる？
// 4. パフォーマンス：filteredTodos毎回計算してる、useMemo必要？
// 5. 状態管理：親で持つべき状態と子で持つべき状態の境界は？
// 6. アクセシビリティ：キーボード操作、スクリーンリーダー対応は？
// 7. 運用：エラー処理、ローディング状態の一貫性は？

// 仮説：
// 「TodoAppは状態管理とビジネスロジックに専念し、
// 表示ロジックは各コンポーネントに委譲することで、
// テストしやすく、再利用可能で、責務が明確になる」

// Container: ビジネスロジックと状態管理
function TodoApp() {
  const {
    todos,
    loading,
    error,
    addTodo,
    toggleTodo,
    deleteTodo,
    updateTodo
  } = useTodoLogic(); // カスタムフックで分離

  const [filter, setFilter] = useState('all');

  const filteredTodos = useMemo(() => 
    todos.filter(todo => {
      if (filter === 'completed') return todo.completed;
      if (filter === 'active') return !todo.completed;
      return true;
    }),
    [todos, filter]
  );

  return (
    <div>
      <TodoForm onAddTodo={addTodo} loading={loading} />
      {error && <ErrorMessage error={error} />}
      <TodoFilter currentFilter={filter} onFilterChange={setFilter} />
      <TodoList 
        todos={filteredTodos}
        onToggle={toggleTodo}
        onDelete={deleteTodo}
        onUpdate={updateTodo}
      />
    </div>
  );
}

// Presentation: 表示に専念、props経由でデータ受け取り
function TodoForm({ onAddTodo, loading }) {
  const [text, setText] = useState('');

  const handleSubmit = (e) => {
    e.preventDefault();
    if (text.trim()) {
      onAddTodo(text);
      setText('');
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input 
        value={text}
        onChange={e => setText(e.target.value)}
        placeholder="What needs to be done?"
        disabled={loading}
      />
      <button type="submit" disabled={loading || !text.trim()}>
        {loading ? 'Adding...' : 'Add Todo'}
      </button>
    </form>
  );
}

function TodoItem({ todo, onToggle, onDelete, onUpdate }) {
  const [isEditing, setIsEditing] = useState(false);
  const [editText, setEditText] = useState(todo.text);

  const handleSave = () => {
    onUpdate(todo.id, editText);
    setIsEditing(false);
  };

  // 思考：編集状態は各アイテムが持つのが自然
  // なぜなら編集は一時的な状態で、他のコンポーネントには影響しないから

  if (isEditing) {
    return (
      <li>
        <input 
          value={editText}
          onChange={e => setEditText(e.target.value)}
          onBlur={handleSave}
          onKeyPress={e => e.key === 'Enter' && handleSave()}
          autoFocus
        />
      </li>
    );
  }

  return (
    <li>
      <span 
        onClick={() => onToggle(todo.id)}
        className={todo.completed ? 'completed' : ''}
      >
        {todo.completed ? '✓' : '○'} {todo.text}
      </span>
      <button onClick={() => setIsEditing(true)}>Edit</button>
      <button onClick={() => onDelete(todo.id)}>Delete</button>
    </li>
  );
}

function TodoList({ todos, onToggle, onDelete, onUpdate }) {
  return (
    <ul>
      {todos.map(todo => (
        <TodoItem 
          key={todo.id}
          todo={todo}
          onToggle={onToggle}
          onDelete={onDelete}
          onUpdate={onUpdate}
        />
      ))}
    </ul>
  );
}

// カスタムフック: ビジネスロジックを分離してテストしやすく
function useTodoLogic() {
  // 実装...
  // テスト時はこのフックだけをテストすればロジックを検証できる
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
