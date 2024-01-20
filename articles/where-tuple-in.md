---
title: "SQLのINで複数カラムを指定する方法とLaravelでの実装"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Laravel", "PHP", "SQL"]

published: true
---

# SQLのIN句に複数カラムを指定する方法について
SQLで
```sql
SELECT * FROM `users`
WHERE (`country` = 'Japan' and `language` = 'Japanese')
OR (`country` = 'US' and `language` = 'English')
OR (`country` = 'China' and `language` = 'Chinese');
```
のように複数カラムで検索をしたいときは以下のように書くことができます
```sql
SELECT * FROM `users`
WHERE (`country`, `language`) IN (
    ('Japan', 'Japanese'),
    ('US', 'English'),
    ('China', 'Chinese')
);
```
複数カラムにインデックスを貼っているテーブルでは便利なSQLです。
LaravelでこのSQLを使えるようにしたいと思います。

# Laravelで複数カラムで検索する方法
## 既存のクエリビルダメソッドを使って実装する方法
既存の`whereIn`メソッドの第一引数には配列を渡すことができません
```php
❌ DB::table('usets')->whereIn(['country', 'language'], [['Japan', 'Japanese']...]);
```
そのため1つ目のSQLのように複数の`orWhere`を繋げてあげる必要があります。
```php
DB::table('users')
    ->where([
        'country' => 'Japan',
        'language' => 'Japanese'
    ])->orWhere(fn ($query) => $query->where([
        'country' => 'US',
        'language' => 'English'
    ]))->orWhere(fn ($query) => $query->where([
        'country' => 'China',
        'language' => 'Chinese'
    ]));
```
これは条件の取得したい要素の数だけwhereOrメソッドを呼び出すことになるのでスマートではなさそうですね

## whereRawメソッドを使って実装する場合
次に`whereRaw`メソッドを使ってIN句で複数カラムを指定するSQLを生成すると以下になります。
```php
DB::table('users')
    ->whereRaw(
        '(`country`, `language`) IN ((?, ?), (?, ?), (?, ?))',
        ['Japan', 'Japanese', 'US', 'English', 'China', 'Chinese']
    );
```
:::message
SQLにバインドするために、第一引数、IN句の後にバインドする値の分だけのプレースホルダ`?`を、
第二引数にはバインドする順番通りに値を1次元配列で渡す必要があります。
:::
これで目的のSQLを生成することができましたが、値の数を数えてプレースホルダを置いたり、配列を1次元に展開したりと面倒なのでこれを汎用性の高いマクロメソッドにしていきます

## マクロでの実装
`whereMultiIn`というマクロメソッドを作成します。
```php
class WhereMultiInProvider extends ServiceProvider
{
    public function boot(): void
    {
        Builder::macro('whereMultiIn', function (array $columns, array $data) {
            /** @var Builder $query */
            $query = $this;
            $columns_str = '(' . implode(',', $columns) . ')';

            $placeholder = array_fill(0, count($columns), '?');
            $placeholders = array_fill(0, count($data), '(' . implode(',', $placeholder) . ')');
            $placeholders_str = '(' . implode(',', $placeholders) . ')';

            $bindings_str = Arr::flatten($data);

            $query->whereRaw(
                "{$columns_str} IN {$placeholders_str}",
                $bindings_str
            );
            
            return $query;
        });
    }
}
```
これでカラムの数や値の数が変わっても柔軟に複数カラムをIN句に指定するSQLを生成するメソッドを作ることができました。

このマクロを登録することで以下のようにビルダから`whereMultiIn`メソッドを呼び出すことができます
```php
DB::table('users')
    ->whereMultiIn(
        ['country', 'language'],
        [
            ['Japan', 'Japanese'],
            ['US', 'English'],
            ['China', 'Chinese']
        ]    
    );
```
さっきの`whereRaw`メソッドを使ったパターンよりも簡単にSQLが生成するようになりました👏

# 本当にやりたかったこと
本当はlaravel/framework自体にメソッドを作りたかったのですが、過去に同じことを考えた人がいたみたいで、しかもこの提案は断られているみたいでした。
https://github.com/laravel/framework/pull/37013

まだ、OSSコントリビュートをしたことなく、折角のチャンスと思っていたのですが、残念です...
この提案が断られた理由は以下の理由でした。
> Unfortunately, I'm going to delay merging this code for now. To preserve our ability to adequately maintain the framework, we need to be very careful regarding the amount of code we include.

既存コードの変更が多いからってことなんですかね？
そしたら既存コードに影響を与えないような実装にしたらいいのでしょうか？

もし既存コードに影響ださないやり方思いついたらチャレンジしてみたいですね！
