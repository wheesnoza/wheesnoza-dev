---
title: Laravelのカスタムクエリビルダーを使った柔軟なクエリの作成方法
author: wheesnoza
pubDatetime: 2024-09-05T23:34:19Z
slug: how-to-create-flexible-queries-using-laravel-custom-query-builder
featured: true
draft: false
tags:
  - Laravel
  - QueryBuilder
ogImage: ""
description: LaravelのEloquentモデルでは、データベースクエリを簡単に作成できます。特に「スコープ」を使用することで、特定の条件に基づいたクエリを簡潔に再利用することができますが、クエリをより柔軟にカスタマイズしたい場合には、カスタムクエリビルダーを使用する方法があります。
---

# はじめに

LaravelのEloquentモデルでは、データベースクエリを簡単に作成できます。特に「スコープ」を使用することで、特定の条件に基づいたクエリを簡潔に再利用することができますが、クエリをより柔軟にカスタマイズしたい場合には、カスタムクエリビルダーを使用する方法があります。本記事では、Laravelでカスタムクエリビルダーを作成し、それを使って柔軟なクエリを実現する方法を紹介します。

## 1. カスタムクエリビルダーとは

カスタムクエリビルダーは、LaravelのEloquentクエリビルダーを拡張して、独自のメソッドを追加するものです。これにより、特定のモデルに対して、再利用可能なクエリロジックを実装することが可能になります。

## 2. カスタムクエリビルダーの作成

まず、カスタムクエリビルダーのクラスを作成します。このクラスは、`Illuminate\Database\Eloquent\Builder` クラスを拡張し、独自のメソッドを追加します。ここでは、`Product`モデル用のクエリビルダークラスを例にします。

```php
<?php

namespace App\Models\QueryBuilders;

use Illuminate\Database\Eloquent\Builder;

class ProductBuilder extends Builder
{
    public function active()
    {
        return $this->where('status', 'active');
    }

    public function withLatestOrder()
    {
        return $this->with(['orders' => function ($query) {
            $query->orderBy('created_at', 'desc')->limit(1);
        }]);
    }

    public function popular($threshold = 100)
    {
        return $this->where('popularity', '>=', $threshold);
    }
}
```

上記の例では、`active`、`withLatestOrder`、`popular`という3つのカスタムメソッドを追加しています。それぞれ、`Product`モデルに対して特定のクエリ条件を追加します。

## 3. モデルでカスタムクエリビルダーを使用する

次に、作成したカスタムクエリビルダーをモデルに適用します。モデル内で`newEloquentBuilder`メソッドをオーバーライドし、カスタムクエリビルダーのインスタンスを返すようにします。また、PHPDocを使って、`query()`メソッドの返り値の型を指定し、エディターでの補完を効かせる方法も説明します。

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use App\Models\QueryBuilders\ProductBuilder;

/**
 * @method static ProductBuilder query()
 */
class Product extends Model
{
    protected function newEloquentBuilder($query)
    {
        return new ProductBuilder($query);
    }
}
```

このようにPHPDocコメントを追加することで、エディターが`ProductBuilder`のメソッドを補完しやすくなります。

## 4. カスタムクエリビルダーの使用例

以下は、カスタムクエリビルダーを使用してクエリを実行する例です。

```php
$products = Product::query()
    ->active()
    ->popular(150)
    ->withLatestOrder()
    ->get();
```

このクエリでは、ステータスが`active`で、人気度が150以上の製品を取得し、さらに最新の注文情報を含めています。カスタムクエリビルダーを使用することで、こうした条件を簡潔に表現し、再利用できる形で管理できます。

## 5. カスタムクエリビルダーを使用するメリット

カスタムクエリビルダーを使用することで、次のような利点があります。

- **クエリロジックの再利用**: カスタムクエリビルダー内で定義したメソッドを、モデル内で何度も再利用できます。
- **クリーンで整理されたコード**: クエリロジックをモデルとは別のクラスにまとめることで、コードの見通しが良くなり、管理がしやすくなります。
- **エディターサポート**: PHPDocを利用することで、エディターでの補完機能が向上し、コーディングの効率がアップします。

# まとめ

Laravelのカスタムクエリビルダーを使用することで、特定のモデルに対して柔軟かつ再利用可能なクエリを実装できます。これにより、クエリロジックが整理され、モデルのコードがシンプルになります。さらに、PHPDocを利用してエディターでの補完機能を強化することで、開発効率を向上させることも可能です。
