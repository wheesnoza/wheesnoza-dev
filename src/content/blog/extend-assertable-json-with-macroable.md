---
title: Macroableを使ってAssertableJsonを拡張する
author: wheesnoza
pubDatetime: 2024-09-05T01:34:19Z
slug: extend-assertable-json-with-macroable
featured: true
draft: false
tags:
  - Laravel
  - AssertableJson
  - Testing
  - API
  - json
  - Macroable
  - laravel-assertable-json-enhancer
ogImage: ""
description: この記事では、AssertableJsonをMacroableを使って拡張する方法について解説します。この記事を通じて、独自のアサーションを追加し、テストの柔軟性を高める手法を学んでいただければと思います。
---

# はじめに

LaravelでのAPI開発において、テストは非常に重要です。その中でも、APIレスポンスを検証するために使われる**AssertableJson**は、シンプルで強力なツールです。しかし、実際のプロジェクトで複雑なレスポンスをテストしようとすると、既存のアサーションでは物足りないと感じることがあります。

例えば、特定のキーが特定の形式であることを確認したい場合や、カスタムロジックを適用したアサーションが必要になるケースがあるでしょう。そうしたときに役立つのが、Laravelに組み込まれている**Macroable**機能です。

この記事では、AssertableJsonをMacroableを使って拡張する方法について解説します。この記事を通じて、独自のアサーションを追加し、テストの柔軟性を高める手法を学んでいただければと思います。

## Macroableを使ったAssertableJsonの拡張

Laravelには、多くのクラスに柔軟な機能を追加できる**Macroable**という便利なトレイトがあります。これを活用することで、既存のクラスに簡単にメソッドを追加することができ、テストや開発の効率を大幅に向上させることが可能です。

### Macroableとは？

Macroableは、Laravelのクラスに対して「マクロ」を登録することで、新しいメソッドを簡単に追加できるトレイトです。これにより、Laravelのコアクラスに手を加えることなく、プロジェクトのニーズに応じてカスタムメソッドを作成することが可能になります。たとえば、Laravelのコレクションや、リクエストオブジェクトに対して独自のメソッドを追加したい場合に非常に役立ちます。

### AssertableJsonの拡張にMacroableを使う理由

APIレスポンスの検証において、標準のアサーションメソッドだけでは十分でないと感じる場面は少なくありません。例えば、以下のようなカスタムアサーションを行いたいケースが考えられます。

- 特定の配列が一定の要素数を持つかどうかを検証したい
- JSONの特定のキーが正規表現にマッチするかどうかをチェックしたい
- 複雑なネスト構造のデータが正しいかどうかを一度に確認したい

こうしたニーズに対応するために、Macroableを使ってAssertableJsonにカスタムメソッドを追加することで、これらの検証をシンプルかつ直感的に行うことができます。以下では、その実装手順について具体的に見ていきましょう。

### Macroableを使った独自アサーションの追加手順

まず、LaravelでMacroableを使用して、AssertableJsonに新しいメソッドを追加する基本的な方法を紹介します。次のコード例では、AssertableJsonに「whereIsArray」というメソッドを追加し、指定されたキーが配列であることを確認するアサーションを実装します。

```php
use Illuminate\Testing\Fluent\AssertableJson;

AssertableJson::macro('whereIsArray', function (string $key) {
    return $this->where($key, fn ($value) => is_array($value));
});
```

このコードでは、`AssertableJson`クラスに対して`macro`メソッドを呼び出し、`whereIsArray`という新しいメソッドを追加しています。このメソッドは、指定されたキーに対応する値が配列であることを確認するシンプルなアサーションです。

### Macroの定義場所について

Macroの定義は、Laravelのサービスプロバイダで行うのが一般的です。特に、`AppServiceProvider`の`boot`メソッド内に定義する方法が推奨されます。これにより、アプリケーションの起動時にマクロが登録され、どのテストケースでも追加されたアサーションメソッドを使用することができます。

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Testing\Fluent\AssertableJson;

class AppServiceProvider extends ServiceProvider
{
    public function boot()
    {
        AssertableJson::macro('whereIsArray', function (string $key) {
            return $this->where($key, fn ($value) => is_array($value));
        });
    }
}
```

このように`AppServiceProvider`にマクロを定義することで、プロジェクト全体でこのカスタムアサーションを利用することができるようになります。

### カスタムアサーションの利用例

このカスタムメソッドを使うと、次のようにテストで利用できます。

```php
$response = $this->getJson('/api/some-endpoint');

$response->assertJson(function (AssertableJson $json) {
    $json->whereIsArray('data.items');
});
```

このテストでは、`data.items`キーが配列であることを確認しています。このように、Macroableを活用することで、プロジェクトに合わせたアサーションを簡単に追加でき、より柔軟なテストを実現することが可能です。

## さらに複雑なアサーションの追加と応用例

前のセクションでは、Macroableを使ってシンプルなカスタムアサーションを追加する方法を紹介しました。このセクションでは、さらに複雑なアサーションを追加し、実際のプロジェクトでどのように活用できるかを見ていきます。

### 例：ネストされた配列やオブジェクトの検証

APIレスポンスが複雑になると、ネストされた配列やオブジェクトを検証する必要が出てきます。標準のアサーションだけでは、これらを効率的にテストするのが難しい場合があります。そこで、Macroableを活用して、ネストされたデータを検証するためのカスタムメソッドを追加しましょう。

以下は、JSONレスポンス内の特定のキーが存在し、そのキーの値が指定された数以上の要素を持つ配列であることを確認するアサーションを追加する例です。

```php
use Illuminate\Testing\Fluent\AssertableJson;

AssertableJson::macro('whereArrayHasAtLeast', function (string $key, int $minCount) {
    return $this->where($key, fn ($value) => is_array($value) && count($value) >= $minCount);
});
```

この`whereArrayHasAtLeast`メソッドは、指定されたキーが配列であり、その配列の要素数が`$minCount`以上であることを検証します。これにより、例えば少なくとも3つのアイテムを持つ配列を確認したい場合に便利です。

### 実際のプロジェクトでの活用例

このカスタムアサーションを使った具体的なテストコードを見てみましょう。

```php
$response = $this->getJson('/api/products');

$response->assertJson(function (AssertableJson $json) {
    $json->whereArrayHasAtLeast('data.products', 3);
});
```

この例では、`/api/products`エンドポイントから返されるレスポンスの`data.products`キーが少なくとも3つの要素を持つ配列であることを確認しています。このような検証は、商品のリストやユーザーのリストなど、一定以上のデータが返されることを期待するケースで非常に有効です。

### より高度な検証：正規表現やカスタムロジックの適用

さらに高度な検証が必要な場合、正規表現やカスタムロジックを適用することもできます。例えば、特定のキーの値が特定のパターンにマッチするかどうかを検証するアサーションを作成することができます。

```php
AssertableJson::macro('whereMatchesPattern', function (string $key, string $pattern) {
    return $this->where($key, fn ($value) => preg_match($pattern, $value));
});
```

この`whereMatchesPattern`メソッドは、指定されたキーの値が正規表現にマッチするかどうかを確認します。例えば、メールアドレスや電話番号の形式を確認する場合に役立ちます。

```php
$response = $this->getJson('/api/user');

$response->assertJson(function (AssertableJson $json) {
    $json

->whereMatchesPattern('data.email', '/^[\w\.-]+@[\w\.-]+\.\w{2,4}$/');
});
```

このテストでは、`data.email`キーの値がメールアドレスの形式であることを確認しています。このように、Macroableを使って柔軟なアサーションを追加することで、プロジェクトの特定のニーズに応じたテストを簡単に作成することが可能です。

## 拡張パッケージassertable-json-enhancerの紹介

ここまで、LaravelのMacroableを使ってAssertableJsonを拡張し、独自のアサーションを追加する方法を紹介してきました。これにより、プロジェクトに特化したテストを柔軟に作成できるようになりました。しかし、これらのカスタムアサーションを毎回手動で追加するのは少し面倒に感じるかもしれません。

そこで、私が公開している[assertable-json-enhancer](https://github.com/wheesnoza/assertable-json-enhancer)パッケージを使うことで、これまで紹介したようなカスタムアサーションを簡単に利用できるようになります。このパッケージは、すでに役立つアサーションがいくつか含まれており、必要に応じてすぐに使える形で提供されています。

### 追加されるアサーションの具体例

このパッケージには、前のセクションで紹介したようなアサーションを含め、複数の便利なアサーションが含まれています。例えば、`whereIsArray`や`whereArrayHasAtLeast`といったメソッドはもちろん、他にも正規表現マッチングや複雑なネスト構造を簡単に検証できるアサーションが用意されています。

インストール後、以下のように使うことができます。

```php
$response = $this->getJson('/api/products');

$response->assertJson(function (AssertableJson $json) {
    $json->whereIsArray('data.items')
         ->whereArrayHasAtLeast('data.items', 3)
         ->whereMatchesPattern('data.email', '/^[\w\.-]+@[\w\.-]+\.\w{2,4}$/');
});
```

このコードでは、`assertable-json-enhancer`パッケージによって提供されるカスタムアサーションを活用しています。これにより、コードの読みやすさと保守性が向上し、より効率的にテストを行うことができます。

### まとめ

Macroableを使って自分でカスタムアサーションを作成する方法もありますが、既に用意されたパッケージを活用することで、開発時間を節約し、テストの信頼性を高めることが可能です。

このパッケージは、オープンソースでGitHubに公開されています。
また、このパッケージはLaravel 11以降をサポートしており、PHP 8.3以上のバージョンが必要です。必要に応じて拡張したり、フィードバックを提供したりすることもできます。
