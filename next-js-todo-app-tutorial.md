# 初めてのTODO管理Webアプリケーション開発ガイド

本ガイドでは、開発初心者向けにNext.jsを使用したTODO管理Webアプリケーションの開発から、Cloudflare Workersへのデプロイまでを0から100まで丁寧に解説します。データ保存にはローカルストレージを使用し、データベースサービスは使用しません。

## 目次

1. [環境準備](#1-環境準備)
2. [プロジェクトの初期化](#2-プロジェクトの初期化)
3. [基本的なTODOアプリの作成](#3-基本的なtodoアプリの作成)
4. [ローカルストレージでのデータ管理](#4-ローカルストレージでのデータ管理)
5. [UI改善とスタイリング](#5-ui改善とスタイリング)
6. [Githubでのコード管理](#6-githubでのコード管理)
7. [Cloudflare Workersへのデプロイ](#7-cloudflare-workersへのデプロイ)
8. [トラブルシューティング](#8-トラブルシューティング)

## 1. 環境準備

### 前提条件

このチュートリアルは以下の環境での実装を想定しています：
- **Windows 10/11** (ネイティブまたはWSL2)
- **WSL2 (Windows Subsystem for Linux 2)**
- **Ubuntu 20.04/22.04 LTS**

### 必要なツールのインストール

#### Node.js（最新安定版）

**Windows/WSL2/Ubuntu共通:**

```bash
# Node.js v20 (LTS) をインストール
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# インストールの確認
node --version    # v20.x.x と表示されるはず
npm --version     # 10.x.x と表示されるはず
```

**Windowsの場合（追加選択肢）:**
- [Node.js公式サイト](https://nodejs.org/)から最新LTS版をダウンロードしてインストール

#### Git

```bash
# Ubuntu/WSL2の場合
sudo apt update
sudo apt install git

# 設定
git config --global user.name "あなたの名前"
git config --global user.email "your-email@example.com"

# 確認
git --version
```

#### テキストエディタ

推奨：**Visual Studio Code**
- [VS Code公式サイト](https://code.visualstudio.com/)からダウンロード
- WSL2を使用する場合は「Remote - WSL」拡張機能をインストール

### 開発環境の確認

以下のコマンドで環境が正しく設定されているか確認します：

```bash
# Node.jsのバージョン確認
node --version

# npmのバージョン確認
npm --version

# gitの確認
git --version

# 現在のディレクトリを確認
pwd
```

## 2. プロジェクトの初期化

### Next.jsプロジェクトの作成

作業ディレクトリを作成し、Next.jsプロジェクトを初期化します：

```bash
# プロジェクトディレクトリの作成
mkdir my-todo-app
cd my-todo-app

# Next.jsプロジェクトの初期化（TypeScript + Tailwind CSS）
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
```

**このコマンドの詳細説明:**
- `create-next-app@latest`: 最新版のNext.jsプロジェクトを作成
- `--typescript`: TypeScriptを使用（型安全性の向上）
- `--tailwind`: Tailwind CSSを自動設定（簡単スタイリング）
- `--eslint`: ESLintを設定（コード品質の維持）
- `--app`: App Routerを使用（Next.js 13+の新機能）
- `--src-dir`: srcディレクトリを作成（プロジェクト構造の整理）
- `--import-alias "@/*"`: インポートエイリアスの設定

### プロジェクト構造の確認

作成されたプロジェクト構造を確認します：

```bash
# プロジェクト構造を確認
ls -la

# src内の構造を確認
tree src/ -I node_modules
```

**期待される構造:**
```
my-todo-app/
├── src/
│   └── app/
│       ├── globals.css
│       ├── layout.tsx
│       └── page.tsx
├── public/
├── package.json
├── tailwind.config.ts
├── tsconfig.json
└── next.config.js
```

### 開発サーバーの起動確認

```bash
# 依存関係のインストール（必要に応じて）
npm install

# 開発サーバーの起動
npm run dev
```

ブラウザで `http://localhost:3000` を開き、Next.jsのウェルカムページが表示されることを確認します。

**表示されるべき内容:**
- Next.jsのロゴ
- 「Get started by editing src/app/page.tsx」のメッセージ
- 各種リンク

## 3. 基本的なTODOアプリの作成

### 基本的なコンポーネント構造の設計

TODOアプリに必要な基本コンポーネントを定義します：

1. **Todo型定義** - TypeScriptでデータ構造を定義
2. **TodoList** - TODO一覧を表示
3. **TodoItem** - 個別のTODOアイテム
4. **AddTodo** - 新しいTODOを追加

### 型定義の作成

```bash
# typesディレクトリを作成
mkdir src/types

# Todo型定義ファイルを作成
touch src/types/todo.ts
```

`src/types/todo.ts`:
```typescript
// TODOアイテムの型定義
export interface Todo {
  id: string;          // 一意識別子
  text: string;        // TODO内容
  completed: boolean;  // 完了状態
  createdAt: Date;     // 作成日時
  updatedAt: Date;     // 更新日時
}

// TODOフィルターの型定義
export type TodoFilter = 'all' | 'active' | 'completed';

// TODOアクションの型定義
export interface TodoAction {
  type: 'ADD' | 'UPDATE' | 'DELETE' | 'TOGGLE' | 'CLEAR_COMPLETED';
  payload?: any;
}
```

**型定義の詳細説明:**
- `interface Todo`: TODOアイテムの構造を定義
- `id`: UUIDを使用して一意性を保証
- `createdAt/updatedAt`: データの管理とソートに使用
- `TodoFilter`: フィルタリング機能用の型
- `TodoAction`: 状態管理用のアクション型

### メインページの作成

`src/app/page.tsx`を書き換えます：

```typescript
'use client';

import React, { useState, useEffect } from 'react';
import { Todo, TodoFilter } from '@/types/todo';
import { v4 as uuidv4 } from 'uuid';

export default function Home() {
  // 状態管理
  const [todos, setTodos] = useState<Todo[]>([]);
  const [filter, setFilter] = useState<TodoFilter>('all');
  const [newTodoText, setNewTodoText] = useState('');

  // ローカルストレージからデータを読み込み
  useEffect(() => {
    const savedTodos = localStorage.getItem('todos');
    if (savedTodos) {
      try {
        const parsedTodos = JSON.parse(savedTodos).map((todo: any) => ({
          ...todo,
          createdAt: new Date(todo.createdAt),
          updatedAt: new Date(todo.updatedAt),
        }));
        setTodos(parsedTodos);
      } catch (error) {
        console.error('ローカルストレージからのデータ読み込みエラー:', error);
      }
    }
  }, []);

  // ローカルストレージにデータを保存
  useEffect(() => {
    localStorage.setItem('todos', JSON.stringify(todos));
  }, [todos]);

  // 新しいTODOを追加
  const addTodo = () => {
    if (newTodoText.trim() === '') return;

    const newTodo: Todo = {
      id: uuidv4(),
      text: newTodoText.trim(),
      completed: false,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    setTodos([...todos, newTodo]);
    setNewTodoText('');
  };

  // TODOの完了状態を切り替え
  const toggleTodo = (id: string) => {
    setTodos(todos.map(todo =>
      todo.id === id
        ? { ...todo, completed: !todo.completed, updatedAt: new Date() }
        : todo
    ));
  };

  // TODOを削除
  const deleteTodo = (id: string) => {
    setTodos(todos.filter(todo => todo.id !== id));
  };

  // TODOを編集
  const editTodo = (id: string, newText: string) => {
    setTodos(todos.map(todo =>
      todo.id === id
        ? { ...todo, text: newText, updatedAt: new Date() }
        : todo
    ));
  };

  // 完了済みTODOを一括削除
  const clearCompleted = () => {
    setTodos(todos.filter(todo => !todo.completed));
  };

  // フィルタリングされたTODOリスト
  const filteredTodos = todos.filter(todo => {
    switch (filter) {
      case 'active':
        return !todo.completed;
      case 'completed':
        return todo.completed;
      default:
        return true;
    }
  });

  return (
    <div className="min-h-screen bg-gray-100 py-8">
      <div className="max-w-md mx-auto bg-white rounded-lg shadow-md p-6">
        <h1 className="text-3xl font-bold text-gray-800 mb-6 text-center">
          TODO管理アプリ
        </h1>

        {/* TODO追加フォーム */}
        <div className="mb-6">
          <div className="flex gap-2">
            <input
              type="text"
              value={newTodoText}
              onChange={(e) => setNewTodoText(e.target.value)}
              onKeyPress={(e) => e.key === 'Enter' && addTodo()}
              placeholder="新しいTODOを入力..."
              className="flex-1 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            />
            <button
              onClick={addTodo}
              className="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500"
            >
              追加
            </button>
          </div>
        </div>

        {/* フィルターボタン */}
        <div className="mb-4 flex gap-2 justify-center">
          {(['all', 'active', 'completed'] as TodoFilter[]).map((filterType) => (
            <button
              key={filterType}
              onClick={() => setFilter(filterType)}
              className={`px-3 py-1 rounded-md text-sm ${
                filter === filterType
                  ? 'bg-blue-500 text-white'
                  : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
              }`}
            >
              {filterType === 'all' ? 'すべて' :
               filterType === 'active' ? 'アクティブ' : '完了済み'}
            </button>
          ))}
        </div>

        {/* TODOリスト */}
        <div className="space-y-2 mb-4">
          {filteredTodos.length === 0 ? (
            <p className="text-gray-500 text-center py-4">
              {filter === 'all' ? 'TODOがありません' :
               filter === 'active' ? 'アクティブなTODOがありません' :
               '完了済みのTODOがありません'}
            </p>
          ) : (
            filteredTodos.map((todo) => (
              <TodoItem
                key={todo.id}
                todo={todo}
                onToggle={toggleTodo}
                onDelete={deleteTodo}
                onEdit={editTodo}
              />
            ))
          )}
        </div>

        {/* 統計情報と操作 */}
        <div className="border-t pt-4 text-sm text-gray-600">
          <div className="flex justify-between items-center">
            <span>
              総計: {todos.length}件 | 
              完了: {todos.filter(t => t.completed).length}件 | 
              残り: {todos.filter(t => !t.completed).length}件
            </span>
            {todos.some(t => t.completed) && (
              <button
                onClick={clearCompleted}
                className="text-red-500 hover:text-red-700 text-sm"
              >
                完了済みを削除
              </button>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}

// TODOアイテムコンポーネント
interface TodoItemProps {
  todo: Todo;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
  onEdit: (id: string, newText: string) => void;
}

function TodoItem({ todo, onToggle, onDelete, onEdit }: TodoItemProps) {
  const [isEditing, setIsEditing] = useState(false);
  const [editText, setEditText] = useState(todo.text);

  const handleEdit = () => {
    if (editText.trim() !== '') {
      onEdit(todo.id, editText.trim());
    }
    setIsEditing(false);
  };

  const handleCancel = () => {
    setEditText(todo.text);
    setIsEditing(false);
  };

  return (
    <div className={`flex items-center gap-3 p-3 border rounded-md ${
      todo.completed ? 'bg-gray-50 opacity-75' : 'bg-white'
    }`}>
      {/* 完了チェックボックス */}
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
        className="w-4 h-4 text-blue-600 rounded focus:ring-blue-500"
      />

      {/* TODO内容 */}
      <div className="flex-1">
        {isEditing ? (
          <div className="flex gap-2">
            <input
              type="text"
              value={editText}
              onChange={(e) => setEditText(e.target.value)}
              onKeyPress={(e) => {
                if (e.key === 'Enter') handleEdit();
                if (e.key === 'Escape') handleCancel();
              }}
              className="flex-1 px-2 py-1 border border-gray-300 rounded text-sm focus:outline-none focus:ring-1 focus:ring-blue-500"
              autoFocus
            />
            <button
              onClick={handleEdit}
              className="px-2 py-1 bg-green-500 text-white rounded text-xs hover:bg-green-600"
            >
              保存
            </button>
            <button
              onClick={handleCancel}
              className="px-2 py-1 bg-gray-500 text-white rounded text-xs hover:bg-gray-600"
            >
              キャンセル
            </button>
          </div>
        ) : (
          <span
            className={`${
              todo.completed ? 'line-through text-gray-500' : 'text-gray-800'
            } cursor-pointer`}
            onDoubleClick={() => setIsEditing(true)}
          >
            {todo.text}
          </span>
        )}
      </div>

      {/* 操作ボタン */}
      {!isEditing && (
        <div className="flex gap-1">
          <button
            onClick={() => setIsEditing(true)}
            className="px-2 py-1 text-blue-500 hover:text-blue-700 text-sm"
          >
            編集
          </button>
          <button
            onClick={() => onDelete(todo.id)}
            className="px-2 py-1 text-red-500 hover:text-red-700 text-sm"
          >
            削除
          </button>
        </div>
      )}
    </div>
  );
}
```

### 必要な依存関係のインストール

UUIDライブラリをインストールします：

```bash
# UUID生成ライブラリのインストール
npm install uuid
npm install --save-dev @types/uuid
```

**UUIDライブラリの説明:**
- `uuid`: 一意識別子（UUID）を生成するライブラリ
- TODOアイテムのIDとして使用
- `v4`は完全にランダムなUUIDを生成

### 動作確認

```bash
# 開発サーバーの起動（まだ起動していない場合）
npm run dev
```

ブラウザで `http://localhost:3000` を開き、以下の機能が動作することを確認します：

1. **TODO追加**: テキスト入力後、「追加」ボタンまたはEnterキーでTODOが追加される
2. **完了切り替え**: チェックボックスクリックで完了状態が切り替わる
3. **編集機能**: TODOテキストをダブルクリックで編集モードに入る
4. **削除機能**: 「削除」ボタンでTODOが削除される
5. **フィルタリング**: 「すべて」「アクティブ」「完了済み」でフィルタリング
6. **データ永続化**: ページをリロードしてもデータが保持される

## 4. ローカルストレージでのデータ管理

### ローカルストレージの仕組み

ローカルストレージはブラウザに内蔵されたデータ保存機能で、以下の特徴があります：

**ローカルストレージの特徴:**
- **永続性**: ブラウザを閉じてもデータが保持される
- **容量**: 通常5-10MBまでデータを保存可能
- **同期**: 同一ドメインの異なるタブ間でデータが共有される
- **セキュリティ**: 同一オリジンからのみアクセス可能

**データベースとの比較:**
| 項目 | ローカルストレージ | データベース |
|------|-------------------|-------------|
| 設定コスト | なし | サーバー設定が必要 |
| 容量 | 制限あり（5-10MB） | 大容量対応 |
| 共有 | 同一ブラウザのみ | 複数ユーザー間で共有 |
| バックアップ | ユーザー依存 | 自動バックアップ可能 |
| 本番運用 | 個人用途向け | エンタープライズ向け |

### ローカルストレージ管理の改善

データ管理を改善するため、専用のユーティリティを作成します：

```bash
# utilsディレクトリを作成
mkdir src/utils

# ローカルストレージユーティリティを作成
touch src/utils/localStorage.ts
```

`src/utils/localStorage.ts`:
```typescript
import { Todo } from '@/types/todo';

// ローカルストレージのキー定数
const STORAGE_KEY = 'todo-app-data';

// ローカルストレージヘルパー関数
export class LocalStorageManager {
  // TODOデータを保存
  static saveTodos(todos: Todo[]): void {
    try {
      const serializedData = JSON.stringify(todos);
      localStorage.setItem(STORAGE_KEY, serializedData);
    } catch (error) {
      console.error('ローカルストレージへの保存エラー:', error);
      throw new Error('データの保存に失敗しました');
    }
  }

  // TODOデータを読み込み
  static loadTodos(): Todo[] {
    try {
      const serializedData = localStorage.getItem(STORAGE_KEY);
      if (!serializedData) {
        return [];
      }

      const parsedData = JSON.parse(serializedData);
      
      // 日付オブジェクトの復元
      return parsedData.map((todo: any) => ({
        ...todo,
        createdAt: new Date(todo.createdAt),
        updatedAt: new Date(todo.updatedAt),
      }));
    } catch (error) {
      console.error('ローカルストレージからの読み込みエラー:', error);
      return [];
    }
  }

  // データを削除
  static clearTodos(): void {
    try {
      localStorage.removeItem(STORAGE_KEY);
    } catch (error) {
      console.error('ローカルストレージのクリアエラー:', error);
      throw new Error('データの削除に失敗しました');
    }
  }

  // データのエクスポート（バックアップ用）
  static exportTodos(): string {
    try {
      const todos = this.loadTodos();
      return JSON.stringify(todos, null, 2);
    } catch (error) {
      console.error('データエクスポートエラー:', error);
      throw new Error('データのエクスポートに失敗しました');
    }
  }

  // データのインポート（復元用）
  static importTodos(jsonData: string): void {
    try {
      const todos: Todo[] = JSON.parse(jsonData);
      
      // データ検証
      const validatedTodos = todos.map(todo => ({
        id: todo.id || crypto.randomUUID(),
        text: todo.text || '',
        completed: Boolean(todo.completed),
        createdAt: new Date(todo.createdAt || Date.now()),
        updatedAt: new Date(todo.updatedAt || Date.now()),
      }));

      this.saveTodos(validatedTodos);
    } catch (error) {
      console.error('データインポートエラー:', error);
      throw new Error('データのインポートに失敗しました');
    }
  }

  // ストレージサイズの確認
  static getStorageSize(): number {
    try {
      const data = localStorage.getItem(STORAGE_KEY);
      return data ? new Blob([data]).size : 0;
    } catch (error) {
      console.error('ストレージサイズ取得エラー:', error);
      return 0;
    }
  }

  // ブラウザサポート確認
  static isSupported(): boolean {
    try {
      const testKey = '__localStorage_test__';
      localStorage.setItem(testKey, 'test');
      localStorage.removeItem(testKey);
      return true;
    } catch (error) {
      return false;
    }
  }
}

// カスタムフック用のヘルパー
export const useLocalStorage = () => {
  return {
    saveTodos: LocalStorageManager.saveTodos,
    loadTodos: LocalStorageManager.loadTodos,
    clearTodos: LocalStorageManager.clearTodos,
    exportTodos: LocalStorageManager.exportTodos,
    importTodos: LocalStorageManager.importTodos,
    getStorageSize: LocalStorageManager.getStorageSize,
    isSupported: LocalStorageManager.isSupported,
  };
};
```

### カスタムフックの作成

React Hooksを使用してローカルストレージの操作を抽象化します：

```bash
# hooksディレクトリを作成
mkdir src/hooks

# カスタムフックを作成
touch src/hooks/useTodos.ts
```

`src/hooks/useTodos.ts`:
```typescript
import { useState, useEffect } from 'react';
import { Todo, TodoFilter } from '@/types/todo';
import { LocalStorageManager } from '@/utils/localStorage';
import { v4 as uuidv4 } from 'uuid';

export const useTodos = () => {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [filter, setFilter] = useState<TodoFilter>('all');
  const [isLoading, setIsLoading] = useState(true);

  // 初期データの読み込み
  useEffect(() => {
    try {
      const savedTodos = LocalStorageManager.loadTodos();
      setTodos(savedTodos);
    } catch (error) {
      console.error('TODOデータの読み込みに失敗しました:', error);
    } finally {
      setIsLoading(false);
    }
  }, []);

  // データ変更時の自動保存
  useEffect(() => {
    if (!isLoading) {
      try {
        LocalStorageManager.saveTodos(todos);
      } catch (error) {
        console.error('TODOデータの保存に失敗しました:', error);
      }
    }
  }, [todos, isLoading]);

  // TODO追加
  const addTodo = (text: string): void => {
    if (text.trim() === '') return;

    const newTodo: Todo = {
      id: uuidv4(),
      text: text.trim(),
      completed: false,
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    setTodos(prevTodos => [...prevTodos, newTodo]);
  };

  // TODO更新
  const updateTodo = (id: string, updates: Partial<Omit<Todo, 'id' | 'createdAt'>>): void => {
    setTodos(prevTodos =>
      prevTodos.map(todo =>
        todo.id === id
          ? { ...todo, ...updates, updatedAt: new Date() }
          : todo
      )
    );
  };

  // TODO削除
  const deleteTodo = (id: string): void => {
    setTodos(prevTodos => prevTodos.filter(todo => todo.id !== id));
  };

  // 完了状態切り替え
  const toggleTodo = (id: string): void => {
    updateTodo(id, {
      completed: !todos.find(todo => todo.id === id)?.completed,
    });
  };

  // テキスト編集
  const editTodo = (id: string, newText: string): void => {
    if (newText.trim() === '') return;
    updateTodo(id, { text: newText.trim() });
  };

  // 完了済みTODOを一括削除
  const clearCompleted = (): void => {
    setTodos(prevTodos => prevTodos.filter(todo => !todo.completed));
  };

  // 全データクリア
  const clearAllTodos = (): void => {
    setTodos([]);
    LocalStorageManager.clearTodos();
  };

  // フィルタリング済みTODOリスト
  const filteredTodos = todos.filter(todo => {
    switch (filter) {
      case 'active':
        return !todo.completed;
      case 'completed':
        return todo.completed;
      default:
        return true;
    }
  });

  // 統計情報
  const stats = {
    total: todos.length,
    completed: todos.filter(todo => todo.completed).length,
    active: todos.filter(todo => !todo.completed).length,
  };

  return {
    // データ
    todos,
    filteredTodos,
    filter,
    stats,
    isLoading,

    // アクション
    addTodo,
    updateTodo,
    deleteTodo,
    toggleTodo,
    editTodo,
    clearCompleted,
    clearAllTodos,
    setFilter,

    // ユーティリティ
    exportTodos: LocalStorageManager.exportTodos,
    importTodos: LocalStorageManager.importTodos,
    getStorageSize: LocalStorageManager.getStorageSize,
  };
};
```

### メインコンポーネントの更新

カスタムフックを使用してメインコンポーネントを簡素化します。

`src/app/page.tsx`を更新:
```typescript
'use client';

import React, { useState } from 'react';
import { TodoFilter } from '@/types/todo';
import { useTodos } from '@/hooks/useTodos';
import TodoItem from '@/components/TodoItem';
import TodoFilter as TodoFilterComponent from '@/components/TodoFilter';
import TodoStats from '@/components/TodoStats';

export default function Home() {
  const {
    filteredTodos,
    filter,
    stats,
    isLoading,
    addTodo,
    toggleTodo,
    deleteTodo,
    editTodo,
    clearCompleted,
    setFilter,
  } = useTodos();

  const [newTodoText, setNewTodoText] = useState('');

  const handleAddTodo = () => {
    addTodo(newTodoText);
    setNewTodoText('');
  };

  if (isLoading) {
    return (
      <div className="min-h-screen bg-gray-100 flex items-center justify-center">
        <div className="text-center">
          <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-500 mx-auto mb-4"></div>
          <p className="text-gray-600">読み込み中...</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-100 py-8">
      <div className="max-w-md mx-auto bg-white rounded-lg shadow-md p-6">
        <h1 className="text-3xl font-bold text-gray-800 mb-6 text-center">
          TODO管理アプリ
        </h1>

        {/* TODO追加フォーム */}
        <div className="mb-6">
          <div className="flex gap-2">
            <input
              type="text"
              value={newTodoText}
              onChange={(e) => setNewTodoText(e.target.value)}
              onKeyPress={(e) => e.key === 'Enter' && handleAddTodo()}
              placeholder="新しいTODOを入力..."
              className="flex-1 px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            />
            <button
              onClick={handleAddTodo}
              className="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500"
            >
              追加
            </button>
          </div>
        </div>

        {/* フィルターコンポーネント */}
        <TodoFilterComponent
          currentFilter={filter}
          onFilterChange={setFilter}
        />

        {/* TODOリスト */}
        <div className="space-y-2 mb-4">
          {filteredTodos.length === 0 ? (
            <p className="text-gray-500 text-center py-4">
              {filter === 'all' ? 'TODOがありません' :
               filter === 'active' ? 'アクティブなTODOがありません' :
               '完了済みのTODOがありません'}
            </p>
          ) : (
            filteredTodos.map((todo) => (
              <TodoItem
                key={todo.id}
                todo={todo}
                onToggle={toggleTodo}
                onDelete={deleteTodo}
                onEdit={editTodo}
              />
            ))
          )}
        </div>

        {/* 統計情報コンポーネント */}
        <TodoStats
          stats={stats}
          onClearCompleted={clearCompleted}
        />
      </div>
    </div>
  );
}
```

### コンポーネントの分離

再利用可能なコンポーネントを作成します：

```bash
# componentsディレクトリを作成
mkdir src/components

# 各コンポーネントファイルを作成
touch src/components/TodoItem.tsx
touch src/components/TodoFilter.tsx
touch src/components/TodoStats.tsx
```

これらのコンポーネントの詳細な実装は、UIセクションで説明します。

## 5. UI改善とスタイリング

### レスポンシブデザインの実装

Tailwind CSSを使用してモバイルフレンドリーなデザインを作成します。

### TodoItemコンポーネント

`src/components/TodoItem.tsx`:
```typescript
import React, { useState } from 'react';
import { Todo } from '@/types/todo';

interface TodoItemProps {
  todo: Todo;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
  onEdit: (id: string, newText: string) => void;
}

export default function TodoItem({ todo, onToggle, onDelete, onEdit }: TodoItemProps) {
  const [isEditing, setIsEditing] = useState(false);
  const [editText, setEditText] = useState(todo.text);

  const handleEdit = () => {
    if (editText.trim() !== '') {
      onEdit(todo.id, editText.trim());
    }
    setIsEditing(false);
  };

  const handleCancel = () => {
    setEditText(todo.text);
    setIsEditing(false);
  };

  return (
    <div className={`flex items-center gap-3 p-3 border rounded-md transition-all duration-200 ${
      todo.completed 
        ? 'bg-gray-50 opacity-75 border-gray-200' 
        : 'bg-white border-gray-300 hover:border-gray-400'
    }`}>
      {/* 完了チェックボックス */}
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={() => onToggle(todo.id)}
        className="w-4 h-4 text-blue-600 rounded focus:ring-blue-500 focus:ring-2"
      />

      {/* TODO内容 */}
      <div className="flex-1 min-w-0">
        {isEditing ? (
          <div className="flex flex-col sm:flex-row gap-2">
            <input
              type="text"
              value={editText}
              onChange={(e) => setEditText(e.target.value)}
              onKeyPress={(e) => {
                if (e.key === 'Enter') handleEdit();
                if (e.key === 'Escape') handleCancel();
              }}
              className="flex-1 px-2 py-1 border border-gray-300 rounded text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
              autoFocus
            />
            <div className="flex gap-1">
              <button
                onClick={handleEdit}
                className="px-3 py-1 bg-green-500 text-white rounded text-xs hover:bg-green-600 transition-colors"
              >
                保存
              </button>
              <button
                onClick={handleCancel}
                className="px-3 py-1 bg-gray-500 text-white rounded text-xs hover:bg-gray-600 transition-colors"
              >
                キャンセル
              </button>
            </div>
          </div>
        ) : (
          <span
            className={`block ${
              todo.completed 
                ? 'line-through text-gray-500' 
                : 'text-gray-800'
            } cursor-pointer break-words`}
            onDoubleClick={() => setIsEditing(true)}
            title="ダブルクリックで編集"
          >
            {todo.text}
          </span>
        )}
        
        {/* 日時情報（デスクトップのみ表示） */}
        <div className="hidden sm:block text-xs text-gray-400 mt-1">
          作成: {todo.createdAt.toLocaleDateString('ja-JP')} {todo.createdAt.toLocaleTimeString('ja-JP', { hour: '2-digit', minute: '2-digit' })}
          {todo.updatedAt.getTime() !== todo.createdAt.getTime() && (
            <span className="ml-2">
              更新: {todo.updatedAt.toLocaleDateString('ja-JP')} {todo.updatedAt.toLocaleTimeString('ja-JP', { hour: '2-digit', minute: '2-digit' })}
            </span>
          )}
        </div>
      </div>

      {/* 操作ボタン */}
      {!isEditing && (
        <div className="flex gap-1">
          <button
            onClick={() => setIsEditing(true)}
            className="px-2 py-1 text-blue-500 hover:text-blue-700 text-sm transition-colors"
            title="編集"
          >
            <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M11 5H6a2 2 0 00-2 2v11a2 2 0 002 2h11a2 2 0 002-2v-5m-1.414-9.414a2 2 0 112.828 2.828L11.828 15H9v-2.828l8.586-8.586z" />
            </svg>
          </button>
          <button
            onClick={() => onDelete(todo.id)}
            className="px-2 py-1 text-red-500 hover:text-red-700 text-sm transition-colors"
            title="削除"
          >
            <svg className="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16" />
            </svg>
          </button>
        </div>
      )}
    </div>
  );
}
```

### TodoFilterコンポーネント

`src/components/TodoFilter.tsx`:
```typescript
import React from 'react';
import { TodoFilter } from '@/types/todo';

interface TodoFilterProps {
  currentFilter: TodoFilter;
  onFilterChange: (filter: TodoFilter) => void;
}

const filterLabels: Record<TodoFilter, string> = {
  all: 'すべて',
  active: 'アクティブ',
  completed: '完了済み',
};

export default function TodoFilterComponent({ currentFilter, onFilterChange }: TodoFilterProps) {
  const filters: TodoFilter[] = ['all', 'active', 'completed'];

  return (
    <div className="mb-4 flex gap-1 justify-center">
      {filters.map((filter) => (
        <button
          key={filter}
          onClick={() => onFilterChange(filter)}
          className={`px-3 py-2 rounded-md text-sm font-medium transition-all duration-200 ${
            currentFilter === filter
              ? 'bg-blue-500 text-white shadow-md'
              : 'bg-gray-200 text-gray-700 hover:bg-gray-300'
          }`}
        >
          {filterLabels[filter]}
        </button>
      ))}
    </div>
  );
}
```

### TodoStatsコンポーネント

`src/components/TodoStats.tsx`:
```typescript
import React from 'react';

interface TodoStatsProps {
  stats: {
    total: number;
    completed: number;
    active: number;
  };
  onClearCompleted: () => void;
}

export default function TodoStats({ stats, onClearCompleted }: TodoStatsProps) {
  return (
    <div className="border-t pt-4 text-sm text-gray-600">
      <div className="flex flex-col sm:flex-row sm:justify-between sm:items-center gap-2">
        <div className="flex flex-wrap gap-4 text-center sm:text-left">
          <span className="font-medium">
            総計: <span className="text-blue-600">{stats.total}</span>件
          </span>
          <span>
            完了: <span className="text-green-600">{stats.completed}</span>件
          </span>
          <span>
            残り: <span className="text-orange-600">{stats.active}</span>件
          </span>
        </div>
        
        {stats.completed > 0 && (
          <button
            onClick={onClearCompleted}
            className="text-red-500 hover:text-red-700 text-sm font-medium transition-colors self-center sm:self-auto"
          >
            完了済みを削除 ({stats.completed}件)
          </button>
        )}
      </div>
    </div>
  );
}
```

### ダークモード対応

ダークモード機能を追加します：

```bash
# ダークモード用のコンテキストを作成
touch src/contexts/ThemeContext.tsx
```

`src/contexts/ThemeContext.tsx`:
```typescript
'use client';

import React, { createContext, useContext, useEffect, useState } from 'react';

type Theme = 'light' | 'dark';

interface ThemeContextType {
  theme: Theme;
  toggleTheme: () => void;
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light');

  useEffect(() => {
    // ローカルストレージから設定を読み込み
    const savedTheme = localStorage.getItem('theme') as Theme;
    if (savedTheme) {
      setTheme(savedTheme);
    } else {
      // システム設定を確認
      const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
      setTheme(prefersDark ? 'dark' : 'light');
    }
  }, []);

  useEffect(() => {
    // HTMLのclass属性を更新
    document.documentElement.classList.toggle('dark', theme === 'dark');
    localStorage.setItem('theme', theme);
  }, [theme]);

  const toggleTheme = () => {
    setTheme(prev => prev === 'light' ? 'dark' : 'light');
  };

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

export function useTheme() {
  const context = useContext(ThemeContext);
  if (context === undefined) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }
  return context;
}
```

### レイアウトファイルの更新

`src/app/layout.tsx`を更新してダークモード対応:
```typescript
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';
import { ThemeProvider } from '@/contexts/ThemeContext';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'TODO管理アプリ',
  description: 'シンプルで使いやすいTODO管理アプリケーション',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja" suppressHydrationWarning>
      <body className={inter.className}>
        <ThemeProvider>
          {children}
        </ThemeProvider>
      </body>
    </html>
  );
}
```

### グローバルスタイルの更新

`src/app/globals.css`にダークモード用のスタイルを追加:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* ダークモードの基本設定 */
@layer base {
  html {
    @apply transition-colors duration-300;
  }
  
  html.dark {
    color-scheme: dark;
  }
}

/* カスタムコンポーネントスタイル */
@layer components {
  .todo-container {
    @apply bg-white dark:bg-gray-800 text-gray-900 dark:text-gray-100;
  }
  
  .todo-input {
    @apply bg-white dark:bg-gray-700 border-gray-300 dark:border-gray-600 
           text-gray-900 dark:text-gray-100 placeholder-gray-500 dark:placeholder-gray-400;
  }
  
  .todo-button {
    @apply transition-all duration-200 focus:ring-2 focus:outline-none;
  }
  
  .todo-button-primary {
    @apply bg-blue-500 hover:bg-blue-600 dark:bg-blue-600 dark:hover:bg-blue-700 
           text-white focus:ring-blue-500;
  }
  
  .todo-item {
    @apply bg-white dark:bg-gray-700 border-gray-300 dark:border-gray-600 
           hover:border-gray-400 dark:hover:border-gray-500;
  }
  
  .todo-item-completed {
    @apply bg-gray-50 dark:bg-gray-800 opacity-75 border-gray-200 dark:border-gray-700;
  }
}

/* スクロールバーのスタイリング */
@layer utilities {
  .scrollbar-thin {
    scrollbar-width: thin;
    scrollbar-color: rgb(156, 163, 175) transparent;
  }
  
  .scrollbar-thin::-webkit-scrollbar {
    width: 6px;
  }
  
  .scrollbar-thin::-webkit-scrollbar-track {
    background: transparent;
  }
  
  .scrollbar-thin::-webkit-scrollbar-thumb {
    background-color: rgb(156, 163, 175);
    border-radius: 3px;
  }
  
  .dark .scrollbar-thin {
    scrollbar-color: rgb(75, 85, 99) transparent;
  }
  
  .dark .scrollbar-thin::-webkit-scrollbar-thumb {
    background-color: rgb(75, 85, 99);
  }
}
```

### Tailwind設定の更新

`tailwind.config.ts`でダークモードを有効化:
```typescript
import type { Config } from 'tailwindcss';

const config: Config = {
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  darkMode: 'class', // クラスベースのダークモード
  theme: {
    extend: {
      colors: {
        background: 'var(--background)',
        foreground: 'var(--foreground)',
      },
      animation: {
        'fade-in': 'fadeIn 0.3s ease-in-out',
        'slide-in': 'slideIn 0.3s ease-out',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideIn: {
          '0%': { transform: 'translateY(-10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
      },
    },
  },
  plugins: [],
};

export default config;
```

## 6. Githubでのコード管理

この章では、開発初心者向けにGitHubを使用したコード管理の基本から応用まで詳細に解説します。

### 6.1 Gitとは

**Git**は分散型バージョン管理システムで、以下の特徴があります：

- **履歴管理**: コードの変更履歴を詳細に記録
- **分散型**: ローカルとリモートの両方でバージョン管理
- **協業**: 複数人での開発に対応
- **ブランチ**: 並列開発をサポート

**GitHubとの違い:**
- **Git**: バージョン管理システム（ツール）
- **GitHub**: Gitを使用したクラウドサービス（プラットフォーム）

### 6.2 GitHubアカウントの設定

#### アカウント作成

1. **GitHubサイトにアクセス**: https://github.com
2. **Sign up**をクリック
3. **必要情報を入力**:
   - ユーザー名（プロジェクトURLに使用される）
   - メールアドレス
   - パスワード
4. **メール認証**を完了

#### SSH認証の設定（推奨）

SSH認証により、パスワード入力なしでGitHubにアクセスできます：

```bash
# SSH鍵の生成
ssh-keygen -t ed25519 -C "your-email@example.com"

# Enter連打でデフォルト設定を使用（パスフレーズは任意）

# 公開鍵の表示
cat ~/.ssh/id_ed25519.pub
```

**GitHubでのSSH鍵登録:**
1. GitHub → Settings → SSH and GPG keys
2. 「New SSH key」をクリック
3. Titleに分かりやすい名前を入力（例：「My Laptop」）
4. Keyフィールドに公開鍵をペースト
5. 「Add SSH key」をクリック

**接続テスト:**
```bash
ssh -T git@github.com
# "Hi [username]! You've successfully authenticated" が表示されればOK
```

### 6.3 ローカルGit設定

#### 基本設定

```bash
# ユーザー名の設定（GitHubのユーザー名と同じにする）
git config --global user.name "Your GitHub Username"

# メールアドレスの設定（GitHubに登録したメール）
git config --global user.email "your-email@example.com"

# デフォルトブランチ名をmainに設定
git config --global init.defaultBranch main

# プルの動作を設定（履歴を線形に保つ）
git config --global pull.rebase false

# 設定の確認
git config --list
```

#### 便利なエイリアス設定

```bash
# よく使うコマンドのエイリアス設定
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual '!gitk'

# ログをきれいに表示
git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

### 6.4 プロジェクトのGit初期化

#### ローカルリポジトリの初期化

プロジェクトディレクトリでGitを初期化します：

```bash
# プロジェクトディレクトリに移動
cd my-todo-app

# Gitリポジトリの初期化
git init

# 現在の状態を確認
git status
```

#### .gitignoreファイルの設定

Node.jsプロジェクト用の`.gitignore`を作成します：

```bash
# .gitignoreファイルを作成
touch .gitignore
```

`.gitignore`の内容:
```
# 依存関係
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# Next.js ビルド出力
.next/
out/

# 本番ビルド
build/
dist/

# 環境変数
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# ログファイル
*.log

# ランタイムデータ
pids
*.pid
*.seed
*.pid.lock

# IDE設定
.vscode/
.idea/
*.swp
*.swo
*~

# OS生成ファイル
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# TypeScript
*.tsbuildinfo

# ESLint
.eslintcache

# Turborepo
.turbo/

# テストカバレッジ
coverage/
.nyc_output/

# Vercel
.vercel

# Cloudflare
.wrangler/
```

#### 初回コミット

```bash
# すべてのファイルをステージング
git add .

# 初回コミット
git commit -m "初回コミット: Next.js TODO アプリの基本構造を追加

- Next.js 14 with TypeScript
- Tailwind CSS for styling
- 基本的なプロジェクト構造
- 基本的なTODO機能（CRUD操作）
- ローカルストレージでのデータ永続化"

# 現在の状態を確認
git status
git log --oneline
```

### 6.5 GitHubリポジトリの作成と連携

#### GitHubでリポジトリ作成

1. **GitHub**にログイン
2. **右上の「+」** → 「New repository」
3. **リポジトリ設定**:
   - Repository name: `my-todo-app`
   - Description: `Next.jsで作成したTODO管理アプリケーション`
   - Public/Private: お好みで選択
   - **重要**: 「Add a README file」「Add .gitignore」「Choose a license」はチェックしない（既にローカルにあるため）
4. **「Create repository」**をクリック

#### リモートリポジトリの追加

GitHubで作成したリポジトリとローカルを連携します：

```bash
# リモートリポジトリを追加（SSHの場合）
git remote add origin git@github.com:your-username/my-todo-app.git

# リモートリポジトリを追加（HTTPSの場合）
# git remote add origin https://github.com/your-username/my-todo-app.git

# リモートの確認
git remote -v

# メインブランチにプッシュ
git branch -M main
git push -u origin main
```

**プッシュ完了後の確認:**
- GitHubのリポジトリページをリロード
- ファイルが正常にアップロードされていることを確認

### 6.6 ブランチ戦略

#### ブランチとは

**ブランチ**は並列開発を可能にする機能で、以下の利点があります：
- **安全な実験**: メインコードに影響せずに新機能を開発
- **協業**: 複数人が同時に異なる機能を開発
- **履歴管理**: 機能ごとの変更履歴を分離

#### 基本的なブランチ戦略

**Git Flow**を簡素化した戦略を採用します：

```
main (メインブランチ)
├── feature/add-dark-mode     (機能開発)
├── feature/improve-ui        (UI改善)
└── hotfix/fix-storage-bug    (緊急修正)
```

#### ブランチの操作

**新しい機能ブランチの作成:**
```bash
# 現在のブランチを確認
git branch

# 新しい機能ブランチを作成して切り替え
git checkout -b feature/add-dark-mode

# または（Git 2.23以降）
git switch -c feature/add-dark-mode

# ブランチ一覧を確認
git branch -a
```

**ブランチ間の移動:**
```bash
# メインブランチに戻る
git checkout main
# または
git switch main

# 機能ブランチに戻る
git checkout feature/add-dark-mode
# または
git switch feature/add-dark-mode
```

### 6.7 コミットのベストプラクティス

#### 良いコミットメッセージの書き方

**基本フォーマット:**
```
タイプ: 簡潔な説明（50文字以内）

詳細な説明（必要に応じて）
- 変更理由
- 変更内容の詳細
- 影響範囲
```

**タイプの例:**
- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメント
- `style`: フォーマット（機能に影響しない変更）
- `refactor`: リファクタリング
- `test`: テスト
- `chore`: ビルドプロセスやツール変更

**良いコミットメッセージの例:**
```bash
git commit -m "feat: ダークモード機能を追加

- ThemeContextでテーマ状態を管理
- ローカルストレージでテーマ設定を永続化
- 全コンポーネントでダークモード対応
- システム設定の自動検出機能を追加"

git commit -m "fix: ローカルストレージのデータ読み込みエラーを修正

不正なJSONデータが保存されている場合の
エラーハンドリングを改善"

git commit -m "refactor: TODOアイテムコンポーネントを分離

再利用性と保守性の向上のため、
TodoItemを独立したコンポーネントとして分離"
```

#### 原子的コミット

**一つのコミットには一つの変更**を心がけます：

```bash
# 悪い例：複数の変更を一度にコミット
# git add .
# git commit -m "UIを改善してバグも修正"

# 良い例：変更を分けてコミット
# 1. UIの改善
git add src/components/TodoItem.tsx
git commit -m "style: TodoItemのスタイリングを改善"

# 2. バグ修正
git add src/utils/localStorage.ts
git commit -m "fix: ローカルストレージのエラーハンドリングを改善"
```

### 6.8 プルリクエスト（PR）の作成

#### 機能開発からPRまでの流れ

**1. 機能ブランチでの開発:**
```bash
# 機能ブランチに切り替え
git checkout feature/add-dark-mode

# 開発作業...

# 変更をコミット
git add .
git commit -m "feat: ダークモード機能を実装"

# リモートブランチにプッシュ
git push origin feature/add-dark-mode
```

**2. プルリクエストの作成:**

GitHubのWebインターフェースで：
1. リポジトリページにアクセス
2. 「Compare & pull request」ボタンをクリック
3. **PRのタイトル**を入力（コミットメッセージと同様）
4. **詳細な説明**を記述

**PRテンプレートの例:**
```markdown
## 概要
ダークモード機能を追加しました。

## 変更内容
- [ ] ThemeContextを追加
- [ ] 全コンポーネントでダークモード対応
- [ ] ローカルストレージでテーマ設定を永続化
- [ ] システム設定の自動検出

## テスト方法
1. アプリを起動
2. ヘッダーのテーマ切り替えボタンをクリック
3. ダークモードに切り替わることを確認
4. ページをリロードしても設定が保持されることを確認

## スクリーンショット
<!-- 必要に応じて画像を添付 -->

## その他
- 破壊的変更: なし
- 依存関係の追加: なし
```

#### コードレビューのポイント

**レビューで確認する項目:**
- 機能が仕様通りに動作するか
- コードの可読性・保守性
- エラーハンドリングの適切性
- パフォーマンスへの影響
- セキュリティ上の問題

**レビューコメントの例:**
```markdown
# 良いレビューコメント
このローカルストレージのエラーハンドリング、とても良いですね！
try-catchでしっかり例外処理されています。

小さな提案ですが、エラーログに具体的な操作内容を含めると
デバッグがしやすくなるかもしれません。

```typescript
console.error('ローカルストレージからのTODOデータ読み込みエラー:', error);
```

# 質問
このテーマ設定の初期値は、システム設定を優先する方針でしょうか？
ユーザーが明示的に設定した場合との優先順位を確認させてください。
```

### 6.9 マージとリリース管理

#### プルリクエストのマージ

**マージ方法の種類:**

1. **Merge commit**: 履歴を保持（推奨）
2. **Squash and merge**: コミットを統合
3. **Rebase and merge**: 履歴を線形化

**マージ後のクリーンアップ:**
```bash
# メインブランチに戻る
git checkout main

# 最新の状態を取得
git pull origin main

# 不要になったローカルブランチを削除
git branch -d feature/add-dark-mode

# リモートの不要ブランチを削除
git push origin --delete feature/add-dark-mode
```

#### リリースタグの作成

**セマンティックバージョニング**に従ってバージョンを管理：

```bash
# 現在の状態を確認
git checkout main
git pull origin main

# リリースタグを作成
git tag -a v1.0.0 -m "リリース v1.0.0

主な機能:
- TODO の CRUD 操作
- ローカルストレージでのデータ永続化
- フィルタリング機能
- レスポンシブデザイン
- ダークモード対応"

# タグをリモートにプッシュ
git push origin v1.0.0

# すべてのタグを確認
git tag -l
```

### 6.10 共同開発のワークフロー

#### フォークベースの開発（オープンソース）

**他の人のプロジェクトに貢献する場合:**

1. **フォーク**: GitHubでリポジトリをフォーク
2. **クローン**: フォークしたリポジトリをローカルにクローン
3. **リモート追加**: 元のリポジトリをupstreamとして追加
4. **開発**: 機能ブランチで開発
5. **プルリクエスト**: 元のリポジトリにPRを送信

```bash
# フォークしたリポジトリをクローン
git clone git@github.com:your-username/original-project.git
cd original-project

# 元のリポジトリをupstreamとして追加
git remote add upstream git@github.com:original-owner/original-project.git

# リモートの確認
git remote -v

# 最新の状態を取得
git fetch upstream
git checkout main
git merge upstream/main
```

#### ブランチ保護ルール

**本番環境の安全性**を確保するため、GitHubでブランチ保護を設定：

1. **Settings** → **Branches**
2. **Add rule**でmainブランチを保護
3. **推奨設定**:
   - Require pull request reviews before merging
   - Require status checks to pass before merging
   - Require branches to be up to date before merging
   - Include administrators

### 6.11 トラブルシューティング

#### よくある問題と解決法

**1. コミットメッセージを間違えた**
```bash
# 最後のコミットメッセージを修正
git commit --amend -m "正しいコミットメッセージ"

# プッシュ済みの場合（注意：共同開発では避ける）
git push --force-with-lease origin feature-branch
```

**2. 間違ったファイルをコミットした**
```bash
# 最後のコミットから特定ファイルを除外
git reset HEAD~1
git add 正しいファイル
git commit -m "修正されたコミット"
```

**3. マージコンフリクトの解決**
```bash
# メインブランチの最新を取得
git checkout main
git pull origin main

# 機能ブランチに戻ってマージ
git checkout feature/your-feature
git merge main

# コンフリクトが発生した場合、ファイルを手動で編集
# コンフリクトマーカーを解決:
# <<<<<<< HEAD
# あなたの変更
# =======
# 他の人の変更
# >>>>>>> main

# 解決後にコミット
git add .
git commit -m "merge: mainブランチとのコンフリクトを解決"
```

**4. 認証エラー**
```bash
# SSH接続を確認
ssh -T git@github.com

# SSH鍵の再生成（必要に応じて）
ssh-keygen -t ed25519 -C "your-email@example.com"
```

### 6.12 継続的インテグレーション（CI）

#### GitHub Actionsの設定

`.github/workflows/ci.yml`を作成してCI/CDを設定：

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18.x, 20.x]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Node.js ${{ matrix.node-version }} を使用
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: 依存関係をインストール
      run: npm ci
    
    - name: リンターを実行
      run: npm run lint
    
    - name: 型チェックを実行
      run: npm run type-check
    
    - name: ビルドを実行
      run: npm run build
    
    - name: テストを実行
      run: npm test
```

このCIパイプラインにより：
- プルリクエスト時の自動テスト
- 複数のNode.jsバージョンでのテスト
- コード品質の自動チェック
- ビルド成功の確認

## 7. Cloudflare Workersへのデプロイ

### 7.1 Cloudflare Workersとは

**Cloudflare Workers**は、エッジでJavaScriptを実行できるサーバーレスプラットフォームです：

**主な特徴:**
- **高速**: 世界中のエッジロケーションで実行
- **スケーラブル**: 自動スケーリング
- **コスト効率**: 低コストでの運用
- **簡単デプロイ**: GitHubとの連携

**従来のホスティングとの比較:**
| 項目 | Cloudflare Workers | 従来のサーバー |
|------|-------------------|---------------|
| 起動時間 | 瞬時 | 数秒〜数分 |
| スケーリング | 自動 | 手動設定 |
| 地理的分散 | 自動 | 手動設定 |
| 料金 | 使用量ベース | 固定 |

### 7.2 Cloudflareアカウントの設定

#### アカウント作成

1. **Cloudflareサイト**にアクセス: https://cloudflare.com
2. **「Sign Up」**をクリック
3. **必要情報を入力**:
   - メールアドレス
   - パスワード
4. **メール認証**を完了

#### Wranglerのインストール

**Wrangler**はCloudflare Workers用のCLIツールです：

```bash
# Wranglerをグローバルインストール
npm install -g wrangler

# バージョン確認
wrangler --version

# Cloudflareにログイン
wrangler login
```

`wrangler login`を実行すると：
1. ブラウザが開きます
2. Cloudflareアカウントでログイン
3. Wranglerのアクセスを許可
4. CLIに認証トークンが保存されます

### 7.3 Next.jsプロジェクトのWorkers対応

#### 静的エクスポートの設定

Cloudflare Workersで動作させるため、Next.jsを静的エクスポート用に設定します：

`next.config.js`を更新:
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',
  trailingSlash: true,
  images: {
    unoptimized: true,
  },
  // 静的エクスポート用の設定
  distDir: 'out',
  // Cloudflare Workers用の最適化
  experimental: {
    esmExternals: true,
  },
}

module.exports = nextConfig;
```

**設定の説明:**
- `output: 'export'`: 静的ファイルとしてエクスポート
- `trailingSlash: true`: URLの末尾にスラッシュを追加
- `images.unoptimized: true`: 画像最適化を無効化（静的エクスポートで必要）
- `distDir: 'out'`: 出力ディレクトリを指定

#### package.jsonスクリプトの追加

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "export": "next build && next export",
    "deploy": "wrangler pages publish out",
    "deploy:preview": "wrangler pages publish out --preview"
  }
}
```

### 7.4 Cloudflare Pagesプロジェクトの作成

#### プロジェクト作成

```bash
# Cloudflare Pagesプロジェクトを作成
wrangler pages project create my-todo-app

# プロジェクト一覧を確認
wrangler pages project list
```

プロジェクト作成時の設定：
- **プロジェクト名**: my-todo-app
- **本番ブランチ**: main
- **プレビューブランチ**: すべてのブランチ

#### 初回デプロイ

```bash
# プロジェクトをビルド
npm run build

# 静的ファイルをデプロイ
wrangler pages publish out --project-name my-todo-app

# または、プロジェクト設定がある場合
npm run deploy
```

デプロイが完了すると：
- **本番URL**: `https://my-todo-app.pages.dev`
- **プレビューURL**: `https://[commit-hash].my-todo-app.pages.dev`

### 7.5 継続的デプロイの設定

#### GitHub連携

GitHubとCloudflare Pagesを連携して自動デプロイを設定：

1. **Cloudflare Dashboard**にアクセス
2. **Pages** → **Connect to Git**
3. **GitHub**を選択
4. **リポジトリを選択**: `your-username/my-todo-app`
5. **ビルド設定**:
   - Build command: `npm run build`
   - Build output directory: `out`
   - Root directory: `/` (デフォルト)

**環境変数の設定**（必要に応じて）:
- `NODE_VERSION`: `20.x`
- `NPM_VERSION`: `latest`

#### wrangler.tomlの作成

プロジェクトルートに`wrangler.toml`を作成：

```toml
name = "my-todo-app"
compatibility_date = "2024-01-01"

[env.production]
name = "my-todo-app"

[env.staging]
name = "my-todo-app-staging"

# 静的ファイルの設定
[site]
bucket = "./out"

# ビルドコマンドの設定
[build]
command = "npm run build"
cwd = "./"
watch_dir = "src"

# 環境変数
[vars]
NODE_ENV = "production"

# カスタムドメインの設定（任意）
# [[route]]
# pattern = "todo.example.com"
# zone_name = "example.com"
```

### 7.6 パフォーマンス最適化

#### バンドルサイズの最適化

```bash
# バンドルアナライザーをインストール
npm install --save-dev @next/bundle-analyzer

# package.jsonにスクリプト追加
"analyze": "ANALYZE=true npm run build"
```

`next.config.js`を更新:
```javascript
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'export',
  trailingSlash: true,
  images: {
    unoptimized: true,
  },
  // Tree shakingの最適化
  experimental: {
    optimizeCss: true,
  },
  // Unused CSS の削除
  swcMinify: true,
};

module.exports = withBundleAnalyzer(nextConfig);
```

#### CDNキャッシュの最適化

静的ファイルの適切なキャッシュ設定：

```bash
# _headersファイルを作成（Cloudflare Pages用）
touch public/_headers
```

`public/_headers`:
```
# 静的アセットのキャッシュ設定
/_next/static/*
  Cache-Control: public, max-age=31536000, immutable

# HTMLファイルのキャッシュ設定
/*.html
  Cache-Control: public, max-age=3600

# APIエンドポイント（該当する場合）
/api/*
  Cache-Control: no-cache

# セキュリティヘッダー
/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: geolocation=(), camera=(), microphone=()
```

### 7.7 セキュリティ設定

#### セキュリティヘッダーの追加

`public/_headers`にセキュリティ設定を追加:
```
/*
  # XSSプロテクション
  X-XSS-Protection: 1; mode=block
  
  # コンテンツタイプスニッフィング防止
  X-Content-Type-Options: nosniff
  
  # クリックジャッキング防止
  X-Frame-Options: DENY
  
  # HTTPS強制（本番環境）
  Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
  
  # リファラーポリシー
  Referrer-Policy: strict-origin-when-cross-origin
  
  # 権限ポリシー
  Permissions-Policy: geolocation=(), camera=(), microphone=(), payment=()
  
  # Content Security Policy（必要に応じて調整）
  Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' static.cloudflareinsights.com; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self';
```

#### 環境変数の管理

機密情報は環境変数で管理：

```bash
# Cloudflare Pagesで環境変数を設定
wrangler pages secret put API_KEY

# または Dashboard経由で設定
# Pages → プロジェクト → Settings → Environment variables
```

### 7.8 本番運用のベストプラクティス

#### デプロイフローの確立

```bash
# 開発フロー
git checkout -b feature/new-feature
# 開発作業...
git commit -m "feat: 新機能を追加"
git push origin feature/new-feature
# GitHub でPR作成 → レビュー → マージ
# → 自動的にCloudflare Pagesにデプロイ
```

#### ロールバック手順

```bash
# 以前のデプロイを確認
wrangler pages deployment list --project-name my-todo-app

# 特定のデプロイメントにロールバック
wrangler pages deployment activate [deployment-id] --project-name my-todo-app
```

#### パフォーマンス監視

定期的なパフォーマンスチェック：
- **Lighthouse**スコアの監視
- **Core Web Vitals**の追跡
- **バンドルサイズ**の監視
- **ローディング時間**の測定

```bash
# Lighthouseでパフォーマンス測定
npx lighthouse https://your-app.pages.dev --output html --output-path lighthouse-report.html
```

## 8. トラブルシューティング

### 8.1 開発環境の問題

#### Node.js関連

**問題: `npm install` エラー**
```bash
# キャッシュクリア
npm cache clean --force

# node_modulesを削除して再インストール
rm -rf node_modules package-lock.json
npm install

# Nodeバージョン確認
node --version
# 推奨: v18.x または v20.x
```

**問題: TypeScriptエラー**
```bash
# TypeScript設定の確認
npx tsc --noEmit

# 型定義ファイルの再インストール
npm install --save-dev @types/node @types/react @types/react-dom
```

#### Next.js関連

**問題: ビルドエラー**
```bash
# Next.jsキャッシュをクリア
rm -rf .next
npm run build

# 詳細なエラー情報を表示
npm run build -- --debug
```

**問題: 静的エクスポートエラー**
```bash
# 問題のあるページを特定
npm run build 2>&1 | grep "Error"

# よくある原因:
# - getServerSidePropsの使用（静的エクスポートでは使用不可）
# - 動的ルートの不適切な設定
# - 画像最適化の設定不備
```

### 8.2 ローカルストレージ関連

#### データの破損・読み込みエラー

**問題診断:**
```javascript
// ブラウザのコンソールで実行
console.log(localStorage.getItem('todo-app-data'));

// データをクリア
localStorage.clear();

// 特定のキーのみクリア
localStorage.removeItem('todo-app-data');
```

**予防策:**
```typescript
// src/utils/localStorage.ts に追加
export class LocalStorageManager {
  // データの整合性チェック
  static validateTodos(todos: any[]): boolean {
    return Array.isArray(todos) && todos.every(todo => 
      typeof todo.id === 'string' &&
      typeof todo.text === 'string' &&
      typeof todo.completed === 'boolean'
    );
  }

  // 安全なデータ読み込み
  static safeLoadTodos(): Todo[] {
    try {
      const data = localStorage.getItem(STORAGE_KEY);
      if (!data) return [];

      const parsed = JSON.parse(data);
      if (!this.validateTodos(parsed)) {
        console.warn('Invalid todo data found, clearing storage');
        this.clearTodos();
        return [];
      }

      return parsed.map((todo: any) => ({
        ...todo,
        createdAt: new Date(todo.createdAt),
        updatedAt: new Date(todo.updatedAt),
      }));
    } catch (error) {
      console.error('Failed to load todos:', error);
      this.clearTodos();
      return [];
    }
  }
}
```

#### ブラウザ互換性

**Safari のプライベートモード**:
```typescript
// ローカルストレージの利用可能性を確認
function isLocalStorageAvailable(): boolean {
  try {
    const test = '__localStorage_test__';
    localStorage.setItem(test, test);
    localStorage.removeItem(test);
    return true;
  } catch (e) {
    return false;
  }
}

// フォールバック機能
class StorageManager {
  private static fallbackData: Todo[] = [];

  static saveTodos(todos: Todo[]): void {
    if (isLocalStorageAvailable()) {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(todos));
    } else {
      this.fallbackData = todos;
      console.warn('LocalStorage not available, using memory storage');
    }
  }

  static loadTodos(): Todo[] {
    if (isLocalStorageAvailable()) {
      return LocalStorageManager.safeLoadTodos();
    } else {
      return this.fallbackData;
    }
  }
}
```

### 8.3 Git・GitHub関連

#### 認証エラー

**SSH認証の問題:**
```bash
# SSH接続テスト
ssh -T git@github.com

# 鍵の確認
ls -la ~/.ssh/

# 新しい鍵の生成
ssh-keygen -t ed25519 -C "your-email@example.com"
cat ~/.ssh/id_ed25519.pub
# GitHubに公開鍵を追加
```

**HTTPS認証の問題:**
```bash
# 認証情報をクリア
git config --global --unset user.password

# Personal Access Tokenを使用
# GitHub → Settings → Developer settings → Personal access tokens
```

#### コミット・プッシュエラー

**大きなファイルのエラー:**
```bash
# ファイルサイズを確認
find . -size +50M -type f

# .gitignoreに追加
echo "*.log" >> .gitignore
echo "node_modules/" >> .gitignore

# 既にコミットされた大きなファイルを履歴から削除
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch path/to/large/file' \
  --prune-empty --tag-name-filter cat -- --all
```

**プッシュの拒否:**
```bash
# リモートの変更を取得
git fetch origin

# マージまたはリベース
git merge origin/main
# または
git rebase origin/main

# 競合がある場合は手動で解決後
git add .
git commit -m "resolve conflicts"
git push origin main
```

### 8.4 Cloudflare Workers・Pages関連

#### デプロイエラー

**ビルドエラーの診断:**
```bash
# ローカルでビルドテスト
npm run build

# 出力ディレクトリの確認
ls -la out/

# Wranglerでローカルテスト
wrangler pages dev out --port 8080
```

**よくあるエラーと解決法:**

**1. "Build failed" エラー**
```bash
# package.jsonのビルドコマンドを確認
{
  "scripts": {
    "build": "next build"
  }
}

# Cloudflare Pagesの設定を確認
# Build command: npm run build
# Build output directory: out
```

**2. "404 Not Found" エラー**
```bash
# next.config.jsの設定を確認
module.exports = {
  output: 'export',
  trailingSlash: true,
  images: {
    unoptimized: true,
  },
};

# 出力ファイルの構造を確認
tree out/
```

**3. "Module not found" エラー**
```bash
# 依存関係の確認
npm ls

# 不要な依存関係を削除
npm prune

# package-lock.jsonを再生成
rm package-lock.json
npm install
```

#### パフォーマンス問題

**読み込みが遅い:**
```bash
# バンドルサイズを確認
npm run analyze

# 使用されていないライブラリを特定
npx depcheck

# Code splittingの実装
# Next.jsのdynamic importを使用
const DynamicComponent = dynamic(() => import('./HeavyComponent'), {
  loading: () => <p>Loading...</p>,
});
```

### 8.5 ブラウザ互換性

#### CSS関連

**Tailwind CSSが適用されない:**
```bash
# Tailwind設定を確認
cat tailwind.config.ts

# ビルド時のCSS生成を確認
npm run build

# PurgeCSS設定を確認
# tailwind.config.tsのcontentパスが正しいか確認
```

**ダークモード関連:**
```typescript
// システム設定の確認
function getSystemTheme(): 'light' | 'dark' {
  if (typeof window === 'undefined') return 'light';
  
  return window.matchMedia('(prefers-color-scheme: dark)').matches 
    ? 'dark' 
    : 'light';
}

// ダークモード切り替えの問題
useEffect(() => {
  const root = document.documentElement;
  if (theme === 'dark') {
    root.classList.add('dark');
  } else {
    root.classList.remove('dark');
  }
}, [theme]);
```

#### JavaScript関連

**ES6+ 機能の互換性:**
```typescript
// Optional chainingの代替
// 問題のあるコード: todo?.createdAt?.toISOString()
// 安全なコード:
const formatDate = (date: Date | undefined) => {
  return date ? date.toISOString() : '';
};

// Array.find の代替
const findTodo = (todos: Todo[], id: string): Todo | undefined => {
  return todos.find(todo => todo.id === id) || undefined;
};
```

### 8.6 パフォーマンス最適化

#### React関連

**不要な再レンダリング:**
```typescript
// useCallbackとuseMemoの活用
const TodoList = ({ todos, onToggle, onDelete }: TodoListProps) => {
  const filteredTodos = useMemo(() => {
    return todos.filter(todo => {
      switch (filter) {
        case 'active': return !todo.completed;
        case 'completed': return todo.completed;
        default: return true;
      }
    });
  }, [todos, filter]);

  const handleToggle = useCallback((id: string) => {
    onToggle(id);
  }, [onToggle]);

  return (
    // JSX
  );
};
```

**大量のデータ処理:**
```typescript
// 仮想化リストの実装（react-window使用）
import { FixedSizeList as List } from 'react-window';

const VirtualizedTodoList = ({ todos }: { todos: Todo[] }) => {
  const Row = ({ index, style }: { index: number; style: any }) => (
    <div style={style}>
      <TodoItem todo={todos[index]} />
    </div>
  );

  return (
    <List
      height={400}
      itemCount={todos.length}
      itemSize={60}
    >
      {Row}
    </List>
  );
};
```

#### ネットワーク関連

**バンドルサイズの最適化:**
```bash
# Tree shakingの確認
npm run build -- --analyze

# 未使用の依存関係を削除
npx depcheck
npm uninstall unused-package

# 動的インポートの活用
const HeavyComponent = lazy(() => import('./HeavyComponent'));
```

### 8.7 セキュリティ問題

#### XSS対策

**ユーザー入力のサニタイゼーション:**
```typescript
// DOMPurifyを使用したサニタイゼーション
import DOMPurify from 'dompurify';

const sanitizeInput = (input: string): string => {
  return DOMPurify.sanitize(input, { 
    ALLOWED_TAGS: [], 
    ALLOWED_ATTR: [] 
  });
};

// TODO作成時のバリデーション
const addTodo = (text: string) => {
  const sanitizedText = sanitizeInput(text.trim());
  if (sanitizedText.length === 0) return;
  
  // TODO追加処理
};
```

#### Content Security Policy

**CSP設定の調整:**
```
# public/_headers
/*
  Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' static.cloudflareinsights.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self'; connect-src 'self' api.github.com; frame-ancestors 'none';
```

### 8.8 データバックアップ・復旧

#### データエクスポート・インポート機能

```typescript
// バックアップ機能の実装
const BackupManager = {
  exportData: (): void => {
    try {
      const todos = LocalStorageManager.loadTodos();
      const backup = {
        version: '1.0',
        exportDate: new Date().toISOString(),
        todos: todos,
      };
      
      const blob = new Blob([JSON.stringify(backup, null, 2)], {
        type: 'application/json'
      });
      
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `todo-backup-${new Date().toISOString().split('T')[0]}.json`;
      a.click();
      
      URL.revokeObjectURL(url);
    } catch (error) {
      console.error('Export failed:', error);
      alert('データのエクスポートに失敗しました');
    }
  },

  importData: (file: File): Promise<void> => {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = (e) => {
        try {
          const content = e.target?.result as string;
          const backup = JSON.parse(content);
          
          // バックアップファイルの検証
          if (!backup.todos || !Array.isArray(backup.todos)) {
            throw new Error('Invalid backup file format');
          }
          
          // データの復旧
          LocalStorageManager.saveTodos(backup.todos);
          resolve();
        } catch (error) {
          reject(error);
        }
      };
      reader.readAsText(file);
    });
  },
};
```

このトラブルシューティングガイドにより、開発からデプロイまでの各段階で発生する可能性のある問題に対応できます。問題が発生した際は、該当するセクションを参照して段階的に解決を進めてください。
