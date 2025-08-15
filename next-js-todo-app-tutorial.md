# 初めてのTODO管理Webアプリケーション開発ガイド

このドキュメントでは、開発初心者向けにNext.jsを使用したTODO管理Webアプリケーションの開発からCloudflare Workersへのデプロイまでを、0から100まで詳細に解説します。

## 目次

1. [概要](#概要)
2. [前提条件と環境準備](#前提条件と環境準備)
3. [プロジェクトの初期化](#プロジェクトの初期化)
4. [基本的なTODOアプリケーションの開発](#基本的なtodoアプリケーションの開発)
5. [データベースの統合](#データベースの統合)
6. [UIの改善とスタイリング](#uiの改善とスタイリング)
7. [Cloudflare Workersへのデプロイ準備](#cloudflare-workersへのデプロイ準備)
8. [デプロイとテスト](#デプロイとテスト)
9. [トラブルシューティング](#トラブルシューティング)

## 概要

このチュートリアルで作成するアプリケーションの特徴：

- **フレームワーク**: Next.js 14 (最新安定版)
- **言語**: TypeScript
- **データベース**: SQLite (本番環境では Cloudflare D1)
- **スタイリング**: Tailwind CSS
- **デプロイ**: Cloudflare Workers
- **機能**: TODO の作成・一覧表示・編集・削除 (CRUD操作)

### アプリケーションの機能概要

1. **TODOの一覧表示**: 作成されたTODOを一覧で表示
2. **TODOの作成**: 新しいTODOアイテムを追加
3. **TODOの編集**: 既存のTODOの内容を変更
4. **TODOの削除**: 不要なTODOを削除
5. **完了状態の切り替え**: TODOの完了・未完了を切り替え

## 前提条件と環境準備

### 必要な環境

- Windows 10/11 または macOS または Ubuntu Linux
- WSL2 (Windowsの場合)
- Node.js 18.17以降
- npm または yarn
- Git
- Visual Studio Code

### 環境構築の確認

Windows環境の場合は、先ほどの `windows-development-setup.md` ガイドに従って環境を構築してください。

1. **Node.jsのバージョン確認**
   ```bash
   node --version
   npm --version
   ```
   
   出力例:
   ```
   v20.9.0
   10.1.0
   ```

2. **Gitの設定確認**
   ```bash
   git --version
   git config --global --list
   ```

## プロジェクトの初期化

### Step 1: Next.jsプロジェクトの作成

1. **プロジェクト用ディレクトリの作成**
   ```bash
   mkdir my-todo-app
   cd my-todo-app
   ```

2. **Next.jsプロジェクトの初期化**
   ```bash
   npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
   ```
   
   このコマンドの説明：
   - `--typescript`: TypeScriptを使用
   - `--tailwind`: Tailwind CSSを使用
   - `--eslint`: ESLintを使用
   - `--app`: App Routerを使用（Next.js 13+の新機能）
   - `--src-dir`: src/ディレクトリ構造を使用
   - `--import-alias "@/*"`: パスエイリアスを設定

3. **依存関係のインストール確認**
   ```bash
   npm ls
   ```

### Step 2: プロジェクト構造の確認

作成されたプロジェクトの構造を確認します：

```
my-todo-app/
├── src/
│   ├── app/
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   └── components/
├── public/
├── package.json
├── tailwind.config.ts
├── tsconfig.json
└── next.config.js
```

### Step 3: 開発サーバーの起動確認

```bash
npm run dev
```

ブラウザで http://localhost:3000 にアクセスして、Next.jsのデフォルトページが表示されることを確認してください。

## 基本的なTODOアプリケーションの開発

### Step 1: 型定義の作成

まず、TODOアイテムの型定義を作成します。

**src/types/todo.ts**
```typescript
export interface Todo {
  id: string;
  title: string;
  description?: string;
  completed: boolean;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateTodoInput {
  title: string;
  description?: string;
}

export interface UpdateTodoInput {
  title?: string;
  description?: string;
  completed?: boolean;
}
```

### Step 2: TODOの状態管理

Next.jsのApp Routerを使用して、シンプルな状態管理を実装します。

**src/app/hooks/useTodos.ts**
```typescript
'use client';

import { useState, useEffect } from 'react';
import { Todo, CreateTodoInput, UpdateTodoInput } from '@/types/todo';

export const useTodos = () => {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [loading, setLoading] = useState(true);

  // 初期データの読み込み（ローカルストレージから）
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
        console.error('TODOデータの読み込みに失敗しました:', error);
      }
    }
    setLoading(false);
  }, []);

  // TODOの保存（ローカルストレージに）
  const saveTodos = (newTodos: Todo[]) => {
    try {
      localStorage.setItem('todos', JSON.stringify(newTodos));
      setTodos(newTodos);
    } catch (error) {
      console.error('TODOデータの保存に失敗しました:', error);
    }
  };

  // TODO作成
  const createTodo = (input: CreateTodoInput): Promise<Todo> => {
    return new Promise((resolve) => {
      const newTodo: Todo = {
        id: Date.now().toString(),
        title: input.title,
        description: input.description,
        completed: false,
        createdAt: new Date(),
        updatedAt: new Date(),
      };
      
      const newTodos = [...todos, newTodo];
      saveTodos(newTodos);
      resolve(newTodo);
    });
  };

  // TODO更新
  const updateTodo = (id: string, input: UpdateTodoInput): Promise<Todo | null> => {
    return new Promise((resolve) => {
      const todoIndex = todos.findIndex(todo => todo.id === id);
      if (todoIndex === -1) {
        resolve(null);
        return;
      }

      const updatedTodo: Todo = {
        ...todos[todoIndex],
        ...input,
        updatedAt: new Date(),
      };

      const newTodos = [...todos];
      newTodos[todoIndex] = updatedTodo;
      saveTodos(newTodos);
      resolve(updatedTodo);
    });
  };

  // TODO削除
  const deleteTodo = (id: string): Promise<boolean> => {
    return new Promise((resolve) => {
      const newTodos = todos.filter(todo => todo.id !== id);
      saveTodos(newTodos);
      resolve(true);
    });
  };

  // 完了状態の切り替え
  const toggleComplete = (id: string): Promise<Todo | null> => {
    const todo = todos.find(t => t.id === id);
    if (!todo) return Promise.resolve(null);
    
    return updateTodo(id, { completed: !todo.completed });
  };

  return {
    todos,
    loading,
    createTodo,
    updateTodo,
    deleteTodo,
    toggleComplete,
  };
};
```

### Step 3: TODOコンポーネントの作成

**src/components/TodoItem.tsx**
```typescript
'use client';

import { Todo } from '@/types/todo';
import { useState } from 'react';

interface TodoItemProps {
  todo: Todo;
  onUpdate: (id: string, updates: { title?: string; description?: string }) => void;
  onDelete: (id: string) => void;
  onToggleComplete: (id: string) => void;
}

export const TodoItem: React.FC<TodoItemProps> = ({
  todo,
  onUpdate,
  onDelete,
  onToggleComplete,
}) => {
  const [isEditing, setIsEditing] = useState(false);
  const [title, setTitle] = useState(todo.title);
  const [description, setDescription] = useState(todo.description || '');

  const handleSave = () => {
    if (title.trim()) {
      onUpdate(todo.id, {
        title: title.trim(),
        description: description.trim() || undefined,
      });
      setIsEditing(false);
    }
  };

  const handleCancel = () => {
    setTitle(todo.title);
    setDescription(todo.description || '');
    setIsEditing(false);
  };

  if (isEditing) {
    return (
      <div className="bg-white rounded-lg shadow-md p-4 border-l-4 border-blue-500">
        <div className="space-y-3">
          <input
            type="text"
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            placeholder="TODOのタイトルを入力..."
            autoFocus
          />
          <textarea
            value={description}
            onChange={(e) => setDescription(e.target.value)}
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            placeholder="詳細説明（任意）..."
            rows={3}
          />
          <div className="flex gap-2">
            <button
              onClick={handleSave}
              className="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500"
            >
              保存
            </button>
            <button
              onClick={handleCancel}
              className="px-4 py-2 bg-gray-500 text-white rounded-md hover:bg-gray-600 focus:outline-none focus:ring-2 focus:ring-gray-500"
            >
              キャンセル
            </button>
          </div>
        </div>
      </div>
    );
  }

  return (
    <div className={`bg-white rounded-lg shadow-md p-4 border-l-4 ${
      todo.completed ? 'border-green-500 bg-gray-50' : 'border-gray-300'
    }`}>
      <div className="flex items-start justify-between">
        <div className="flex-1">
          <div className="flex items-center gap-3">
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => onToggleComplete(todo.id)}
              className="w-5 h-5 text-blue-600 rounded focus:ring-blue-500"
            />
            <h3 className={`text-lg font-medium ${
              todo.completed ? 'line-through text-gray-500' : 'text-gray-900'
            }`}>
              {todo.title}
            </h3>
          </div>
          {todo.description && (
            <p className={`mt-2 ml-8 text-sm ${
              todo.completed ? 'text-gray-400' : 'text-gray-600'
            }`}>
              {todo.description}
            </p>
          )}
          <div className="mt-3 ml-8 text-xs text-gray-400">
            作成日: {todo.createdAt.toLocaleString('ja-JP')}
            {todo.updatedAt.getTime() !== todo.createdAt.getTime() && (
              <span className="ml-4">
                更新日: {todo.updatedAt.toLocaleString('ja-JP')}
              </span>
            )}
          </div>
        </div>
        <div className="flex gap-2 ml-4">
          <button
            onClick={() => setIsEditing(true)}
            className="px-3 py-1 text-sm bg-yellow-500 text-white rounded hover:bg-yellow-600 focus:outline-none focus:ring-2 focus:ring-yellow-500"
          >
            編集
          </button>
          <button
            onClick={() => onDelete(todo.id)}
            className="px-3 py-1 text-sm bg-red-500 text-white rounded hover:bg-red-600 focus:outline-none focus:ring-2 focus:ring-red-500"
          >
            削除
          </button>
        </div>
      </div>
    </div>
  );
};
```

### Step 4: TODO作成フォームの作成

**src/components/TodoForm.tsx**
```typescript
'use client';

import { useState } from 'react';
import { CreateTodoInput } from '@/types/todo';

interface TodoFormProps {
  onSubmit: (input: CreateTodoInput) => void;
  loading?: boolean;
}

export const TodoForm: React.FC<TodoFormProps> = ({ onSubmit, loading = false }) => {
  const [title, setTitle] = useState('');
  const [description, setDescription] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    
    if (title.trim()) {
      onSubmit({
        title: title.trim(),
        description: description.trim() || undefined,
      });
      
      // フォームをリセット
      setTitle('');
      setDescription('');
    }
  };

  return (
    <form onSubmit={handleSubmit} className="bg-white rounded-lg shadow-md p-6">
      <h2 className="text-xl font-semibold text-gray-900 mb-4">新しいTODOを追加</h2>
      
      <div className="space-y-4">
        <div>
          <label htmlFor="title" className="block text-sm font-medium text-gray-700 mb-2">
            タイトル（必須）
          </label>
          <input
            type="text"
            id="title"
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
            placeholder="やることを入力してください..."
            required
            disabled={loading}
          />
        </div>
        
        <div>
          <label htmlFor="description" className="block text-sm font-medium text-gray-700 mb-2">
            詳細説明（任意）
          </label>
          <textarea
            id="description"
            value={description}
            onChange={(e) => setDescription(e.target.value)}
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
            placeholder="詳細な説明があれば入力してください..."
            rows={3}
            disabled={loading}
          />
        </div>
        
        <button
          type="submit"
          disabled={!title.trim() || loading}
          className="w-full px-4 py-2 bg-blue-500 text-white font-medium rounded-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:bg-gray-400 disabled:cursor-not-allowed"
        >
          {loading ? '追加中...' : 'TODOを追加'}
        </button>
      </div>
    </form>
  );
};
```

### Step 5: メインページの実装

**src/app/page.tsx**
```typescript
'use client';

import { useTodos } from '@/app/hooks/useTodos';
import { TodoForm } from '@/components/TodoForm';
import { TodoItem } from '@/components/TodoItem';
import { CreateTodoInput } from '@/types/todo';
import { useState } from 'react';

export default function Home() {
  const { todos, loading, createTodo, updateTodo, deleteTodo, toggleComplete } = useTodos();
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all');

  const handleCreateTodo = async (input: CreateTodoInput) => {
    try {
      await createTodo(input);
    } catch (error) {
      console.error('TODO作成エラー:', error);
      alert('TODOの作成に失敗しました。');
    }
  };

  const handleUpdateTodo = async (id: string, updates: { title?: string; description?: string }) => {
    try {
      await updateTodo(id, updates);
    } catch (error) {
      console.error('TODO更新エラー:', error);
      alert('TODOの更新に失敗しました。');
    }
  };

  const handleDeleteTodo = async (id: string) => {
    if (confirm('このTODOを削除しますか？')) {
      try {
        await deleteTodo(id);
      } catch (error) {
        console.error('TODO削除エラー:', error);
        alert('TODOの削除に失敗しました。');
      }
    }
  };

  const handleToggleComplete = async (id: string) => {
    try {
      await toggleComplete(id);
    } catch (error) {
      console.error('TODO状態更新エラー:', error);
      alert('TODOの状態更新に失敗しました。');
    }
  };

  // フィルタリング
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

  // 統計情報の計算
  const stats = {
    total: todos.length,
    active: todos.filter(todo => !todo.completed).length,
    completed: todos.filter(todo => todo.completed).length,
  };

  if (loading) {
    return (
      <div className="min-h-screen bg-gray-100 flex items-center justify-center">
        <div className="text-center">
          <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-500 mx-auto"></div>
          <p className="mt-4 text-gray-600">読み込み中...</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-100 py-8">
      <div className="container mx-auto px-4 max-w-4xl">
        {/* ヘッダー */}
        <header className="text-center mb-8">
          <h1 className="text-4xl font-bold text-gray-900 mb-2">TODO管理アプリ</h1>
          <p className="text-gray-600">シンプルで使いやすいタスク管理ツール</p>
        </header>

        {/* 統計情報 */}
        <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-8">
          <div className="bg-white rounded-lg shadow-md p-4 text-center">
            <div className="text-2xl font-bold text-blue-500">{stats.total}</div>
            <div className="text-sm text-gray-600">総TODO数</div>
          </div>
          <div className="bg-white rounded-lg shadow-md p-4 text-center">
            <div className="text-2xl font-bold text-yellow-500">{stats.active}</div>
            <div className="text-sm text-gray-600">未完了</div>
          </div>
          <div className="bg-white rounded-lg shadow-md p-4 text-center">
            <div className="text-2xl font-bold text-green-500">{stats.completed}</div>
            <div className="text-sm text-gray-600">完了済み</div>
          </div>
        </div>

        {/* TODO作成フォーム */}
        <div className="mb-8">
          <TodoForm onSubmit={handleCreateTodo} />
        </div>

        {/* フィルター */}
        <div className="mb-6">
          <div className="flex justify-center">
            <div className="bg-white rounded-lg shadow-md p-1 flex">
              {(['all', 'active', 'completed'] as const).map((filterType) => (
                <button
                  key={filterType}
                  onClick={() => setFilter(filterType)}
                  className={`px-4 py-2 rounded-md font-medium transition-colors ${
                    filter === filterType
                      ? 'bg-blue-500 text-white'
                      : 'text-gray-600 hover:bg-gray-100'
                  }`}
                >
                  {filterType === 'all' && 'すべて'}
                  {filterType === 'active' && '未完了'}
                  {filterType === 'completed' && '完了済み'}
                </button>
              ))}
            </div>
          </div>
        </div>

        {/* TODO一覧 */}
        <div className="space-y-4">
          {filteredTodos.length === 0 ? (
            <div className="text-center py-8">
              <div className="text-gray-400 text-lg">
                {filter === 'all' && 'まだTODOがありません。上のフォームから追加してください。'}
                {filter === 'active' && '未完了のTODOはありません。'}
                {filter === 'completed' && '完了したTODOはありません。'}
              </div>
            </div>
          ) : (
            filteredTodos.map((todo) => (
              <TodoItem
                key={todo.id}
                todo={todo}
                onUpdate={handleUpdateTodo}
                onDelete={handleDeleteTodo}
                onToggleComplete={handleToggleComplete}
              />
            ))
          )}
        </div>
      </div>
    </div>
  );
}
```

### Step 6: レイアウトの調整

**src/app/layout.tsx**
```typescript
import type { Metadata } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';

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
    <html lang="ja">
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

### Step 7: 基本機能のテスト

1. **開発サーバーの起動**
   ```bash
   npm run dev
   ```

2. **ブラウザでテスト**
   - http://localhost:3000 にアクセス
   - TODOの作成、編集、削除、完了状態の切り替えをテスト
   - ブラウザを再読み込みしてもデータが保持されることを確認

## データベースの統合

### Step 1: Prismaのセットアップ

実際のアプリケーションでは、データをローカルストレージではなくデータベースに保存する必要があります。開発環境ではSQLiteを使用し、本番環境ではCloudflare D1を使用します。

1. **Prismaのインストール**
   ```bash
   npm install prisma @prisma/client
   npm install -D prisma
   ```

2. **Prismaの初期化**
   ```bash
   npx prisma init --datasource-provider sqlite
   ```

### Step 2: データベーススキーマの定義

**prisma/schema.prisma**
```prisma
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model Todo {
  id          String   @id @default(cuid())
  title       String
  description String?
  completed   Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@map("todos")
}
```

### Step 3: 環境変数の設定

**.env.local**
```
DATABASE_URL="file:./dev.db"
```

### Step 4: データベースの生成とマイグレーション

```bash
npx prisma migrate dev --name init
npx prisma generate
```

### Step 5: APIルートの作成

**src/app/api/todos/route.ts**
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// GET: TODO一覧取得
export async function GET() {
  try {
    const todos = await prisma.todo.findMany({
      orderBy: {
        createdAt: 'desc',
      },
    });
    
    return NextResponse.json(todos);
  } catch (error) {
    console.error('TODO取得エラー:', error);
    return NextResponse.json(
      { error: 'TODOの取得に失敗しました' },
      { status: 500 }
    );
  }
}

// POST: TODO作成
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { title, description } = body;

    if (!title || typeof title !== 'string' || !title.trim()) {
      return NextResponse.json(
        { error: 'タイトルは必須です' },
        { status: 400 }
      );
    }

    const todo = await prisma.todo.create({
      data: {
        title: title.trim(),
        description: description?.trim() || null,
      },
    });

    return NextResponse.json(todo, { status: 201 });
  } catch (error) {
    console.error('TODO作成エラー:', error);
    return NextResponse.json(
      { error: 'TODOの作成に失敗しました' },
      { status: 500 }
    );
  }
}
```

**src/app/api/todos/[id]/route.ts**
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// PUT: TODO更新
export async function PUT(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const body = await request.json();
    const { title, description, completed } = body;
    const { id } = params;

    // TODOの存在確認
    const existingTodo = await prisma.todo.findUnique({
      where: { id },
    });

    if (!existingTodo) {
      return NextResponse.json(
        { error: 'TODOが見つかりません' },
        { status: 404 }
      );
    }

    // 更新データの準備
    const updateData: any = {};
    if (title !== undefined) {
      if (typeof title !== 'string' || !title.trim()) {
        return NextResponse.json(
          { error: 'タイトルは必須です' },
          { status: 400 }
        );
      }
      updateData.title = title.trim();
    }
    if (description !== undefined) {
      updateData.description = description?.trim() || null;
    }
    if (completed !== undefined) {
      updateData.completed = Boolean(completed);
    }

    const todo = await prisma.todo.update({
      where: { id },
      data: updateData,
    });

    return NextResponse.json(todo);
  } catch (error) {
    console.error('TODO更新エラー:', error);
    return NextResponse.json(
      { error: 'TODOの更新に失敗しました' },
      { status: 500 }
    );
  }
}

// DELETE: TODO削除
export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const { id } = params;

    // TODOの存在確認
    const existingTodo = await prisma.todo.findUnique({
      where: { id },
    });

    if (!existingTodo) {
      return NextResponse.json(
        { error: 'TODOが見つかりません' },
        { status: 404 }
      );
    }

    await prisma.todo.delete({
      where: { id },
    });

    return NextResponse.json({ message: 'TODOが削除されました' });
  } catch (error) {
    console.error('TODO削除エラー:', error);
    return NextResponse.json(
      { error: 'TODOの削除に失敗しました' },
      { status: 500 }
    );
  }
}
```

### Step 6: APIクライアントの作成

**src/lib/api.ts**
```typescript
import { Todo, CreateTodoInput, UpdateTodoInput } from '@/types/todo';

const API_BASE = '/api';

// APIエラーハンドリング
class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message);
    this.name = 'ApiError';
  }
}

// レスポンスのハンドリング
async function handleResponse<T>(response: Response): Promise<T> {
  if (!response.ok) {
    const errorData = await response.json().catch(() => ({}));
    throw new ApiError(response.status, errorData.error || 'APIエラーが発生しました');
  }
  return response.json();
}

// TODO API クライアント
export const todoApi = {
  // TODO一覧取得
  async getTodos(): Promise<Todo[]> {
    const response = await fetch(`${API_BASE}/todos`);
    return handleResponse<Todo[]>(response);
  },

  // TODO作成
  async createTodo(input: CreateTodoInput): Promise<Todo> {
    const response = await fetch(`${API_BASE}/todos`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(input),
    });
    return handleResponse<Todo>(response);
  },

  // TODO更新
  async updateTodo(id: string, input: UpdateTodoInput): Promise<Todo> {
    const response = await fetch(`${API_BASE}/todos/${id}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(input),
    });
    return handleResponse<Todo>(response);
  },

  // TODO削除
  async deleteTodo(id: string): Promise<void> {
    const response = await fetch(`${API_BASE}/todos/${id}`, {
      method: 'DELETE',
    });
    await handleResponse<{ message: string }>(response);
  },
};
```

### Step 7: フックの更新（API版）

**src/app/hooks/useTodos.ts** を更新：
```typescript
'use client';

import { useState, useEffect } from 'react';
import { Todo, CreateTodoInput, UpdateTodoInput } from '@/types/todo';
import { todoApi } from '@/lib/api';

export const useTodos = () => {
  const [todos, setTodos] = useState<Todo[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  // TODOの読み込み
  const loadTodos = async () => {
    try {
      setLoading(true);
      setError(null);
      const todosData = await todoApi.getTodos();
      setTodos(todosData.map(todo => ({
        ...todo,
        createdAt: new Date(todo.createdAt),
        updatedAt: new Date(todo.updatedAt),
      })));
    } catch (error) {
      console.error('TODO読み込みエラー:', error);
      setError('TODOの読み込みに失敗しました');
    } finally {
      setLoading(false);
    }
  };

  // 初期データの読み込み
  useEffect(() => {
    loadTodos();
  }, []);

  // TODO作成
  const createTodo = async (input: CreateTodoInput): Promise<Todo> => {
    try {
      const newTodo = await todoApi.createTodo(input);
      const todoWithDates = {
        ...newTodo,
        createdAt: new Date(newTodo.createdAt),
        updatedAt: new Date(newTodo.updatedAt),
      };
      setTodos(prev => [todoWithDates, ...prev]);
      return todoWithDates;
    } catch (error) {
      console.error('TODO作成エラー:', error);
      throw error;
    }
  };

  // TODO更新
  const updateTodo = async (id: string, input: UpdateTodoInput): Promise<Todo | null> => {
    try {
      const updatedTodo = await todoApi.updateTodo(id, input);
      const todoWithDates = {
        ...updatedTodo,
        createdAt: new Date(updatedTodo.createdAt),
        updatedAt: new Date(updatedTodo.updatedAt),
      };
      
      setTodos(prev => prev.map(todo => 
        todo.id === id ? todoWithDates : todo
      ));
      
      return todoWithDates;
    } catch (error) {
      console.error('TODO更新エラー:', error);
      throw error;
    }
  };

  // TODO削除
  const deleteTodo = async (id: string): Promise<boolean> => {
    try {
      await todoApi.deleteTodo(id);
      setTodos(prev => prev.filter(todo => todo.id !== id));
      return true;
    } catch (error) {
      console.error('TODO削除エラー:', error);
      throw error;
    }
  };

  // 完了状態の切り替え
  const toggleComplete = async (id: string): Promise<Todo | null> => {
    const todo = todos.find(t => t.id === id);
    if (!todo) return null;
    
    return updateTodo(id, { completed: !todo.completed });
  };

  return {
    todos,
    loading,
    error,
    createTodo,
    updateTodo,
    deleteTodo,
    toggleComplete,
    reload: loadTodos,
  };
};
```

## UIの改善とスタイリング

### Step 1: レスポンシブデザインの改善

既存のコンポーネントにさらなる改善を加えます。

**src/components/ErrorBoundary.tsx**
```typescript
'use client';

import React from 'react';

interface ErrorBoundaryState {
  hasError: boolean;
  error?: Error;
}

interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ComponentType<{ error: Error; reset: () => void }>;
}

export class ErrorBoundary extends React.Component<ErrorBoundaryProps, ErrorBoundaryState> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('エラーバウンダリでエラーをキャッチしました:', error, errorInfo);
  }

  reset = () => {
    this.setState({ hasError: false, error: undefined });
  };

  render() {
    if (this.state.hasError && this.state.error) {
      const FallbackComponent = this.props.fallback || DefaultErrorFallback;
      return <FallbackComponent error={this.state.error} reset={this.reset} />;
    }

    return this.props.children;
  }
}

const DefaultErrorFallback: React.FC<{ error: Error; reset: () => void }> = ({ error, reset }) => (
  <div className="min-h-screen bg-gray-100 flex items-center justify-center p-4">
    <div className="bg-white rounded-lg shadow-md p-8 max-w-md w-full text-center">
      <div className="text-red-500 text-6xl mb-4">⚠️</div>
      <h2 className="text-2xl font-bold text-gray-900 mb-4">エラーが発生しました</h2>
      <p className="text-gray-600 mb-6">
        申し訳ございません。予期しないエラーが発生しました。
      </p>
      <details className="text-left mb-6">
        <summary className="cursor-pointer text-sm text-gray-500 hover:text-gray-700">
          エラーの詳細
        </summary>
        <pre className="mt-2 text-xs bg-gray-100 p-2 rounded overflow-auto">
          {error.message}
        </pre>
      </details>
      <button
        onClick={reset}
        className="px-6 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 focus:outline-none focus:ring-2 focus:ring-blue-500"
      >
        再試行
      </button>
    </div>
  </div>
);
```

### Step 2: ローディング状態の改善

**src/components/LoadingSpinner.tsx**
```typescript
interface LoadingSpinnerProps {
  size?: 'sm' | 'md' | 'lg';
  text?: string;
}

export const LoadingSpinner: React.FC<LoadingSpinnerProps> = ({ 
  size = 'md', 
  text = '読み込み中...' 
}) => {
  const sizeClasses = {
    sm: 'h-4 w-4',
    md: 'h-8 w-8',
    lg: 'h-12 w-12',
  };

  return (
    <div className="flex flex-col items-center justify-center p-4">
      <div className={`animate-spin rounded-full border-b-2 border-blue-500 ${sizeClasses[size]}`} />
      {text && <p className="mt-2 text-gray-600 text-sm">{text}</p>}
    </div>
  );
};
```

### Step 3: アニメーション効果の追加

**src/app/globals.css** に追加：
```css
/* カスタムアニメーション */
@keyframes slideIn {
  from {
    opacity: 0;
    transform: translateY(-20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes slideOut {
  from {
    opacity: 1;
    transform: translateY(0);
  }
  to {
    opacity: 0;
    transform: translateY(-20px);
  }
}

.slide-in {
  animation: slideIn 0.3s ease-out;
}

.slide-out {
  animation: slideOut 0.3s ease-out;
}

/* TODOアイテムのホバー効果 */
.todo-item {
  transition: all 0.2s ease-in-out;
}

.todo-item:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

/* フォーカス状態の改善 */
.focus-ring:focus {
  outline: none;
  ring: 2px;
  ring-color: #3b82f6;
  ring-offset: 2px;
}
```

## Cloudflare Workersへのデプロイ準備

### Step 1: Cloudflare Workers用の設定

1. **Wranglerのインストール**
   ```bash
   npm install -D wrangler
   ```

2. **wrangler.toml**の作成
   ```toml
   name = "my-todo-app"
   main = "src/index.js"
   compatibility_date = "2024-01-01"

   [[d1_databases]]
   binding = "DB"
   database_name = "todo-app-db"
   database_id = "your-database-id-here"

   [vars]
   NODE_ENV = "production"
   ```

### Step 2: Next.js設定の調整

**next.config.js**
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    runtime: 'edge',
  },
  images: {
    unoptimized: true,
  },
  trailingSlash: true,
  output: 'export',
  distDir: 'dist',
}

module.exports = nextConfig
```

### Step 3: Cloudflare D1データベースのセットアップ

1. **D1データベースの作成**
   ```bash
   wrangler d1 create todo-app-db
   ```

2. **wrangler.tomlの更新**
   上記で取得したdatabase_idを設定

3. **スキーマの適用**
   ```bash
   wrangler d1 execute todo-app-db --file=./schema.sql
   ```

**schema.sql**
```sql
CREATE TABLE todos (
    id TEXT PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT,
    completed INTEGER DEFAULT 0,
    createdAt TEXT DEFAULT CURRENT_TIMESTAMP,
    updatedAt TEXT DEFAULT CURRENT_TIMESTAMP
);
```

### Step 4: Edge Runtime対応の修正

**src/lib/db-edge.ts**
```typescript
// Edge Runtime用のデータベースクライアント
interface CloudflareEnv {
  DB: D1Database;
}

export interface Todo {
  id: string;
  title: string;
  description?: string;
  completed: boolean;
  createdAt: string;
  updatedAt: string;
}

export class TodoService {
  constructor(private db: D1Database) {}

  async getAllTodos(): Promise<Todo[]> {
    const result = await this.db.prepare(
      'SELECT * FROM todos ORDER BY createdAt DESC'
    ).all();
    
    return result.results.map(this.mapRowToTodo);
  }

  async createTodo(title: string, description?: string): Promise<Todo> {
    const id = crypto.randomUUID();
    const now = new Date().toISOString();
    
    await this.db.prepare(
      'INSERT INTO todos (id, title, description, completed, createdAt, updatedAt) VALUES (?, ?, ?, ?, ?, ?)'
    ).bind(id, title, description || null, 0, now, now).run();
    
    return {
      id,
      title,
      description,
      completed: false,
      createdAt: now,
      updatedAt: now,
    };
  }

  async updateTodo(id: string, updates: Partial<Pick<Todo, 'title' | 'description' | 'completed'>>): Promise<Todo | null> {
    const now = new Date().toISOString();
    const fields: string[] = [];
    const values: any[] = [];

    if (updates.title !== undefined) {
      fields.push('title = ?');
      values.push(updates.title);
    }
    if (updates.description !== undefined) {
      fields.push('description = ?');
      values.push(updates.description);
    }
    if (updates.completed !== undefined) {
      fields.push('completed = ?');
      values.push(updates.completed ? 1 : 0);
    }
    
    fields.push('updatedAt = ?');
    values.push(now);
    values.push(id);

    const query = `UPDATE todos SET ${fields.join(', ')} WHERE id = ?`;
    await this.db.prepare(query).bind(...values).run();

    const result = await this.db.prepare('SELECT * FROM todos WHERE id = ?').bind(id).first();
    return result ? this.mapRowToTodo(result) : null;
  }

  async deleteTodo(id: string): Promise<boolean> {
    const result = await this.db.prepare('DELETE FROM todos WHERE id = ?').bind(id).run();
    return result.changes > 0;
  }

  private mapRowToTodo(row: any): Todo {
    return {
      id: row.id,
      title: row.title,
      description: row.description,
      completed: Boolean(row.completed),
      createdAt: row.createdAt,
      updatedAt: row.updatedAt,
    };
  }
}
```

### Step 5: API ルートの Edge Runtime 対応

**src/app/api/todos/route.ts** を更新：
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { TodoService } from '@/lib/db-edge';

export const runtime = 'edge';

// GET: TODO一覧取得
export async function GET(request: NextRequest) {
  try {
    const env = process.env as any;
    const todoService = new TodoService(env.DB);
    const todos = await todoService.getAllTodos();
    
    return NextResponse.json(todos);
  } catch (error) {
    console.error('TODO取得エラー:', error);
    return NextResponse.json(
      { error: 'TODOの取得に失敗しました' },
      { status: 500 }
    );
  }
}

// POST: TODO作成
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { title, description } = body;

    if (!title || typeof title !== 'string' || !title.trim()) {
      return NextResponse.json(
        { error: 'タイトルは必須です' },
        { status: 400 }
      );
    }

    const env = process.env as any;
    const todoService = new TodoService(env.DB);
    const todo = await todoService.createTodo(title.trim(), description?.trim());

    return NextResponse.json(todo, { status: 201 });
  } catch (error) {
    console.error('TODO作成エラー:', error);
    return NextResponse.json(
      { error: 'TODOの作成に失敗しました' },
      { status: 500 }
    );
  }
}
```

## デプロイとテスト

### Step 1: プロダクションビルドの確認

1. **ローカルでのビルドテスト**
   ```bash
   npm run build
   npm run start
   ```

2. **型チェック**
   ```bash
   npx tsc --noEmit
   ```

3. **ESLintチェック**
   ```bash
   npm run lint
   ```

### Step 2: Cloudflare Workers へのデプロイ

1. **Cloudflareアカウントの認証**
   ```bash
   wrangler login
   ```

2. **デプロイの実行**
   ```bash
   wrangler deploy
   ```

3. **デプロイ後の確認**
   - デプロイされたURLにアクセス
   - 全ての機能が正常に動作することを確認

### Step 3: 本番環境でのテスト

1. **基本機能のテスト**
   - TODO作成
   - TODO一覧表示
   - TODO編集
   - TODO削除
   - 完了状態の切り替え

2. **パフォーマンステスト**
   - ページ読み込み速度
   - API レスポンス時間

3. **レスポンシブデザインの確認**
   - モバイル端末での表示
   - タブレットでの表示

## トラブルシューティング

### 開発中によくある問題

#### 1. Nodeモジュールの問題

**エラー**: `Module not found` エラー

**解決策**:
```bash
# node_modules を削除して再インストール
rm -rf node_modules package-lock.json
npm install

# または yarn の場合
rm -rf node_modules yarn.lock
yarn install
```

#### 2. TypeScript エラー

**エラー**: 型定義関連のエラー

**解決策**:
```bash
# 型定義を確認
npx tsc --noEmit

# 型定義ファイルの再生成
npm run build
```

#### 3. Prisma の問題

**エラー**: データベース接続エラー

**解決策**:
```bash
# Prisma クライアントの再生成
npx prisma generate

# データベースのリセット
npx prisma db push --force-reset
```

#### 4. Cloudflare Workers デプロイエラー

**エラー**: デプロイ時のエラー

**解決策**:
1. **設定ファイルの確認**
   - `wrangler.toml` の設定を確認
   - 環境変数の設定を確認

2. **D1データベースの確認**
   ```bash
   wrangler d1 list
   wrangler d1 info todo-app-db
   ```

3. **ログの確認**
   ```bash
   wrangler tail
   ```

### パフォーマンス最適化

#### 1. 画像の最適化

**next.config.js** で画像の設定を調整：
```javascript
module.exports = {
  images: {
    formats: ['image/webp'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920, 2048, 3840],
  },
}
```

#### 2. バンドルサイズの最適化

```bash
# バンドルアナライザーの追加
npm install @next/bundle-analyzer

# 分析の実行
ANALYZE=true npm run build
```

#### 3. キャッシュ戦略

**src/app/api/todos/route.ts** でキャッシュヘッダーを追加：
```typescript
export async function GET() {
  // ... existing code ...
  
  const response = NextResponse.json(todos);
  response.headers.set('Cache-Control', 'public, s-maxage=60, stale-while-revalidate=300');
  return response;
}
```

### セキュリティ対策

#### 1. 入力値の検証

**src/lib/validation.ts**
```typescript
export const validateTodo = (data: any) => {
  const errors: string[] = [];
  
  if (!data.title || typeof data.title !== 'string' || data.title.trim().length === 0) {
    errors.push('タイトルは必須です');
  }
  
  if (data.title && data.title.length > 200) {
    errors.push('タイトルは200文字以内で入力してください');
  }
  
  if (data.description && data.description.length > 1000) {
    errors.push('説明は1000文字以内で入力してください');
  }
  
  return {
    isValid: errors.length === 0,
    errors,
  };
};
```

#### 2. XSS対策

コンポーネントでは常にエスケープされた値を使用：
```typescript
// 危険 - 使用しない
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// 安全
<div>{userInput}</div>
```

#### 3. CSRF対策

API ルートでCSRFトークンを検証（必要に応じて実装）

## まとめ

このチュートリアルでは、Next.js を使用したTODO管理Webアプリケーションの開発からCloudflare Workersへのデプロイまでを詳細に解説しました。

### 学習したポイント

1. **Next.js の基本**
   - App Router の使用方法
   - TypeScript との統合
   - API ルートの作成

2. **React の状態管理**
   - カスタムフックの作成
   - useEffect と useState の活用

3. **データベース統合**
   - Prisma の使用方法
   - SQLite から Cloudflare D1 への移行

4. **UI/UX の改善**
   - Tailwind CSS を使用したスタイリング
   - レスポンシブデザイン
   - アニメーション効果

5. **デプロイメント**
   - Cloudflare Workers への配置
   - 本番環境での最適化

### 次のステップ

このアプリケーションをさらに拡張するための提案：

1. **ユーザー認証の追加**
   - Auth0 や Clerk を使用した認証システム
   - ユーザーごとのTODO管理

2. **検索・フィルタリング機能**
   - タイトルや説明での検索
   - 日付でのフィルタリング
   - タグ機能の追加

3. **通知機能**
   - 期限が近いTODOの通知
   - プッシュ通知の実装

4. **コラボレーション機能**
   - TODOの共有
   - リアルタイム更新

5. **モバイルアプリ化**
   - PWA の実装
   - オフライン対応

このチュートリアルが、モダンなWebアプリケーション開発の基礎を理解する助けになれば幸いです。開発を続ける中で疑問点があれば、公式ドキュメントやコミュニティを活用して学習を深めてください。