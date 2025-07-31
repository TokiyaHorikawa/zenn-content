---
title: "UI/Container分離 × Hooks × Composition で責務を整理してみた話"
emoji: "🏗️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "frontend", "hooks", "architecture", "design-patterns"]
published: false
publication_name: globis
---

最近のプロジェクトで、Reactコンポーネントのテストを書いていて「あれ？」と思うことがありました。

Hooksが普及して、関数コンポーネント内で状態管理やAPI呼び出しができるようになって便利になった一方で、VRT（Visual Regression Test）を書くときや、Storybookでデザインレビューするときに、ちょっとした困りごとが。

「見た目だけテストしたいのに、なんでAPIモックの準備が必要なんだろう...？」

そんなときに思い出したのが、Container/Presentational パターン。
一時期「もう古い」と言われがちでしたが、Hooksと組み合わせることで、意外と使いやすい形になるんじゃないかと思って試してみました。

その実践記録を共有してみます。

## Container/Presentational パターンについて振り返ってみる

React を書いている方なら、Presentational/Container パターンについて聞いたことがあるかもしれません。

2015年頃から広まったこのパターンですが、2019年にReact HooksのコアチームメンバーであるDan Abramovが自身のブログ記事で「現在は推奨していない」と[更新したこと](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0)で、使われる機会が減ってきました。

彼が指摘したのは、Hooksの登場により「関数コンポーネントでも状態管理やライフサイクル処理ができるようになったため、パフォーマンス最適化以外の理由で無理に分ける必要がなくなった」ということでした。

確かにその通りで、Hooksがあれば一つのコンポーネント内で多くのことができるようになりました。

## よく聞く困りごとの例

たとえば、ユーザー編集モーダルのVRT（Visual Regression Test）を追加したい場面を考えてみましょう。

```tsx
// 見た目だけテストしたい
<UserEditModal isOpen={true} userId="123" />
```

このコンポーネントは一見シンプルに見えますが、内部では：
- ユーザー情報のAPI取得
- フォームバリデーション
- 更新API呼び出し
- 楽観的更新

など、複数の処理が動いているとします。

そうすると、純粋に「見た目が崩れていないか」を確認したいだけなのに：

- APIサーバーのモック設定
- 認証状態のセットアップ
- 各種ビジネスロジックのテストデータ準備

これらの準備作業が必要になってしまいます。

Storybookでデザインレビューする場合も同様で、「ちょっとレイアウト確認したいだけなのに、なんでこんなに準備が必要なんだろう...」という声をよく聞きます。

## Hooksを使ったよくある実装例を見てみる

Hooksが便利なので、機能を一つのコンポーネントにまとめることがよくあります。例えばこんな感じです：

```tsx
// ユーザー管理ページ
const UserManagementPage = () => {
  // 状態管理、データ取得、ビジネスロジックなどがここに
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
      {/* ここでさらにコンポーネントがネストされる */}
      <UserEditModal
        isOpen={isEditModalOpen}
        user={selectedUser}
        onClose={() => setIsEditModalOpen(false)}
      />
    </div>
  );
};
```

こういう実装はよく見かけますし、動作もします。

### 子コンポーネントでも同様の構造が続く

```tsx
// UserEditModal内でも同様の構造
const UserEditModal = ({ isOpen, user, onClose }) => {
  // フォームの状態管理、バリデーション、API呼び出しなど
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

### UserForm 内でも同様に

```tsx
// 各コンポーネントで同じような構造が繰り返される
const UserForm = ({ initialData, onSubmit, errors, isSubmitting }) => {
  // フォーム各項目の状態管理
  const [name, setName] = useState(initialData.name);
  const [email, setEmail] = useState(initialData.email);
  // ... 省略

  // バリデーション処理など
  // ... 省略

  return <form onSubmit={handleSubmit}>{/* フォームの中身 */}</form>;
};
```

### この構造で感じたちょっとした不便さ

このような実装でも十分動作しますが、開発していていくつか気になることがありました：

**テストを書くとき**：
- UIの見た目だけテストしたいのに、APIモックの設定が必要
- 子コンポーネントの動作を制御するための準備が多い

**Storybookでデザインレビューするとき**：
- レイアウトだけ確認したいのに、ビジネスロジックの依存関係を準備する必要

**機能を再利用したいとき**：
- 同じUIで異なるビジネスロジックを使いたい場合に対応しにくい

こういったことから、「もしかしてContainer/Presentationalパターンって、今でも使い道あるんじゃない？」と思うようになりました。

## 試してみた提案：UI/Container + Composition + Hooks の組み合わせ

こんな困りごとを解決するために、Container/PresentationalパターンをHooksやCompositionと組み合わせることを試してみました。

### 4つの要素で責務を整理するアイデア

先ほどの実装例を、4つの要素で整理してみることを考えました：

**1. UI層（.ui.tsx）**：純粋な見た目とインタラクション
**2. Container層（.container.tsx）**：データ取得とロジック統合
**3. Composition**：外部からのコンポーネント合成・注入
**4. Hooks（useXxx.ts）**：再利用可能なビジネスロジック

### 実際に書き換えてみた例

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

### こんな変化がありました

**以前の実装**：
- UIとビジネスロジックが同じコンポーネント内に混在
- コンポーネントが深くネストされる構造
- テストで子コンポーネントの動作を制御する必要

**書き換え後**：
- UI層は純粋にpropsを受け取って表示するだけ
- Compositionで外部からコンポーネントを注入する構造
- Hooksで切り出したロジックを他のコンポーネントでも使える
- UIのテストとロジックのテストを分離できる

### 実践するために考えたルール

この設計を試すために、チームで考えたルールを紹介します：

| ファイル種別        | 役割             | 命名例                   | やっていいこと                                              | やってはダメなこと                          |
| ------------------- | ---------------- | ------------------------ | ----------------------------------------------------------- | ------------------------------------------- |
| `xxx.ui.tsx`        | 見た目専用       | `UserForm.ui.tsx`        | ・props の表示<br>・UI インタラクション<br>・レイアウト     | ・useState<br>・useEffect<br>・API 呼び出し |
| `xxx.container.tsx` | ロジック統合     | `UserForm.container.tsx` | ・Hooks の呼び出し<br>・UI への props 渡し<br>・Composition | ・直接的な DOM 操作<br>・スタイリング       |
| `useXxx.ts`         | ビジネスロジック | `useUserForm.ts`         | ・状態管理<br>・API 呼び出し<br>・計算処理                  | ・JSX の return<br>・UI 固有の処理          |

#### 大切にしたポイント

**1. UI 層は子の振る舞いを知らない**

```tsx
// 以前の実装：子の状態を気にしている
const PageUI = ({ users }) => {
  return (
    <div>
      <UserList users={users} />
      {/* UserList の内部状態に依存した処理 */}
      {users.some((u) => u.isEditing) && <div>編集中です</div>}
    </div>
  );
};

// 書き換え後：必要な情報は外部から注入
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
// 以前の実装：UI層でコンポーネントを直接入れ子
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

// 書き換え後：外部で合成されたものを受け取って配置
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

## まとめ：個人的には、まだ使い道があると思っています

Container/Presentational パターンについて「もう古い」という話もありますが、個人的には、Hooks や Composition と組み合わせることで、まだまだ現役で使えるパターンだと感じています。

今回試してみた構成：

**1. UI層（.ui.tsx）**：純粋な見た目とインタラクション
**2. Container層（.container.tsx）**：データ取得とロジック統合  
**3. Composition**：外部からのコンポーネント合成・注入
**4. Hooks（useXxx.ts）**：再利用可能なビジネスロジック

### 感じたメリット

実際に試してみて感じたのは：

- **テストが書きやすくなった**：UIのテストは見た目だけ、ロジックのテストは Hooks だけに集中できる
- **Storybook でのデザインレビューが楽になった**：ビジネスロジックの準備なしで UI を確認できる
- **チーム開発がしやすくなった**：UI 担当とロジック担当で並行作業できる
- **A/B テストに対応しやすくなった**：同じ UI で異なるロジックを試せる

### Hooks 時代だからこそ

Hooks が便利だからこそ、ついつい一つのコンポーネントに全部詰め込みがちです。でも「できる」ことと「やるべき」ことは別かもしれません。

もちろん、全てのプロジェクトでこの構成が最適というわけではありません。チームの規模、プロジェクトの複雑さ、開発期間など、様々な要因を考慮して選択するのが良いと思います。

ただ、「テストが書きにくい」「Storybook の準備が大変」といった困りごとがある場合は、一度試してみる価値があるのではないでしょうか。

UI 層が見た目に集中できるだけで、開発体験はかなり変わります。
