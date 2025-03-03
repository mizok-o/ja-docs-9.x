# Eloquent：ミューテタ／キャスト

- [イントロダクション](#introduction)
- [アクセサ／ミューテタ](#accessors-and-mutators)
    - [アクセサの定義](#defining-an-accessor)
    - [ミューテタの定義](#defining-a-mutator)
- [属性のキャスト](#attribute-casting)
    - [配列とJSONのキャスト](#array-and-json-casting)
    - [日付のキャスト](#date-casting)
    - [Enumキャスト](#enum-casting)
    - [暗号化キャスト](#encrypted-casting)
    - [クエリ時のキャスト](#query-time-casting)
- [カスタムキャスト](#custom-casts)
    - [値オブジェクトのキャスト](#value-object-casting)
    - [配列／JSONのシリアル化](#array-json-serialization)
    - [インバウンドのキャスト](#inbound-casting)
    - [キャストのパラメータ](#cast-parameters)
    - [Castables](#castables)

<a name="introduction"></a>
## イントロダクション

アクセサ、ミューテタ、および属性キャストを使用すると、Eloquentモデルインスタンスで属性値を取得または設定するときに、それらの属性値を変換できます。たとえば、[Laravel暗号化](/docs/{{version}}/encoding)を使用して、データベースに保存されている値を暗号化し、Eloquentモデル上でそれにアクセスしたときに属性を自動的に復号できます。他に、Eloquentモデルを介してアクセスするときに、データベースに格納されているJSON文字列を配列に変換することもできます。

<a name="accessors-and-mutators"></a>
## アクセサ／ミューテタ

<a name="defining-an-accessor"></a>
### アクセサの定義

アクセサは、Eloquentの属性値にアクセスが合った時に、その値を変換するものです。アクセサを定義するには、アクセス可能な属性を表すprotectedなメソッドをモデル上に作成します。このメソッド名は、裏に存在するモデル属性やデータベースカラムの「キャメルケース」表現に対応させる必要があります。

この例では、`first_name`属性に対するアクセサを定義します。このアクセサは、`first_name`属性の値を取得しようとしたときに、Eloquentから自動的に呼び出されます。すべての属性アクセサ/ミューテタメソッドは、戻り値のタイプヒントを`Illuminate\Database\Eloquent\Casts\Attribute`で宣言する必要があります。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Casts\Attribute;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * ユーザーの名前の取得
         *
         * @return \Illuminate\Database\Eloquent\Casts\Attribute
         */
        protected function firstName(): Attribute
        {
            return Attribute::make(
                get: fn ($value) => ucfirst($value),
            );
        }
    }

すべてのアクセサメソッドは`Attribute`インスタンスを返します。このインスタンスは、属性にアクセスする方法と、オプションとして変異させる方法を定義します。この例では、属性にアクセスする方法のみを定義しています。そのために、`ttribute`クラスのコンストラクタに`get`引数を与えます。

ご覧のとおり、カラムの元の値がアクセサに渡され、値を操作でき、結果値を返します。アクセサの値へアクセスするには、モデルインスタンスの`first_name`属性にアクセスするだけです。

    use App\Models\User;

    $user = User::find(1);

    $firstName = $user->first_name;

> **Note**
> こうした計算値をモデルの配列／JSON表現に追加したい場合は、[手作業で追加する必要があります](/docs/{{version}}/eloquent-serialization#appending-values-to-json)。

<a name="building-value-objects-from-multiple-attributes"></a>
#### 複数の属性からの値オブジェクト構築

複数のモデル属性を一つの「値オブジェクト」へアクセサで、変換する必要がある場合も起きるでしょう。そのため、`get`クロージャの第２引数は`$attributes`であり、自動的にこのクロージャに用意され、モデルの現在の属性をすべて配列で持っています。

```php
use App\Support\Address;
use Illuminate\Database\Eloquent\Casts\Attribute;

/**
 * ユーザーの住所を操作
 *
 * @return  \Illuminate\Database\Eloquent\Casts\Attribute
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn ($value, $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
    );
}
```

<a name="accessor-caching"></a>
#### アクセサのキャッシュ

アクセサから値オブジェクトを返すとき、値オブジェクトに加えられたすべての変更は、モデルが保存される前に自動的にモデルに同期して戻されます。これはEloquentがアクセサから返したインスタンスを保持し、アクセサが呼び出されるたびに同じインスタンスを返すことができるためです。

    use App\Models\User;

    $user = User::find(1);

    $user->address->lineOne = 'Updated Address Line 1 Value';
    $user->address->lineTwo = 'Updated Address Line 2 Value';

    $user->save();

しかし、文字列やブーリアンなどのプリミティブな値については、特に計算量が多い場合、キャッシュを有効にしたい場合が起きます。その場合は、アクセサを定義するときに、`shouldCache`メソッドを呼び出してください。

```php
protected function hash(): Attribute
{
    return Attribute::make(
        get: fn ($value) => bcrypt(gzuncompress($value)),
    )->shouldCache();
}
```

属性のオブジェクトキャッシュ動作を無効にしたい場合は、属性の定義時に`withoutObjectCaching`メソッドを呼び出してください。

```php
/**
 * ユーザーの住所を操作
 *
 * @return  \Illuminate\Database\Eloquent\Casts\Attribute
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn ($value, $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
    )->withoutObjectCaching();
}
```

<a name="defining-a-mutator"></a>
### ミューテタの定義

ミューテタはEloquentの属性値を設定するときに、その値を変換するものです。ミューテタを定義するには、属性を定義するときに `set` という引数を指定します。ここでは、`first_name`属性に対してミューテタを定義してみましょう。このミューテタは、モデルの`first_name`属性の値を設定しようとすると、自動的に呼び出されます。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Casts\Attribute;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * ユーザーの名を操作
         *
         * @return \Illuminate\Database\Eloquent\Casts\Attribute
         */
        protected function firstName(): Attribute
        {
            return Attribute::make(
                get: fn ($value) => ucfirst($value),
                set: fn ($value) => strtolower($value),
            );
        }
    }

ミューテタクロージャは、属性に設定しようとする値を受け取り、その値を操作して、操作した値を返します。このミューテタを使うには、Eloquentモデルに`first_name`属性をセットするだけでよいのです。

    use App\Models\User;

    $user = User::find(1);

    $user->first_name = 'Sally';

この例では、`set`コールバックが`Sally`という値で呼び出されます。ミューテタは`strtolower`関数を名前に適用し、その結果をモデルの内部配列`$attributes`へセットします。

<a name="mutating-multiple-attributes"></a>
#### 複数属性のミュート

時には、ミューテーターは裏のモデルへ複数の属性をセットする必要があるかもしれません。その場合は、`set`クロージャから配列を返します。配列の各キーは、モデルと関連付けられた属性やデータベースカラムに対応している必要があります。

```php
use App\Support\Address;
use Illuminate\Database\Eloquent\Casts\Attribute;

/**
 * ユーザーの住所操作
 *
 * @return  \Illuminate\Database\Eloquent\Casts\Attribute
 */
protected function address(): Attribute
{
    return Attribute::make(
        get: fn ($value, $attributes) => new Address(
            $attributes['address_line_one'],
            $attributes['address_line_two'],
        ),
        set: fn (Address $value) => [
            'address_line_one' => $value->lineOne,
            'address_line_two' => $value->lineTwo,
        ],
    );
}
```

<a name="attribute-casting"></a>
## 属性のキャスト

属性キャストは、モデルで追加のメソッドを定義することなく、アクセサやミューテタと同様の機能を提供します。定義する代わりに、モデルの`$casts`プロパティにより属性を一般的なデータ型に変換する便利な方法を提供します。

`$casts`プロパティは、キーがキャストする属性の名前であり、値がそのカラムをキャストするタイプである配列である必要があります。サポートしているキャストタイプは以下のとおりです。

<div class="content-list" markdown="1">

- `array`
- `AsStringable::class`
- `boolean`
- `collection`
- `date`
- `datetime`
- `immutable_date`
- `immutable_datetime`
- <code>decimal:&lt;precision&gt;</code>
- `double`
- `encrypted`
- `encrypted:array`
- `encrypted:collection`
- `encrypted:object`
- `float`
- `integer`
- `object`
- `real`
- `string`
- `timestamp`

</div>

属性のキャストをデモンストレートするため、データベースに整数(`0`または`1`)として格納している`is_admin`属性をブール値にキャストしてみましょう。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * キャストする必要のある属性
         *
         * @var array
         */
        protected $casts = [
            'is_admin' => 'boolean',
        ];
    }

キャストを定義した後、基になる値が整数としてデータベースに格納されていても、アクセス時`is_admin`属性は常にブール値にキャストされます。

    $user = App\Models\User::find(1);

    if ($user->is_admin) {
        //
    }

実行時に新しく一時的なキャストを追加する必要がある場合は、`mergeCasts`メソッドを使用します。こうしたキャストの定義は、モデルで既に定義しているキャストのいずれかに追加されます。

    $user->mergeCasts([
        'is_admin' => 'integer',
        'options' => 'object',
    ]);

> **Warning**
> `null`である属性はキャストしません。また、リレーションと同じ名前のキャスト(または属性)を定義しないでください。

<a name="stringable-casting"></a>
#### Stringableのキャスト

モデルの属性を[fluentの`Illuminate\Support\Stringable`オブジェクト](/docs/{{version}}/helpers#fluent-strings-method-list)へキャストするには、`Illuminate\Database\Eloquent\Casts\AsStringable`キャストクラスが使用できます。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Casts\AsStringable;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * キャストする必要のある属性
         *
         * @var array
         */
        protected $casts = [
            'directory' => AsStringable::class,
        ];
    }

<a name="array-and-json-casting"></a>
### 配列とJSONのキャスト

`array`キャストは、シリアル化されたJSONとして保存されているカラムを操作するときに特に役立ちます。たとえば、データベースにシリアル化されたJSONを含む`JSON`または`TEXT`フィールドタイプがある場合、その属性へ`array`キャストを追加すると、Eloquentモデル上でアクセス時に、属性がPHP配列へ自動的に逆シリアル化されます。

    <?php

    namespace App\Models;

    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * キャストする必要のある属性
         *
         * @var array
         */
        protected $casts = [
            'options' => 'array',
        ];
    }

このキャストを定義すると、`options`属性にアクセスでき、JSONからPHP配列に自動的に逆シリアル化されます。`options`属性の値を設定すると、指定する配列が自動的にシリアル化されてJSONに戻されて保存されます。

    use App\Models\User;

    $user = User::find(1);

    $options = $user->options;

    $options['key'] = 'value';

    $user->options = $options;

    $user->save();

JSON属性の単一のフィールドをより簡潔な構文で更新するには、`update`メソッドを呼び出すときに`->`演算子を使用します。

    $user = User::find(1);

    $user->update(['options->key' => 'value']);

<a name="array-object-and-collection-casting"></a>
#### 配列オブジェクトとコレクションのキャスト

多くのアプリケーションには標準の`array`キャストで十分ですが、いくつかの欠点を持ちます。`array`キャストはプリミティブ型を返すので、配列のオフセットを直接変更することはできません。たとえば、次のコードはPHPエラーを起こします。

    $user = User::find(1);

    $user->options['key'] = $value;

これを解決するために、Laravelは、JSON属性を[ArrayObject](https://www.php.net/manual/ja/class.arrayobject.php)クラスにキャストする`asArrayObject`キャストを提供します。この機能はLaravelの[カスタムキャスト](#custom-cast)の実装を使用しており、Laravelがインテリジェントにキャッシュし、PHPエラーを引き起こすことなく、個々のオフセットを変更できるように、ミューテートしたオブジェクトを変換することができます。AsArrayObject`のキャストを使用するには、単純に属性に割り当てるだけです。

    use Illuminate\Database\Eloquent\Casts\AsArrayObject;

    /**
     * キャストする属性
     *
     * @var array
     */
    protected $casts = [
        'options' => AsArrayObject::class,
    ];

同様に、LaravelはJSON属性をLaravel[コレクション](/docs/{{version}}/collections)へキャストする`ASCollection`キャストを提供しています

    use Illuminate\Database\Eloquent\Casts\AsCollection;

    /**
     * キャストする属性
     *
     * @var array
     */
    protected $casts = [
        'options' => AsCollection::class,
    ];

<a name="date-casting"></a>
### 日付のキャスト

デフォルトでは、Eloquentは`created_at`カラムと`updated_at`カラムを[Carbon](https://github.com/briannesbitt/Carbon)のインスタンスへキャストします。これによりPHPの`DateTime`クラスを拡張した、多くの便利なメソッドが提供されます。モデルの`$casts`プロパティ配列内で日付キャストを追加定義すれば、他の日付属性をキャストできます。通常、日付は`datetime`か`immutable_datetime`キャストタイプを使用してキャストする必要があります。

`date`または`datetime`キャストを定義するときに、日付の形式を指定することもできます。この形式は、[モデルが配列またはJSONにシリアル化される](/docs/{{version}}/eloquent-serialization)場合に使用されます。

    /**
     * キャストする属性
     *
     * @var array
     */
    protected $casts = [
        'created_at' => 'datetime:Y-m-d',
    ];

カラムが日付へキャストされる場合、対応するモデル属性の値として、UNIXのタイムスタンプ、日付文字列（`Y-m-d`）、日付時間文字列、または`DateTime`／`Carbon`インスタンスを設定することができます。日付の値は正しく変換され、データベースへ保存されます。

モデルに`serializeDate`メソッドを定義することで、モデルのすべての日付のデフォルトのシリアル化形式をカスタマイズできます。この方法は、データベースへ保存するために日付をフォーマットする方法には影響しません。

    /**
     * 配列/JSONシリアル化の日付を準備
     *
     * @param  \DateTimeInterface  $date
     * @return string
     */
    protected function serializeDate(DateTimeInterface $date)
    {
        return $date->format('Y-m-d');
    }

データベース内にモデルの日付を実際に保存するときに使用する形式を指定するには、モデルに`$dateFormat`プロパティを定義する必要があります。

    /**
     * モデルの日付カラムのストレージ形式
     *
     * @var string
     */
    protected $dateFormat = 'U';

<a name="date-casting-and-timezones"></a>
#### 日付のキャストとシリアライズ、タイムゾーン

`date`と`datetime`のキャストはデフォルトで、アプリケーションの`timezone`設定オプションで指定されているタイムゾーンに関わらず、日付をUTC ISO-8601の日付文字列（`1986-05-28T21:05:54.000000Z`）にシリアライズします。アプリケーションの`timezone`設定オプションをデフォルトの`UTC`から変更せずに、常にこのシリアライズ形式を使用し、アプリケーションの日付をUTCタイムゾーンで保存することを強く推奨します。アプリケーション全体でUTCタイムゾーンを一貫して使用することで、PHPやJavaScriptで書かれた他の日付操作ライブラリとの相互運用性を最大限に高められます。

`datetime:Y-m-d H:i:s`のようなカスタムフォーマットを`date`や`datetime`キャストで適用する場合は、日付のシリアライズの際に、Carbonインスタンスの内部タイムゾーンが使用されます。一般的には、アプリケーションの`timezone`設定オプションで指定したタイムゾーンを使用します。

<a name="enum-casting"></a>
### Enumキャスト

> **Warning**
> Enumキャストは、PHP8.1以上で使用できます。

Eloquentは、属性値をPHPの[Enum](https://www.php.net/manual/ja/language.enumerations.backed.php) にキャストすることも可能です。これを実現するには、モデルの`$casts`プロパティ配列にキャストしたい属性と列挙型を指定します。

    use App\Enums\ServerStatus;

    /**
     * キャストする属性
     *
     * @var array
     */
    protected $casts = [
        'status' => ServerStatus::class,
    ];

モデルにキャストを定義すると、指定した属性を操作する際に、自動的にenumへキャストしたり、enumからキャストされたりするようになります。

    if ($server->status == ServerStatus::provisioned) {
        $server->status = ServerStatus::ready;

        $server->save();
    }

<a name="encrypted-casting"></a>
### 暗号化キャスト

`encrypted`キャストは、Laravelに組み込まれた[暗号化](/docs/{{version}}/encryption)機能を使って、モデルの属性値を暗号化します。さらに、`encrypted:array`、`encrypted:collection`、`encrypted:object`、`AsEncryptedArrayObject`、`AsEncryptedCollection`のキャストは、暗号化されていないものと同様の動作をしますが、ご期待通りにデータベースに保存される際に、基本的な値を暗号化します。

暗号化したテキストの最終的な長さは予測できず、プレーンテキストのものよりも長くなるので、関連するデータベースのカラムが `TEXT` 型以上であることを確認してください。さらに、値はデータベース内で暗号化されているので、暗号化された属性値を問い合わせたり検索したりすることはできません。

<a name="key-rotation"></a>
#### キーの変更

ご存知のように、Laravelはアプリケーションの`app`設定ファイルで指定した`key`設定値を使い、文字列を暗号化します。通常、この値は環境変数`APP_KEY`の値です。もし、アプリケーションの暗号化キーを変更する必要がある場合は、新しいキーを使い、暗号化した属性を手作業で再暗号化する必要があります。

<a name="query-time-casting"></a>
### クエリ時のキャスト

テーブルから元の値でセレクトするときなど、クエリの実行中にキャストを適用する必要が起きる場合があります。たとえば、次のクエリを考えてみましょう。

    use App\Models\Post;
    use App\Models\User;

    $users = User::select([
        'users.*',
        'last_posted_at' => Post::selectRaw('MAX(created_at)')
                ->whereColumn('user_id', 'users.id')
    ])->get();

このクエリ結果の`last_posted_at`属性は単純な文字列になります。クエリを実行するときに、この属性に「datetime」キャストを適用できれば素晴らしいと思うでしょう。幸運なことに、`withCasts`メソッドを使用してこれができます。

    $users = User::select([
        'users.*',
        'last_posted_at' => Post::selectRaw('MAX(created_at)')
                ->whereColumn('user_id', 'users.id')
    ])->withCasts([
        'last_posted_at' => 'datetime'
    ])->get();

<a name="custom-casts"></a>
## カスタムキャスト

Laravelには、さまざまな組み込みの便利なキャストタイプがあります。それでも、独自のキャストタイプを定義する必要が起きる場合があります。これは、`CastsAttributes`インターフェイスを実装するクラスを定義することで実現できます。

このインターフェイスを実装するクラスは、`get`および`set`メソッドを定義する必要があります。`get`メソッドはデータベースからの素の値をキャスト値に変換する役割を果たしますが、`set`メソッドはキャスト値をデータベースに保存できる素の値に変換する必要があります。例として、組み込みの`json`キャストタイプをカスタムキャストタイプとして再実装します。

    <?php

    namespace App\Casts;

    use Illuminate\Contracts\Database\Eloquent\CastsAttributes;

    class Json implements CastsAttributes
    {
        /**
         * 指定値をキャスト
         *
         * @param  \Illuminate\Database\Eloquent\Model  $model
         * @param  string  $key
         * @param  mixed  $value
         * @param  array  $attributes
         * @return array
         */
        public function get($model, $key, $value, $attributes)
        {
            return json_decode($value, true);
        }

        /**
         * 指定値をストレージ用に準備
         *
         * @param  \Illuminate\Database\Eloquent\Model  $model
         * @param  string  $key
         * @param  array  $value
         * @param  array  $attributes
         * @return string
         */
        public function set($model, $key, $value, $attributes)
        {
            return json_encode($value);
        }
    }

カスタムキャストタイプを定義したら、そのクラス名をモデル属性へ指定できます。

    <?php

    namespace App\Models;

    use App\Casts\Json;
    use Illuminate\Database\Eloquent\Model;

    class User extends Model
    {
        /**
         * キャストする必要のある属性
         *
         * @var array
         */
        protected $casts = [
            'options' => Json::class,
        ];
    }

<a name="value-object-casting"></a>
### 値オブジェクトのキャスト

値をプリミティブ型にキャストすることに限定されません。オブジェクトへ値をキャストすることもできます。オブジェクトへ値をキャストするカスタムキャストの定義は、プリミティブ型へのキャストと非常によく似ています。ただし、`set`メソッドは、モデルに素の保存可能な値を設定するために使用するキー／値のペアの配列を返す必要があります。

例として、複数のモデル値を単一の`Address`値オブジェクトにキャストするカスタムキャストクラスを定義します。`Address`値には、`lineOne`と`lineTwo`の２つのパブリックプロパティがあると想定します。

    <?php

    namespace App\Casts;

    use App\ValueObjects\Address as AddressValueObject;
    use Illuminate\Contracts\Database\Eloquent\CastsAttributes;
    use InvalidArgumentException;

    class Address implements CastsAttributes
    {
        /**
         * 指定値をキャスト
         *
         * @param  \Illuminate\Database\Eloquent\Model  $model
         * @param  string  $key
         * @param  mixed  $value
         * @param  array  $attributes
         * @return \App\ValueObjects\Address
         */
        public function get($model, $key, $value, $attributes)
        {
            return new AddressValueObject(
                $attributes['address_line_one'],
                $attributes['address_line_two']
            );
        }

        /**
         * 指定値をストレージ用に準備
         *
         * @param  \Illuminate\Database\Eloquent\Model  $model
         * @param  string  $key
         * @param  \App\ValueObjects\Address  $value
         * @param  array  $attributes
         * @return array
         */
        public function set($model, $key, $value, $attributes)
        {
            if (! $value instanceof AddressValueObject) {
                throw new InvalidArgumentException('The given value is not an Address instance.');
            }

            return [
                'address_line_one' => $value->lineOne,
                'address_line_two' => $value->lineTwo,
            ];
        }
    }

値オブジェクトにキャストする場合、値オブジェクトに加えられた変更は、モデルが保存される前に自動的にモデルに同期されます。

    use App\Models\User;

    $user = User::find(1);

    $user->address->lineOne = 'Updated Address Value';

    $user->save();

> **Note**
> 値オブジェクトを含むEloquentモデルをJSONまたは配列にシリアル化する場合は、値オブジェクトに`Illuminate\Contracts\Support\Arrayable`および`JsonSerializable`インターフェイスを実装する必要があります。

<a name="array-json-serialization"></a>
### 配列／JSONのシリアル化

Eloquentモデルを`toArray`および`toJson`メソッドを使用して配列やJSONへ変換する場合、カスタムキャスト値オブジェクトは通常、`Illuminate\Contracts\Support\Arrayable`および`JsonSerializable`インターフェイスを実装している限りシリアル化されます。しかし、サードパーティライブラリによって提供される値オブジェクトを使用する場合、これらのインターフェイスをオブジェクトに追加できない場合があります。

したがって、カスタムキャストクラスが値オブジェクトのシリアル化を担当するように指定できます。そのためには、カスタムクラスキャストで`Illuminate\Contracts\Database\Eloquent\SerializesCastableAttributes`インターフェイスを実装する必要があります。このインターフェイスは、クラスに「serialize」メソッドが含まれている必要があることを示しています。このメソッドは、値オブジェクトのシリアル化された形式を返す必要があります。

    /**
     * 値をシリアル化した表現の取得
     *
     * @param  \Illuminate\Database\Eloquent\Model  $model
     * @param  string  $key
     * @param  mixed  $value
     * @param  array  $attributes
     * @return mixed
     */
    public function serialize($model, string $key, $value, array $attributes)
    {
        return (string) $value;
    }

<a name="inbound-casting"></a>
### インバウンドのキャスト

場合によっては、モデルに値を設定するときのみ変換し、モデルから属性を取得するときは操作をしないカスタムキャストを作成する必要があります。インバウンドのみのキャストの典型的な例は、「ハッシュ（hashing）」キャストです。インバウンドのみのカスタムキャストは、`CastsInboundAttributes`インターフェイスを実装する必要があります。これには`set`メソッドの定義のみが必要です。

    <?php

    namespace App\Casts;

    use Illuminate\Contracts\Database\Eloquent\CastsInboundAttributes;

    class Hash implements CastsInboundAttributes
    {
        /**
         * ハッシュアルゴリズム
         *
         * @var string
         */
        protected $algorithm;

        /**
         * 新しいキャストクラスインスタンスの生成
         *
         * @param  string|null  $algorithm
         * @return void
         */
        public function __construct($algorithm = null)
        {
            $this->algorithm = $algorithm;
        }

        /**
         * 指定値をストレージ用に準備
         *
         * @param  \Illuminate\Database\Eloquent\Model  $model
         * @param  string  $key
         * @param  array  $value
         * @param  array  $attributes
         * @return string
         */
        public function set($model, $key, $value, $attributes)
        {
            return is_null($this->algorithm)
                        ? bcrypt($value)
                        : hash($this->algorithm, $value);
        }
    }

<a name="cast-parameters"></a>
### キャストのパラメータ

カスタムキャストをモデルへ指定する場合、`:`文字を使用してクラス名から分離し、複数のパラメータをコンマで区切ることでキャストパラメータを指定できます。パラメータは、キャストクラスのコンストラクタへ渡されます。

    /**
     * キャストする属性
     *
     * @var array
     */
    protected $casts = [
        'secret' => Hash::class.':sha256',
    ];

<a name="castables"></a>
### Castables

アプリケーションの値オブジェクトが独自のカスタムキャストクラスを定義できるようにすることができます。カスタムキャストクラスをモデルにアタッチする代わりに、`Illuminate\Contracts\Database\Eloquent\Castable`インターフェイスを実装する値オブジェクトクラスをアタッチすることもできます。

    use App\Models\Address;

    protected $casts = [
        'address' => Address::class,
    ];

`Castable`インターフェイスを実装するオブジェクトは、`Castable`クラスにキャストする／される責務を受け持つ、カスタムキャスタークラスのクラス名を返す`castUsing`メソッドを定義する必要があります。

    <?php

    namespace App\Models;

    use Illuminate\Contracts\Database\Eloquent\Castable;
    use App\Casts\Address as AddressCast;

    class Address implements Castable
    {
        /**
         * このキャストターゲットにキャストする／されるときに使用するキャスタークラスの名前を取得
         *
         * @param  array  $arguments
         * @return string
         */
        public static function castUsing(array $arguments)
        {
            return AddressCast::class;
        }
    }

`Castable`クラスを使用する場合でも、`$casts`定義に引数を指定できます。引数は`castUsing`メソッドに渡されます。

    use App\Models\Address;

    protected $casts = [
        'address' => Address::class.':argument',
    ];

<a name="anonymous-cast-classes"></a>
#### Castableと匿名キャストクラス

"Castable"をPHPの[匿名クラス](https://www.php.net/manual/ja/language.oop5.anonymous.php)と組み合わせることで、値オブジェクトとそのキャストロジックを単一のCastableオブジェクトとして定義できます。これを実現するには、値オブジェクトの`castUsing`メソッドから匿名クラスを返します。匿名クラスは`CastsAttributes`インターフェイスを実装する必要があります。

    <?php

    namespace App\Models;

    use Illuminate\Contracts\Database\Eloquent\Castable;
    use Illuminate\Contracts\Database\Eloquent\CastsAttributes;

    class Address implements Castable
    {
        // ...

        /**
         * このキャストターゲットにキャストする／されるときに使用するキャスタークラスの名前を取得
         *
         * @param  array  $arguments
         * @return object|string
         */
        public static function castUsing(array $arguments)
        {
            return new class implements CastsAttributes
            {
                public function get($model, $key, $value, $attributes)
                {
                    return new Address(
                        $attributes['address_line_one'],
                        $attributes['address_line_two']
                    );
                }

                public function set($model, $key, $value, $attributes)
                {
                    return [
                        'address_line_one' => $value->lineOne,
                        'address_line_two' => $value->lineTwo,
                    ];
                }
            };
        }
    }
