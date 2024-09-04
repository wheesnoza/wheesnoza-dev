---
title: Laravelで少し高度な絞り込み検索機能の実装方法
author: wheesnoza
pubDatetime: 2024-09-04T11:34:19Z
slug: implement-a-advanced-laravel-filter-feature
featured: true
draft: false
tags:
  - Laravel
  - QueryBuilder
  - Filter
ogImage: ""
description: PHPとLaravelを用いて、商品検索の絞り込み機能の実装方法を解説します。具体的には、検索条件を受け取り、条件に基づいてデータベースから商品を絞り込みする方法を紹介します。
canonicalURL: https://zenn.dev/arsaga/articles/ad20f42ed399ed
---

# はじめに

Webアプリケーションで検索機能を実装する際、絞り込み機能は重要な役割を果たします。私は、過去に仕事で実際に長いコードが書かれていた経験があり、そのコードの拡張性や管理性に課題を感じていました。そこで、今回はより使いやすく拡張性の高い絞り込み機能の実装方法について紹介します。

本記事では、PHPとLaravelを用いて、商品検索の絞り込み機能の実装方法を解説します。具体的には、検索条件を受け取り、条件に基づいてデータベースから商品を絞り込みする方法を紹介します。

# 前提条件

絞り込み機能を実現するために以下のようなURIのクエリパラメータを使います。
複数の絞り込みが適用されると想定して`filter`というキーで全ての条件を管理します。

```
http://localhots/api/products?filter[raiting]=4.5
http://localhots/api/products?filter[raiting]=4.5&filter[size:in]=L,LL
```

ちなみにこちらの構造は[JSON:API](https://jsonapi.org/)を参考にした書き方です。
メリットとしてはLaravel側の`FormRequest`クラスで以下のように簡単にまとまった形で条件を取得することができます。

```php
// コントローラーで受け取ったリクエストのfilterをCollection化して取得する
$request->collect('filter');

[
    "raiting" => 4.5,
    "size:in" => "L,LL",
]
```

# 実装

実際にクラスとそのロジックの解説をしていこうと思うのですが、その前に少し絞り込み条件の構造について触れていこうと思います。

商品には次の項目があります。

1. 名前
2. 値段
3. 評価
4. サイズ
5. 送料無料フラグ

上記を絞り込みしようと考えたと時に「絞り込み対象」と「絞り込み対象の条件値」と考えることができます。

前提条件でも説明した以下のような構造になります。

```php
[
    "name" => "ジャケット", // 商品名で絞り込み
    "price" => 30000, // 値段が30000以下の商品で絞り込み
    "raiting" => 4.5, // 商品評価が4.5以上で絞り込み
    "free_shipping" => true, // 配送が無料の商品だけ絞り込む
]
```

絞り込みの「対象」に対しての「条件」を表現できます。
各対象は違ったデータ型の値を持っていても考え方は同じで、「対象」と「条件」を持ちます。そのため対象を表現する`Filter`という抽象クラスを用意し、条件を表現するvalueを外から受け取り適切に処理するようにします。

:::message
「各対象は違ったデータ型の値を持つ」問題に関しては今回の記事の続きとして別の記事で紹介しようと思います。
:::

では、実際に実装していきましょう。

## Filterクラスを用意する

Filterクラスを以下のように定義します。

```php: Filter.php
<?php

namespace App\Filters;

use Illuminate\Database\Eloquent\Builder;

abstract class Filter
{
    public function __construct(protected readonly mixed $value)
    {
    }

    abstract public function handle(Builder $query): void;
}
```

説明しますと`new`する時にコンストラクターに条件値を渡します。継承するクラス自体が絞り込み対象を表現しているためvalueだけの処理を行えれば問題ないです。

また、LaravelなのでEloquent Query Builderを前提とした実装になっています。`handle`は外から受け取るクエリに条件を追加するための関数です。

例えば`Product::query()`や`Product::with('')`などを渡す想定です。`handle`の中でクエリを操作して絞り込み条件が追加されるようにします。

コードで表現すると以下のような感じになります。

```
(new 条件対象クラス(条件値))->handle($query);
```

## 条件対象のクラスを実装する

Filterクラスを継承する具体的な絞り込み対象のクラスを実装します。例えば商品の評価絞り込みのクラスの場合は以下のようになります。

```php: ProductRaitingFilter.php
<?php

namespace App\Filters\Product;

use Illuminate\Database\Eloquent\Builder;
use App\Filters\Filter;

class ProductRaitingFilter extends Filter
{
    public function handle(Builder $query): void
    {
        $query->where('raiting', '>=', $this->value);
    }
}

```

説明しますとFilterクラスを継承している`ProductRaitingFilter`を用意してhandleでクエリにその条件をwhereで追加します。Filterクラスを継承しているのでprotectedな`value`にアクセスできます。

実際の動きとしては以下のようになります。
最初は`$query`のSQL文としては条件なしのselect文なのですが、Filterクラスで`$query`を操作しているため条件が追加されます。

```bash
> $query = Product::query();
= Illuminate\Database\Eloquent\Builder {#}

> $query->toSql();
= "select * from `products`"

> $filter = new ProductRaitingFilter(4.5);
= App\Filters\Product\ProductRaitingFilter {#}

> $filter->handle($query);
= null

> $query->toSql()
= "select * from `products` where `raiting` >= ?"
```

ついでに商品のサイズの複数指定の条件クラスも用意しましょう。

```php: ProductSizeInFilter.php
<?php

namespace App\Filters\Product;

use Illuminate\Database\Eloquent\Builder;
use App\Filters\Filter;

class ProductSizeInFilter extends Filter
{
    public function handle(Builder $query): void
    {
        $query->whereIn('size', explode(',', $this->value));
    }
}

```

評価の条件と流れは同じです。1つ違うのは複数のサイズをカンマ区切りで指定されるのでそれらを配列に変換している箇所です。

ちなみに別記事では条件値の型の扱いも説明する予定ですが、とりあえず条件クラス内で配列変換しても問題ないと思います。

これで2つの条件クラスが出来上がりました。動きとしては以下のようになります。

```bash
> $query = Product::query();
= Illuminate\Database\Eloquent\Builder {#}

> $query->toSql();
= "select * from `products`"

> $raitingFilter = new ProductRaitingFilter(4.5);
= App\Filters\Product\ProductRaitingFilter {#}

> $raitingFilter->handle($query);
= null

> $query->toSql()
= "select * from `products` where `raiting` >= ?"

> $sizeInFilter = new ProductSizeInFilter('L,LL');
= App\Filters\Product\ProductSizeInFilter {#}

> $sizeInFilter->handle($query);
= null

> $query->toSql()
= "select * from `variants` where `raiting` >= ? and `size` in (?, ?)"
```

このようにクエリが作られていきます。

## 動的に条件対象を返す仕組み

少し前に以下のようなリクエストが来ると話したことを覚えていますか？

```php
[
    "name" => "ジャケット", // 商品名で絞り込み
    "price" => 30000, // 値段が30000以下の商品で絞り込み
    "raiting" => 4.5, // 商品評価が4.5以上で絞り込み
    "free_shipping" => true, => // 配送が無料の商品だけ絞り込む
]
```

ユーザーが指定した条件を実装した条件クラスに当てはめる必要があります。

例えばif文で`filter`の持つキーバリューをループで回して適切な条件クラスを実行する。

```php
$query = Product::query();
foreach ($request->collect('filter') as $name => $value) {
    if ($name === 'raiting') {
        $filter = new ProductRaitingFilter($value);
        $filter->handle($query);
    }
    if ($name === 'size:in') {
        $filter = new ProductSizeInFilter($value);
        $filter->handle($query);
    }
}
```

ただ、いくつか問題点があります。
あまりスマートではないのと条件が増えれば増えるほど分岐が多くなりコードが長くなってしまいます。
また、ユーザーが指定する条件を予測できないのでif文で全てまかなうのは難しいと思います。

そこで、動的にキーから適切に該当する条件クラスを生成して返すような仕組みが必要になります。

採用した方法としてはPHP8.1から使えるEnumを使った方法です。

ProductFiltersというEnumを用意し、create関数で条件値を受け取り適切な条件クラスを作成して返すようにします。

```php: ProductFilters.php
<?php

namespace App\Enums\Product;

use App\Filters\Filter;
use App\Filters\Product\ProductRaitingFilter;
use App\Filters\Product\ProductSizeInFilter;

enum ProductFilters: string
{
    case Raiting = 'raiting';
    case SizeIn = 'size:in';

    public function create(mixed $value): Filter
    {
        return match ($this) {
            self::Raiting => new ProductRaitingFilter($value),
	    self::SizeIn => new ProductSizeInFilter($value),
        };
    }
}
```

使い方としてはEnumの`from`を使って該当する条件クラスを取得します。

```php
> ProductFilters::from('raiting')->create(4.5)
= App\Filters\Product\ProductRaitingFilter {#}

> ProductFilters::from('size:in')->create('L,LL')
= App\Filters\Product\ProductSizeInFilter {#}
```

このように動的にキー名から適切な条件クラスを取得することができます。
あとはリクエストから受け取った`filter`をループで回しながらクエリに条件を加えていくだけです。

```php
$query = Product::query();
$filters = $request->collect('filter');

foreach ($filters as $name => $value) {
    $filter = ProductFilters::from($name)->create($value);

    $filter->handle($query);
}

return $query->get(); // すべての指定された条件で絞り込みされます
```

上記を1つのクラスにまとめてもいいと思います。

```php: HandleFilterProductAction.php
<?php

namespace App\Actions\Product;

class HandleFilterProductAction
{
    public static function execute(Builder $query, Collection $filters): void
    {
        foreach ($filters as $name => $value) {
            $filter = ProductFilters::from($name)->create($value);

            $filter->handle($query);
        }
    }
}
```

そうすることでコントローラーがすっきりします。

```php: ProductController.php
<?php

namespace App\Controllers\Api\Product;

class ProductController extends Controller
{
    public function index(Request $request)
    {
        $query = Product::query();

        HandleFilterProductAction::execute(
            $query,
            $request->collect('filters')
        );

        return $query->get(); // paginateも可能
    }
}
```

ここで実装に1つの問題があります。ユーザーがURIに何を入力するか予想できないため存在しない絞り込み対象を指定されたらEnum側で`Value Error`になります。

```
ValueError  "test" is not a valid backing value for enum App\Enums\Product\ProductFilters.
```

もちろん例外処理で対応もできますが、今回は`tryFrom`で単にnullを返すようにしました。

```php
class HandleFilterProductAction
{
    public static function execute(Builder $query, Collection $filters): void
    {
        foreach ($filters as $name => $value) {
            $filter = ProductFilters::tryFrom($name);

            if (!is_null($filter)) {
                $filter->create($value)->handle($query);
            }
        }
    }
}
```

これで以下のURIのような複数条件に柔軟に対応できます。

```
http://localhots/api/products?filter[raiting]=4.5&filter[size:in]=L,LL
```

条件が増えてもクラスを追加してEnumに登録すればあとは自動で実行されます。

```
http://localhots/api/products?filter[raiting]=4.5&filter[size:in]=L,LL&name=ジャケット&free_shipping=true
```

もちろん適当な条件がURIに含まれていたとしても先ほどの修正でスルーされます。

```
http://localhots/api/products?filter[raiting]=4.5&filter[size:in]=L,LL&filter[hogehoge]=test
```

# まとめ

以上でLaravelで絞り込み機能を実装する方法でした。

メリットをまとめますと、以下が挙げられます。

1. 条件の管理のしやすさ
2. 責務毎に分けられているため変更に強い
3. 1クラスのコード量が減って読みやすい
4. 条件分岐を減らせる
5. 柔軟に条件の追加、削除、編集ができる

# 参考文献

- [Laravel Concepts of Martin Joo](https://laravel-concepts.io/)
- [How To Use Laravel Pipelines To Implement More Advanced Filters](https://martinjoo.dev/how-to-use-laravel-pipelines-to-implement-more-advanced-filters)
- [Strategy in PHP](https://refactoring.guru/design-patterns/strategy/php/example)
- [Factory Method in PHP](https://refactoring.guru/design-patterns/factory-method/php/example)

# 最後に

今回の実装はリポジトリにまとめてあります。
https://github.com/wheesnoza/laravel-advanced-filter

改良と修正を行っているので多少実装に違いがあると思いますが、考え方は同じです。

少しでも参考になって面白い実装の手助けになればと思います。

最後まで読んでいただきありがとうございます！
