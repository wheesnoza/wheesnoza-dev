---
title: AssertableJsonでLaravel APIテストの柔軟性を向上させる方法
author: wheesnoza
pubDatetime: 2024-09-04T01:34:19Z
slug: how-improves-assertable-json-flexibility-in-laravel-api-testing
featured: true
draft: false
tags:
  - Laravel
  - AssertableJson
  - Testing
  - API
  - json
ogImage: ""
description: この記事では、Laravel 8以降導入された`AssertableJson`の機能を活用し、特に`where`メソッドに`Closure`を渡すことで、APIテストをより柔軟に行う方法をご紹介します。
---

# はじめに

Laravelは、その直感的なAPI開発機能で広く利用されています。しかし、APIのテストにおいて、標準のアサーションだけでは十分に対応できない場合もあります。私もLaravelを使った開発において、APIテストでAssertableJsonを頻繁に使用していますが、時折「もう少し柔軟にテストできれば」と感じることがありました。

この記事では、Laravel 8以降導入された`AssertableJson`の機能を活用し、特に`where`メソッドに`Closure`を渡すことで、APIテストをより柔軟に行う方法をご紹介します。

---

## AssertableJsonの基本的な使い方

まず、`AssertableJson`の基本的な使い方を確認しておきましょう。`AssertableJson`は、LaravelでAPIレスポンスのJSONをテストする際に非常に有用なアサーションメソッドを提供します。

例えば、特定のキーが期待される値を持つかどうかを確認するには、次のように`where`メソッドを使用します。

```php
$response->assertJson(fn (AssertableJson $json) =>
    $json->where('data.id', 1)
         ->where('data.name', 'John Doe')
         ->etc()
);
```

さらに、特定のキーが存在するかを確認するには、`has`メソッドが役立ちます。

```php
$response->assertJson(fn (AssertableJson $json) =>
    $json->has('data.user.id')
         ->has('data.user.email')
         ->etc()
);
```

`has`メソッドを使うことで、キーの存在を簡単に検証できます。これらの基本的なアサーションを押さえた上で、次に進んで、より高度なテスト方法を見ていきましょう。

---

## `where`メソッドの拡張: `Closure`を使った柔軟なアサーション

`AssertableJson`では、`where`メソッドの第2引数に`Closure`を渡すことができ、これによりより柔軟なテストを実現できます。単純な値比較だけでなく、複雑な条件を用いてレスポンスを検証することが可能です。

例えば、次のように`Closure`を利用して、特定の条件を満たすかどうかを確認します。ここで、`Closure`をわかりやすい変数名に置き換えることで、テストの可読性とメンテナンス性を向上させることができます。

```php
$shouldBeAdult = fn($age) => $age > 18;
$shouldBeValidName = fn($name) => is_string($name) && strlen($name) > 3;

$response->assertJson(fn (AssertableJson $json) =>
    $json->where('data.age', $shouldBeAdult)
         ->where('data.name', $shouldBeValidName)
         ->etc()
);
```

この例では、`$shouldBeAdult`という変数を使用して、`data.age`が18歳以上であることを確認し、`$shouldBeValidName`という変数を使用して、`data.name`が3文字以上の文字列であることを確認しています。変数名を付けることで、テスト内容が何を意図しているかが明確になり、他の開発者がコードを読んだときにも理解しやすくなります。

このように、`Closure`をわかりやすい変数名にして渡すことで、テストの可読性が向上し、より良いテストコードを書くことができます。

---

# まとめ

Laravelの`AssertableJson`を活用することで、APIテストの精度と柔軟性を高めることが可能です。特に、`where`メソッドの第2引数に`Closure`を渡す機能を活用することで、カスタムアサーションを簡単に実装でき、テストをさらに強化できます。また、`Closure`をわかりやすい変数名で表現することで、テストコードの可読性を向上させることができます。

Laravelの豊富な機能を最大限に活用し、より高品質なソフトウェアを提供するために、これらのテクニックをぜひ取り入れてみてください。

---

### 参考リンク

- [Laravel公式ドキュメント - Testing JSON APIs](https://laravel.com/docs/8.x/http-tests#fluent-json-testing)
- [PHP Documentation - Closures](https://www.php.net/manual/en/functions.anonymous.php)
