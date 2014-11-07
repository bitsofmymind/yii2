フラグメントキャッシュ
================

フラグメントキャッシュは、ウェブページの断片をキャッシュすることを言います。例えば、ページ内の表に年間販売の概要が表示されている場合、リクエスト毎にこの表を生成するのにかかる時間を削減するために、キャッシュにこの表を格納することができます。フラグメントキャッシュは [データキャッシュ](caching-data.md) 上に構築されています。

フラグメントキャッシュを使用するには [ビュー](structure-views.md) で以下の構文を使用します:

```php
if ($this->beginCache($id)) {

    // ... ここに生成するコンテンツを書く ...

    $this->endCache();
}
```

つまり [[yii\base\View::beginCache()|beginCache()]] と [[yii\base\View::endCache()|endCache()]] をペアにして囲み、その中にコンテンツ生成ロジックを書いていきます。コンテンツがキャッシュ内で見つかった場合、キャッシュされたコンテンツをレンダリングし [[yii\base\View::beginCache()|beginCache()]] は false を返します。結果として、コンテンツ生成ロジックはスキップされます。それ以外の場合はコンテンツ生成ロジックが呼ばれ、そして [[yii\base\View::endCache()|endCache()]] が呼ばれたとき生成されたコンテンツがキャプチャされ、キャッシュに格納されます。

[データキャッシュ](caching-data.md) と同様に、キャッシュされたコンテンツを識別するためにユニークな `$id` が必要になります。


## キャッシュのオプション <a name="caching-options"></a>

[[yii\base\View::beginCache()|beginCache()]] メソッドの 2 番目のパラメータを配列にすることで、フラグメントキャッシュに関する追加のオプションを指定することもできます。裏で、この配列のオプションは実際にフラグメントキャッシュ機能を実装している [[yii\widgets\FragmentCache]] ウィジェットを構成するために使用されます。

### 持続時間 <a name="duration"></a>


おそらくフラグメントキャッシュで通常よく使われるであろうオプションは [[yii\widgets\FragmentCache::duration|duration]] でしょう。このオプションにはコンテンツがどれだけの時間キャッシュ内において有効であるかを指定します。以下のコードは最大で 1 時間コンテンツの断片をキャッシュします:

```php
if ($this->beginCache($id, ['duration' => 3600])) {

    // ... ここに生成するコンテンツを書く ...

    $this->endCache();
}
```

オプションがセットされていない場合は、デフォルトである 60 が使われ、つまり有効期限が 60 秒間のキャッシュされたコンテンツを意味します。


### 依存関係 <a name="dependencies"></a>

[データキャッシュ](caching-data.md#cache-dependencies) と同様に、キャッシュされたコンテンツの断片は依存関係を持つことができます。例えば、表示されている投稿の内容は、投稿が変更されたか否かに依存する、といった具合です。

依存関係を指定するには [[yii\widgets\FragmentCache::dependency|dependency]] オプションに [[yii\caching\Dependency]] オブジェクトを指定するか、または依存関係オブジェクトを作成するための配列構成を指定します。以下のコードはコンテンツの断片が `updated_at` カラムの値の変化に依存していることを指定しています:

```php
$dependency = [
    'class' => 'yii\caching\DbDependency',
    'sql' => 'SELECT MAX(updated_at) FROM post',
];

if ($this->beginCache($id, ['dependency' => $dependency])) {

    // ... ここに生成するコンテンツを書く ...

    $this->endCache();
}
```


### バリエーション <a name="variations"></a>

キャッシュされたコンテンツはいくつかのパラメータによって変化させることもできます。例えば、複数の言語をサポートしているウェブアプリケーションに対して、ビューコードの同じ部分を、異なる言語で生成することができます。現在のアプリケーションの言語に応じて、キャッシュされたコンテンツに変更を加えるといったことが可能になります。

キャッシュのバリエーションを指定するには [[yii\widgets\FragmentCache::variations|variations]] オプションに配列で、それぞれが特定のバリエーションの要素を表すスカラー値をセットします。例えば、言語によってキャッシュされたコンテンツを変化させるには、以下のコードを使うことができます:

```php
if ($this->beginCache($id, ['variations' => [Yii::$app->language]])) {

    // ... ここに生成するコンテンツを書く ...

    $this->endCache();
}
```


### トグルキャッシュ <a name="toggling-caching"></a>

また、ある条件が満たされた場合にのみフラグメントキャッシュを有効にすることもできます。たとえば、フォームが表示されているページに対して、最初の (GET リクエストによる) リクエストの場合だけはキャッシュしたいと思いますが、その後の (POST リクエストによる) フォームの表示では、フォームにユーザ入力が含まれている可能性があるため、キャッシュをすべきではありません。これを行うには、以下のように [[yii\widgets\FragmentCache::enabled|enabled]] オプションをセットします:

```php
if ($this->beginCache($id, ['enabled' => Yii::$app->request->isGet])) {

    // ... ここに生成するコンテンツを書く ...

    $this->endCache();
}
```


## キャッシュのネスト <a name="nested-caching"></a>

フラグメントキャッシュはネストすることができます。つまり、キャッシュされる断片を、より大きなキャッシュされる断片で囲むことができます。例えば、コメントが内側のフラグメントキャッシュ内にキャッシュされ、それらが外側のフラグメントキャッシュに記事内容と一緒にキャッシュされます。以下のコードは 2 つのフラグメントキャッシュをどのようにネストできるかを示したものです:

```php
if ($this->beginCache($id1)) {

    // ...コンテンツ生成ロジック...

    if ($this->beginCache($id2, $options2)) {

        // ...コンテンツ生成ロジック...

        $this->endCache();
    }

    // ...コンテンツ生成ロジック...

    $this->endCache();
}
```

ネストされたキャッシュには、異なるキャッシュオプションを設定することができます。 たとえば、上記の例における内側のキャッシュと外側のキャッシュに対して、異なる持続期間の値を設定する事が可能です。 これによって、外側のキャッシュでキャッシュされたデータが無効になった場合でも、内側のキャッシュが有効な内側の断片を提供することが可能になります。 しかし、その逆は真ではありません。 外側のキャッシュが有効であると判断された場合には、内側のキャッシュが無効になった後でも、外側のキャッシュが古くなったコンテンツのコピーを提供し続けます。 ネストされたキャッシュの持続時間や依存関係の設定を間違うと、無効になった内側のキャッシュデータが外側のキャッシュに残り続けることになるので、注意が必要です。


## ダイナミックコンテンツ <a name="dynamic-content"></a>

フラグメントキャッシュを使用する際、出力全体が比較的静的で、一ヶ所ないし数ヶ所だけが例外的に動的であるというような状況に遭遇します。例えば、ページ上部にはメインメニューバーと現在のユーザの名前とが一緒に表示される場合があります。他には、リクエスト毎に実行しなければいけない PHP のコードが含まれている場合(例えば、アセットバンドルを登録するためのコード)などです。この両方の問題は、いわゆる *ダイナミックコンテンツ* 機能によって解決することができます。

ダイナミックコンテンツは、それがフラグメントキャッシュの中に含まれていても、キャッシュすべきではない出力の部分を意味します。コンテンツを常に動的にするためには、外側のコンテンツがキャッシュから提供されている場合でも、すべてのリクエストに対して、いくつかのPHP コードを実行することにより生成しなければいけません。

以下のように、ダイナミックコンテンツを目的の場所に挿入するには、キャッシュされた断片内で [[yii\base\View::renderDynamic()]] を呼び出します。

```php
if ($this->beginCache($id1)) {

    // ...コンテンツ生成ロジック...

    echo $this->renderDynamic('return Yii::$app->user->identity->name;');

    // ...コンテンツ生成ロジック...

    $this->endCache();
}
```

[[yii\base\View::renderDynamic()|renderDynamic()]] メソッドはパラメータとして PHP コードの一部を使用します。PHP コードの戻り値は、ダイナミックコンテンツとして扱われます。同じ PHP コードはすべてのリクエストに対して実行されますが、囲まれている断片がキャッシュから提供されているか否かは問いません。