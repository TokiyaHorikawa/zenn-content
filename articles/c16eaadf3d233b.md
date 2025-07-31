---
title: "「UIとContainerを分けるのはもう古い？」Hooks時代に再定義された責務分離の実践"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "typescript", "frontend", "hooks", "architecture"]
published: false
publication_name: globis
---

Container-Presenter Pattern や Presentational and Container Components って聞いたことありますか？
また、Composition って何かイメージできますか？
最近あまり聞かなくなった言葉ってありますよね。
今回は古くなってきた？考え方を今の時代に適用していい感じにやれるんじゃね？という記事です。

実行犯: Claude Code (sonnet 4)
計画班: https://x.com/horikawatokiya

Agentic writing で記事を書くことに Try しました。
※AI による無駄記事量産では無いです。また、ちょっと過剰な表現は「AI さんがやりました」と言わせてもらいます。
SOW.md を作って、指示して計画を立て、H2, H3 の見出し単位で細かく指示を出すスタイルで書くことでそれっぽく意図どおりになることが分かって良かったです。

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

Hooks が便利だからって、全部を一つのコンポーネントに詰め込むとどうなるか。実際に見てみましょう：

```tsx
// ユーザー管理ページ
const UserManagementPage = () => {
  // 👎 UI層なのに全部のロジックを持ってる
  const [users, setUsers] = useState([]);
  const [selectedUser, setSelectedUser] = useState(null);
  const [isEditModalOpen, setIsEditModalOpen] = useState(false);
  // ... 省略

  // データ取得、検索フィルタリング処理など
  // ... 省略

  return (
    <div>
      <SearchBox value={searchQuery} onChange={setSearchQuery} />
      <UserList users={filteredUsers} onEditClick={handleEditClick} />
      {/* 👎 さらに深い入れ子が始まる */}
      <UserEditModal
        isOpen={isEditModalOpen}
        user={selectedUser}
        onClose={() => setIsEditModalOpen(false)}
      />
    </div>
  );
};
```

一見、何も問題なさそうですが...

### 子コンポーネントでも同じパターンが繰り返される

```tsx
// UserEditModal内でも同じことが起きる
const UserEditModal = ({ isOpen, user, onClose }) => {
  // 👎 またここでもロジックが複雑化
  const [formData, setFormData] = useState({});
  const [validationErrors, setValidationErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  // フォーム送信処理、バリデーションなど
  // ... 省略

  return (
    <Modal isOpen={isOpen}>
      <UserForm
        initialData={user}
        onSubmit={handleSubmit}
        errors={validationErrors}
        isSubmitting={isSubmitting}
      />
    </Modal>
  );
};
```

### さらに UserForm 内でも...

```tsx
// container/ui/container/ui/container/ui... の入れ子地獄
const UserForm = ({ initialData, onSubmit, errors, isSubmitting }) => {
  // 👎 またここでもロジック...
  const [name, setName] = useState(initialData.name);
  const [email, setEmail] = useState(initialData.email);
  // ... 省略

  // バリデーション処理など
  // ... 省略

  return <form onSubmit={handleSubmit}>{/* フォームの中身 */}</form>;
};
```

### 何が問題なのか？

1. **テストが複雑**：

   - UserManagementPage の VRT を取りたいだけなのに、API モック、データ準備が必要
   - 子コンポーネントの振る舞いまで制御する必要がある

2. **UI 層が見た目に集中できない**：

   - ページレイアウトを確認したいのに、ビジネスロジックが邪魔をする
   - Storybook でデザインレビューするにも、全ての依存関係を準備する必要

3. **再利用が困難**：
   - 同じような UI でも、ロジックが絡んでいるため別の場面で使いにくい
   - A/B テストで異なる処理を注入したくても、コンポーネント自体を分岐する必要

この「入れ子地獄」から抜け出すには、どうすればいいのでしょうか？

## 解決策：UI/Container + Composition + Hooks の組み合わせ

### 4 つの要素による責務分離

先ほどの入れ子地獄を、4 つの要素で整理してみましょう：

**1. UI 層（.ui.tsx）**：純粋な見た目とインタラクション
**2. Container 層（.container.tsx）**：データ取得とロジック統合
**3. Composition**：外部からのコンポーネント合成・注入
**4. Hooks（useXxx.ts）**：再利用可能なビジネスロジック

### 実際にリファクタリングしてみる

#### 1. Hooks でロジックを切り出し

```tsx
// useUserManagement.ts
export const useUserManagement = () => {
  const [users, setUsers] = useState([]);
  const [searchQuery, setSearchQuery] = useState("");

  // データ取得
  useEffect(() => {
    fetchUsers().then(setUsers);
  }, []);

  // 検索フィルタリング
  const filteredUsers = useMemo(() => {
    return users.filter(
      (user) =>
        user.name.includes(searchQuery) || user.email.includes(searchQuery)
    );
  }, [users, searchQuery]);

  return {
    users: filteredUsers,
    searchQuery,
    setSearchQuery,
  };
};

// useUserEdit.ts
export const useUserEdit = () => {
  const [selectedUser, setSelectedUser] = useState(null);
  const [isModalOpen, setIsModalOpen] = useState(false);

  const openEditModal = (user) => {
    setSelectedUser(user);
    setIsModalOpen(true);
  };

  const closeEditModal = () => {
    setSelectedUser(null);
    setIsModalOpen(false);
  };

  return {
    selectedUser,
    isModalOpen,
    openEditModal,
    closeEditModal,
  };
};
```

#### 2. UI 層は純粋な見た目のみ

```tsx
// UserManagementPage.ui.tsx
export const UserManagementPageUI = ({
  users,
  searchQuery,
  onSearchChange,
  onEditClick,
  editModalElement, // 👈 Composition：外部から注入
}) => {
  return (
    <div>
      <SearchBox value={searchQuery} onChange={onSearchChange} />
      <UserList users={users} onEditClick={onEditClick} />
      {editModalElement} {/* 👈 入れ子を避けて外部から注入 */}
    </div>
  );
};

// UserEditModal.ui.tsx
export const UserEditModalUI = ({
  isOpen,
  onClose,
  userFormElement, // 👈 Composition：外部から注入
}) => {
  return (
    <Modal isOpen={isOpen} onClose={onClose}>
      {userFormElement} {/* 👈 フォーム部分も外部から注入 */}
    </Modal>
  );
};

// UserForm.ui.tsx - 完全に純粋なフォーム
export const UserFormUI = ({
  name,
  email,
  department,
  onNameChange,
  onEmailChange,
  onDepartmentChange,
  onSubmit,
  errors,
  isSubmitting,
}) => {
  return (
    <form onSubmit={onSubmit}>
      <input value={name} onChange={onNameChange} error={errors.name} />
      <input value={email} onChange={onEmailChange} error={errors.email} />
      {/* ... 省略 */}
      <button type="submit" disabled={isSubmitting}>
        更新
      </button>
    </form>
  );
};
```

#### 3. Container 層でロジックを統合

```tsx
// UserManagementPage.container.tsx
export const UserManagementPage = () => {
  const userManagement = useUserManagement();
  const userEdit = useUserEdit();

  return (
    <UserManagementPageUI
      users={userManagement.users}
      searchQuery={userManagement.searchQuery}
      onSearchChange={userManagement.setSearchQuery}
      onEditClick={userEdit.openEditModal}
      editModalElement={
        <UserEditModalContainer
          isOpen={userEdit.isModalOpen}
          user={userEdit.selectedUser}
          onClose={userEdit.closeEditModal}
        />
      }
    />
  );
};

// UserEditModal.container.tsx
export const UserEditModalContainer = ({ isOpen, user, onClose }) => {
  const userForm = useUserForm(user);

  return (
    <UserEditModalUI
      isOpen={isOpen}
      onClose={onClose}
      userFormElement={
        <UserFormUI
          {...userForm.formData}
          {...userForm.handlers}
          errors={userForm.errors}
          isSubmitting={userForm.isSubmitting}
        />
      }
    />
  );
};
```

### 何が変わったのか？

**Before（入れ子地獄）**:

- 👎 UI 層なのにロジックが混在
- 👎 container/ui/container/ui の深い入れ子
- 👎 子コンポーネントの振る舞いに依存したテスト

**After（4 要素の組み合わせ）**:

- ✅ **UI 層は純粋な見た目のみ**：props を受け取って表示するだけ
- ✅ **入れ子構造の解消**：Composition で外部から注入
- ✅ **ロジックの再利用**：Hooks で切り出したロジックを他でも使用可能
- ✅ **テストの分離**：UI のテストは見た目のみ、ロジックのテストは Hooks のみ

### 私たちのチームルール

この設計を実践するために、チームで決めたルールを紹介します：

| ファイル種別        | 役割             | 命名例                   | やっていいこと                                              | やってはダメなこと                          |
| ------------------- | ---------------- | ------------------------ | ----------------------------------------------------------- | ------------------------------------------- |
| `xxx.ui.tsx`        | 見た目専用       | `UserForm.ui.tsx`        | ・props の表示<br>・UI インタラクション<br>・レイアウト     | ・useState<br>・useEffect<br>・API 呼び出し |
| `xxx.container.tsx` | ロジック統合     | `UserForm.container.tsx` | ・Hooks の呼び出し<br>・UI への props 渡し<br>・Composition | ・直接的な DOM 操作<br>・スタイリング       |
| `useXxx.ts`         | ビジネスロジック | `useUserForm.ts`         | ・状態管理<br>・API 呼び出し<br>・計算処理                  | ・JSX の return<br>・UI 固有の処理          |

#### 重要な原則

**1. UI 層は子の振る舞いを知らない**

```tsx
// ❌ 悪い例：子の状態を気にしている
const PageUI = ({ users }) => {
  return (
    <div>
      <UserList users={users} />
      {/* UserList の内部状態に依存した処理 */}
      {users.some((u) => u.isEditing) && <div>編集中です</div>}
    </div>
  );
};

// ✅ 良い例：必要な情報は外部から注入
const PageUI = ({ users, isAnyUserEditing, editingIndicator }) => {
  return (
    <div>
      <UserList users={users} />
      {editingIndicator} {/* 外部から注入された要素 */}
    </div>
  );
};
```

**2. Composition で入れ子を避ける**

```tsx
// ❌ 悪い例：UI層でコンポーネントを直接入れ子
const ParentUI = () => {
  return (
    <div>
      <Child1>
        <Child2>
          <Child3 />
        </Child2>
      </Child1>
    </div>
  );
};

// ✅ 良い例：外部で合成されたものを受け取って配置
const ParentUI = ({ composedChildElement }) => {
  return (
    <div>
      {composedChildElement} {/* 外部で Child1>Child2>Child3 が合成済み */}
    </div>
  );
};

// Container側で合成
const ParentContainer = () => {
  const composedChild = (
    <Child1Container>
      <Child2Container>
        <Child3Container />
      </Child2Container>
    </Child1Container>
  );

  return <ParentUI composedChildElement={composedChild} />;
};
```

**3. Container は UI とロジックの橋渡し役**

```tsx
// Container の責務は「UI に必要な形でデータを渡すこと」
const UserFormContainer = ({ user }) => {
  const form = useUserForm(user);
  const validation = useFormValidation(form.data);

  return (
    <UserFormUI
      {...form.data}
      {...form.handlers}
      errors={validation.errors}
      onSubmit={form.submit}
    />
  );
};
```

## この構成で得られる効果

### 1. テストの分離と効率化

**UI 層のテスト**は純粋に見た目のみ。API モックや子コンポーネントの振る舞い制御が不要になり、VRT やスナップショットテストが簡単に書けます。

**ロジックのテスト**は Hooks だけをテスト。UI の描画確認不要で、ビジネスロジックに集中できます。テストの実行速度も大幅に向上。

### 2. 開発・デザインレビューの効率化

**Storybook での開発**が劇的に楽になります。UI 層が純粋なので、API 接続やビジネスロジックなしでデザインレビューが可能。エラー状態、ローディング状態なども簡単に再現でき、デザイナーとの協業がスムーズです。

### 3. 柔軟性と再利用性の向上

**A/B テスト**では同じ UI で異なるロジックを注入可能。**環境別処理**も開発環境では詳細ログ、本番環境では最小限のログといった使い分けが簡単です。UI を変更せずにビジネスロジックだけ差し替えられるのは大きなメリット。

### 4. 複雑なビジネスロジックの柔軟な構築

**複数の Hooks の組み合わせ**により、フォーム処理、バリデーション、権限管理、監査ログなどを組み合わせた複雑な処理も整理して構築できます。各 Hooks が独立しているため、組み合わせパターンも自由自在。

### 5. チーム開発の効率化

**並行開発**が可能に。フロントエンドエンジニアは UI 層、バックエンドエンジニアは Hooks 内の API 連携、デザイナーは Storybook での UI レビューを同時進行できます。

**コードレビュー**も効率化。UI の変更は見た目のみ、ロジックの変更はビジネスロジックのみに集中してレビューできるため、レビューポイントが絞りやすくなります。

**新メンバーの学習コスト**も削減。ファイル命名規則が統一され、責務が明確なので、どこに何があるか分かりやすく、変更箇所も特定しやすくなります。

## まとめ：責務分離 + Composition は今でも強力

「UI と Container を分けるのはもう古い？」

**答えは NO です。**

確かに Dan Abramov が 2019 年に「もう分けなくていい」とコメントしたのは事実。でも、それは「Hooks があるから無理に分ける必要はない」という意味であって、「分けること自体が悪い」ではありません。

実際に開発現場で起きていることを見てください：

- VRT を取りたいだけなのに、API モックの準備で時間を取られる
- UI のデザインレビューなのに、ビジネスロジックの理解が必要
- 子コンポーネントの振る舞いを制御するために、テストが複雑化
- A/B テストで異なる処理を試したいのに、コンポーネント全体を分岐する必要

**これらの問題は、Hooks だけでは解決できません。**

必要なのは、Hooks の便利さを活かしつつ、適切な責務分離を行うこと。そして重要なのは、単純な分離ではなく **Composition による外部注入** です。

### 現代的な責務分離の 4 要素

1. **UI 層（.ui.tsx）**：純粋な見た目とインタラクション
2. **Container 層（.container.tsx）**：データ取得とロジック統合
3. **Composition**：外部からのコンポーネント合成・注入
4. **Hooks（useXxx.ts）**：再利用可能なビジネスロジック

この 4 つが組み合わさって、やっと完成します。どれか一つ欠けても、入れ子地獄や責務の曖昧さが残ってしまいます。

### Hooks 時代だからこそ、責務分離が重要

Hooks が便利だからこそ、ついつい一つのコンポーネントに全てを詰め込みがち。でも **「できる」と「やるべき」は別** です。

責務分離 + Composition により：

- テストが書きやすく、保守しやすいコードになる
- チーム開発での並行作業が可能になる
- UI の変更とロジックの変更を独立して行える
- 同じ UI で異なるビジネスロジックを試せる

**これは単なる理想論ではなく、実践で効果が証明された現代的なアーキテクチャパターンです。**

Hooks の登場で「古い」とされた Presentational/Container パターンも、Composition という新しい要素と組み合わせることで、より強力で柔軟な設計手法として進化しています。

UI 層が見た目に集中できる。これだけで、開発体験は劇的に向上します。
