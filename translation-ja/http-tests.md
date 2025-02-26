# HTTPテスト

- [イントロダクション](#introduction)
- [リクエストの作成](#making-requests)
    - [リクエストヘッダのカスタマイズ](#customizing-request-headers)
    - [クッキー](#cookies)
    - [セッション／認証](#session-and-authentication)
    - [レスポンスのデバッグ](#debugging-responses)
    - [例外処理](#exception-handling)
- [JSON APIのテスト](#testing-json-apis)
    - [FluentなJSONテスト](#fluent-json-testing)
- [ファイルアップロードのテスト](#testing-file-uploads)
- [ビューのテスト](#testing-views)
    - [Bladeとコンポーネントのレンダ](#rendering-blade-and-components)
- [利用可能なアサート](#available-assertions)
    - [レスポンスのアサート](#response-assertions)
    - [認証のアサート](#authentication-assertions)

<a name="introduction"></a>
## イントロダクション

Laravelは、アプリケーションにHTTPリクエストを送信し、レスポンスを調べるための非常に流暢なAPIを提供しています。例として、以下に定義している機能テストをご覧ください。

    <?php

    namespace Tests\Feature;

    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;
    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * 基本的なテスト例
         *
         * @return void
         */
        public function test_a_basic_request()
        {
            $response = $this->get('/');

            $response->assertStatus(200);
        }
    }

`get`メソッドはアプリケーションに`GET`リクエストを送信し、`assertStatus`メソッドは返えされたレスポンスに指定するHTTPステータスコードが必要であることを宣言しています。この単純なアサートに加え、Laravelはレスポンスヘッダ、コンテンツ、JSON構造などを検査するためにさまざまなアサートも用意しています。

<a name="making-requests"></a>
## リクエストの作成

アプリケーションにリクエストを送信するには、テスト内で`get`、`post`、`put`、`patch`、`delete`メソッドを呼び出してください。これらのメソッドは、実際にはアプリケーションへ「本当の」HTTPリクエストを発行しません。代わりに、ネットワークリクエスト全体が内部でシミュレートされます。

テストリクエストメソッドは、`Illuminate\Http\Response`インスタンスを返す代わりに、`Illuminate\Testing\TestResponse`のインスタンスを返します。これは、アプリケーションのレスポンスの検査を可能にする[さまざまな有用なアサート](#available-assertions)を提供します。

    <?php

    namespace Tests\Feature;

    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;
    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * 基本的なテスト例
         *
         * @return void
         */
        public function test_a_basic_request()
        {
            $response = $this->get('/');

            $response->assertStatus(200);
        }
    }

通常、各テストはアプリケーションに対して 1 つのリクエストしか行わないようにしてください。１つのテストメソッドの中で複数のリクエストを実行すると、 予期せぬ動作が起きる可能性があります。

> **Note**
> 利便性を良くするため、テストの実行時にCSRFミドルウェアを自動で無効にします。

<a name="customizing-request-headers"></a>
### リクエストヘッダのカスタマイズ

`withHeaders`メソッドを使用して、アプリケーションに送信する前にリクエストのヘッダをカスタマイズできます。このメソッドを使用すると、リクエストに必要なカスタムヘッダを追加できます。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * 基本的な機能テストの例
         *
         * @return void
         */
        public function test_interacting_with_headers()
        {
            $response = $this->withHeaders([
                'X-Header' => 'Value',
            ])->post('/user', ['name' => 'Sally']);

            $response->assertStatus(201);
        }
    }

<a name="cookies"></a>
### クッキー

リクエストを行う前に、`withCookie`または`withCookies`メソッドを使用してクッキー値を設定できます。`withCookie`メソッドは２つの引数としてCookieの名前と値を引数に取りますが、`withCookies`メソッドは名前／値ペアの配列を引数に取ります。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        public function test_interacting_with_cookies()
        {
            $response = $this->withCookie('color', 'blue')->get('/');

            $response = $this->withCookies([
                'color' => 'blue',
                'name' => 'Taylor',
            ])->get('/');
        }
    }

<a name="session-and-authentication"></a>
### セッション／認証

Laravelは、HTTPテスト中にセッションを操作ためヘルパをいくつか提供しています。まず、`withSession`メソッドへ配列を渡し、セッションデータを設定できます。これは、アプリケーションにリクエストを発行する前に、セッションにデータをロードする場合に役立ちます。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        public function test_interacting_with_the_session()
        {
            $response = $this->withSession(['banned' => false])->get('/');
        }
    }

Laravelのセッションは通常、現在認証しているユーザーの状態を維持するために使用します。したがって、`actingAs`ヘルパメソッドは、指定ユーザーを現在のユーザーとして認証する簡単な方法を提供します。たとえば、[モデルファクトリ](/docs/{{version}}/eloquent-factories)を使用して、ユーザーを生成および認証できます。

    <?php

    namespace Tests\Feature;

    use App\Models\User;
    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        public function test_an_action_that_requires_authentication()
        {
            $user = User::factory()->create();

            $response = $this->actingAs($user)
                             ->withSession(['banned' => false])
                             ->get('/');
        }
    }

`actingAs`メソッドの２番目の引数としてガード名を渡すことにより、特定のユーザーを認証するために使用するガードを指定することもできます。`actingAs`メソッドに指定したガードは、テストの間、デフォルトのガードにもなります。

    $this->actingAs($user, 'web')

<a name="debugging-responses"></a>
### レスポンスのデバッグ

アプリケーションにテストリクエストを送ったあとは、`dump`、`dumpHeaders`、`dumpSession`メソッドを使用して、レスポンスの内容を調べてデバッグできます。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * 基本的なテスト例
         *
         * @return void
         */
        public function test_basic_test()
        {
            $response = $this->get('/');

            $response->dumpHeaders();

            $response->dumpSession();

            $response->dump();
        }
    }

あるいは、`dd`、`ddHeaders`、`ddSession`メソッドを使って、レスポンスに関する情報をダンプしてから、実行を停止することもできます。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * 基本的なテスト例
         *
         * @return void
         */
        public function test_basic_test()
        {
            $response = $this->get('/');

            $response->ddHeaders();

            $response->ddSession();

            $response->dd();
        }
    }

<a name="exception-handling"></a>
### 例外処理

時には、アプリケーションが特定の例外を投げているかをテストしたい場合も起きます。例外がLaravelの例外ハンドラに捕捉され、HTTPレスポンスとして返されないようにするために、リクエストを行う前に、`withoutExceptionHandling`メソッドを呼び出してください。

    $response = $this->withoutExceptionHandling()->get('/');

加えて、PHP 言語やアプリケーションが使用しているライブラリで廃止された機能をアプリケーションが使用していないことを確認したい場合は、リクエストを行う前に`withoutDeprecationHandling`メソッドを呼び出してください。この廃止例外の処理を無効にすると、deprecationの警告が例外に変換され、テストが失敗するようになります。

    $response = $this->withoutDeprecationHandling()->get('/');

<a name="testing-json-apis"></a>
## JSON APIのテスト

Laravelは、JSON APIとそのレスポンスをテストするためのヘルパもいくつか提供しています。たとえば、`json`、`getJson`、`postJson`、`putJson`、`patchJson`、`deleteJson`、`optionsJson`メソッドを使用して、さまざまなHTTP動詞でJSONリクエストを発行できます。これらのメソッドにデータとヘッダを簡単に渡すこともできます。まず、`/api/user`に対して`POST`リクエストを作成し、期待されるJSONデータが返されたことを宣言するテストを作成しましょう。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * 基本的な機能テストの例
         *
         * @return void
         */
        public function test_making_an_api_request()
        {
            $response = $this->postJson('/api/user', ['name' => 'Sally']);

            $response
                ->assertStatus(201)
                ->assertJson([
                    'created' => true,
                ]);
        }
    }

さらに、JSONレスポンスデータは、レスポンス上の配列変数としてアクセスできるため、JSONレスポンス内で返された個々の値を調べるのに便利です。

    $this->assertTrue($response['created']);

> **Note**
> `assertJson`メソッドはレスポンスを配列に変換し、`PHPUnit::assertArraySubset`を利用して、指定する配列がアプリケーションによって返されるJSONレスポンス内に存在することを確認しています。したがって、JSONレスポンスに他のプロパティがある場合でも、指定するフラグメントが存在する限り、このテストは合格します。

<a name="verifying-exact-match"></a>
#### 厳密なJSON一致のアサート

前述のように、`assertJson`メソッドを使用して、JSONのフラグメントがJSONレスポンス内に存在することをアサートできます。指定した配列とアプリケーションが返してきたJSONが**完全に一致**することを確認したい場合は、`assertExactJson`メソッドを使用する必要があります。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * 基本的な機能テストの例
         *
         * @return void
         */
        public function test_asserting_an_exact_json_match()
        {
            $response = $this->postJson('/user', ['name' => 'Sally']);

            $response
                ->assertStatus(201)
                ->assertExactJson([
                    'created' => true,
                ]);
        }
    }

<a name="verifying-json-paths"></a>
#### JSONパスでのアサート

JSONレスポンスの指定パスに指定データが含まれていることを確認する場合は、`assertJsonPath`メソッドを使用する必要があります。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        /**
         * 基本的な機能テストの例
         *
         * @return void
         */
        public function test_asserting_a_json_paths_value()
        {
            $response = $this->postJson('/user', ['name' => 'Sally']);

            $response
                ->assertStatus(201)
                ->assertJsonPath('team.owner.name', 'Darian');
        }
    }

`assertJsonPath`メソッドはクロージャも引数に取れます。クロージャはアサートをパスするか動的に判断するため使用されます。

    $response->assertJsonPath('team.owner.name', fn ($name) => strlen($name) >= 3);

<a name="fluent-json-testing"></a>
### FluentなJSONテスト

また、LaravelはアプリケーションのJSONレスポンスを流暢にテストする美しい方法を提供しています。まず始めに、`assertJson`メソッドにクロージャを渡します。このクロージャは、アプリケーションが返すJSONに対してアサートを行うために使用できる、`Illuminate\Testing\Fluent\AssertableJson`のインスタンスで呼び出されます。`where`メソッドはJSONの特定の属性に対してアサートを行うために使用でき、`missing`メソッドはJSONに特定の属性がないことをアサートするために使用できます。

    use Illuminate\Testing\Fluent\AssertableJson;

    /**
     * 基本的な機能テストの例
     *
     * @return void
     */
    public function test_fluent_json()
    {
        $response = $this->getJson('/users/1');

        $response
            ->assertJson(fn (AssertableJson $json) =>
                $json->where('id', 1)
                     ->where('name', 'Victoria Faith')
                     ->whereNot('status', 'pending')
                     ->missing('password')
                     ->etc()
            );
    }

#### `etc`メソッドを理解する

上記の例では、アサートのチェーンの最後で`etc`メソッドを呼び出したことに気づかれた方もいらっしゃるでしょう。このメソッドはLaravelへJSONオブジェクト中に他の属性が存在する可能性があることを伝えます。`etc`メソッドが使用されていない場合は、JSONオブジェクトに他の属性が存在していることをアサートしていないため、テストは失敗します。

この動作の意図は、属性に対して明示的にアサーションを行うか、`etc` メソッドで追加の属性を明示的に許可することで、JSONレスポンスで意図せず機密情報を公開してしまうことを防ぐことにあります。

<a name="asserting-json-attribute-presence-and-absence"></a>
#### 属性の有無をアサートする

属性が存在しているかどうかをアサートするには、`has`と`missing`メソッドを使います。

    $response->assertJson(fn (AssertableJson $json) =>
        $json->has('data')
             ->missing('message')
    );

さらに、`hasAll`と`missingAll`メソッドは、複数の属性の有無を同時にアサートできます。

    $response->assertJson(fn (AssertableJson $json) =>
        $json->hasAll(['status', 'data'])
             ->missingAll(['message', 'code'])
    );

`hasAny`メソッドを使用して、指定する属性リスト中に少なくとも１つ存在しているか判断できます。

    $response->assertJson(fn (AssertableJson $json) =>
        $json->has('status')
             ->hasAny('data', 'message', 'code')
    );

<a name="asserting-against-json-collections"></a>
#### JSONコレクションに対するアサート

多くの場合、ルートは複数のユーザーなど、複数の項目を含むJSONレスポンスを返します。

    Route::get('/users', function () {
        return User::all();
    });

このような状況では、レスポンスに含まれているユーザーに対するアサートを行うために、fluentなJSONオブジェクトの`has`メソッドを使用することができます。例として、JSONレスポンスに３つのユーザーが含まれていることをアサートしましょう。次に、`first`メソッドを使用して、コレクション内の最初のユーザーに関するいくつかのアサートを行います。`first`メソッドは、JSONコレクション内の最初のオブジェクトに関するアサートを行うために使用できる別のアサートJSON文字列を受信するクロージャを引数に取ります。

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has(3)
                 ->first(fn ($json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->missing('password')
                         ->etc()
                 )
        );

<a name="scoping-json-collection-assertions"></a>
#### JSONコレクションをスコープするアサーション

時々、アプリケーションのルートは名前付きキーが割り当てられているJSONコレクションを返します。

    Route::get('/users', function () {
        return [
            'meta' => [...],
            'users' => User::all(),
        ];
    })

これらのルートをテストするときは、コレクション内の項目数に対して`has`メソッドを使えます。さらに、`has`メソッドを使用してアサーションチェーンをスコープできます。

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has('meta')
                 ->has('users', 3)
                 ->has('users.0', fn ($json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->missing('password')
                         ->etc()
                 )
        );

ただし、`has`コレクションに`has`コレクションにアサートする２つの別々の呼び出しをするのではなく、クロージャを３番目の引数に渡す呼び出し一つにまとめられます。これで、クロージャが自動的に呼び出され、コレクション内の最初の項目にスコープされます。

    $response
        ->assertJson(fn (AssertableJson $json) =>
            $json->has('meta')
                 ->has('users', 3, fn ($json) =>
                    $json->where('id', 1)
                         ->where('name', 'Victoria Faith')
                         ->missing('password')
                         ->etc()
                 )
        );

<a name="asserting-json-types"></a>
#### JSONタイプのアサート

JSONレスポンス内のプロパティが特定の型のものであることをアサートできます。`Illuminate\Testing\Fluent\AssertableJson`クラスは、これを行う`whereType`と`whereAllType`メソッドを提供しています。

    $response->assertJson(fn (AssertableJson $json) =>
        $json->whereType('id', 'integer')
             ->whereAllType([
                'users.0.name' => 'string',
                'meta' => 'array'
            ])
    );

複数のタイプを指定するには `|` 文字を使用するか、タイプの配列を `whereType` メソッドの 2 番目のパラメータとして渡します。レスポンスの値がリストアップされたタイプのいずれかであれば、アサーションは成功します。

    $response->assertJson(fn (AssertableJson $json) =>
        $json->whereType('name', 'string|null')
             ->whereType('id', ['string', 'integer'])
    );

`whereType`と`whereAllType`メソッドは、`string`、`integer`、`double`、`boolean`、`array`、`null`タイプを認識します。

<a name="testing-file-uploads"></a>
## ファイルアップロードのテスト

`Illuminate\Http\UploadedFile`クラスは、テスト用のダミーファイルまたは画像を生成するために使用できる`fake`メソッドを提供しています。これを`Storage`ファサードの`fake`メソッドと組み合わせると、ファイルアップロードのテストが大幅に簡素化されます。たとえば、次の２つの機能を組み合わせて、アバターのアップロードフォームを簡単にテストできます。

    <?php

    namespace Tests\Feature;

    use Illuminate\Foundation\Testing\RefreshDatabase;
    use Illuminate\Foundation\Testing\WithoutMiddleware;
    use Illuminate\Http\UploadedFile;
    use Illuminate\Support\Facades\Storage;
    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        public function test_avatars_can_be_uploaded()
        {
            Storage::fake('avatars');

            $file = UploadedFile::fake()->image('avatar.jpg');

            $response = $this->post('/avatar', [
                'avatar' => $file,
            ]);

            Storage::disk('avatars')->assertExists($file->hashName());
        }
    }

指定ファイルが存在しないことを宣言したい場合は、`Storage`ファサードが提供する`assertMissing`メソッドを使用できます。

    Storage::fake('avatars');

    // ...

    Storage::disk('avatars')->assertMissing('missing.jpg');

<a name="fake-file-customization"></a>
#### fakeファイルのカスタマイズ

`UploadedFile`クラスが提供する`fake`メソッドを使用してファイルを作成する場合、アプリケーションのバリデーションルールをより適切にテストするために、画像の幅、高さ、およびサイズ(キロバイト単位)を指定できます。

    UploadedFile::fake()->image('avatar.jpg', $width, $height)->size(100);

画像に加えて、`create`メソッドを使用して他のタイプのファイルも作成できます。

    UploadedFile::fake()->create('document.pdf', $sizeInKilobytes);

必要に応じて、メソッドに`$mimeType`引数を渡し、ファイルが返すMIMEタイプを明示的に定義できます。

    UploadedFile::fake()->create(
        'document.pdf', $sizeInKilobytes, 'application/pdf'
    );

<a name="testing-views"></a>
## ビューのテスト

Laravelを使用すると、アプリケーションに対してシミュレートするHTTPリクエストを作成せずにビューをレンダすることもできます。それには、テスト内で`view`メソッドを呼び出してください。`view`メソッドは、ビュー名とオプションのデータ配列を引数に取ります。このメソッドは`Illuminate\Testing\TestView`のインスタンスを返します。これは、ビューのコンテンツに関するアサートを簡単に作成するためのメソッドをいくつか提供しています。

    <?php

    namespace Tests\Feature;

    use Tests\TestCase;

    class ExampleTest extends TestCase
    {
        public function test_a_welcome_view_can_be_rendered()
        {
            $view = $this->view('welcome', ['name' => 'Taylor']);

            $view->assertSee('Taylor');
        }
    }

`TestView`クラスは、次のアサートメソッドを提供します。`assertSee`、`assertSeeInOrder`、`assertSeeText`、`assertSeeTextInOrder`、`assertDontSee`、`assertDontSeeText`

必要に応じて、`TestView`インスタンスを文字列にキャストすることで、レンダリングされた素のビューコンテンツを取得できます。

    $contents = (string) $this->view('welcome');

<a name="sharing-errors"></a>
#### エラーの共有

一部のビューは、[Laravelが提供するグローバルエラーバッグ](/docs/{{version}}/validation#quick-displaying-the-validation-errors)で共有されるエラーに依存する場合があります。エラーバッグをエラーメッセージでハイドレイトするには、`withViewErrors`メソッドを使用できます。

    $view = $this->withViewErrors([
        'name' => ['Please provide a valid name.']
    ])->view('form');

    $view->assertSee('Please provide a valid name.');

<a name="rendering-blade-and-components"></a>
### Bladeとコンポーネントのレンダ

必要に応じて、`blade`メソッドを使用して、素の[Blade](/docs/{{version}}/blade)文字列を評価およびレンダできます。`view`メソッドと同様に、`blade`メソッドは`Illuminate\Testing\TestView`のインスタンスを返します。

    $view = $this->blade(
        '<x-component :name="$name" />',
        ['name' => 'Taylor']
    );

    $view->assertSee('Taylor');

`component`メソッドを使用して、[ブレードコンポーネント](/docs/{{version}}/Blade#components)を評価およびレンダできます。`view`メソッドと同様に、`component`メソッドは`Illuminate\Testing\TestView`のインスタンスを返します。

    $view = $this->component(Profile::class, ['name' => 'Taylor']);

    $view->assertSee('Taylor');

<a name="available-assertions"></a>
## 利用可能なアサート

<a name="response-assertions"></a>
### レスポンスのアサート

Laravelの`Illuminate\Testing\TestResponse`クラスは、アプリケーションをテストするときに利用できるさまざまなカスタムアサートメソッドを提供します。これらのアサートは、`json`、`get`、`post`、`put`、`delete`テストメソッドが返すレスポンスからアクセスできます。

<style>
    .collection-method-list > p {
        columns: 14.4em 2; -moz-columns: 14.4em 2; -webkit-columns: 14.4em 2;
    }

    .collection-method-list a {
        display: block;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }
</style>

<div class="collection-method-list" markdown="1">

[assertCookie](#assert-cookie)
[assertCookieExpired](#assert-cookie-expired)
[assertCookieNotExpired](#assert-cookie-not-expired)
[assertCookieMissing](#assert-cookie-missing)
[assertCreated](#assert-created)
[assertDontSee](#assert-dont-see)
[assertDontSeeText](#assert-dont-see-text)
[assertDownload](#assert-download)
[assertExactJson](#assert-exact-json)
[assertForbidden](#assert-forbidden)
[assertHeader](#assert-header)
[assertHeaderMissing](#assert-header-missing)
[assertJson](#assert-json)
[assertJsonCount](#assert-json-count)
[assertJsonFragment](#assert-json-fragment)
[assertJsonMissing](#assert-json-missing)
[assertJsonMissingExact](#assert-json-missing-exact)
[assertJsonMissingValidationErrors](#assert-json-missing-validation-errors)
[assertJsonPath](#assert-json-path)
[assertJsonMissingPath](#assert-json-missing-path)
[assertJsonStructure](#assert-json-structure)
[assertJsonValidationErrors](#assert-json-validation-errors)
[assertJsonValidationErrorFor](#assert-json-validation-error-for)
[assertLocation](#assert-location)
[assertNoContent](#assert-no-content)
[assertNotFound](#assert-not-found)
[assertOk](#assert-ok)
[assertPlainCookie](#assert-plain-cookie)
[assertRedirect](#assert-redirect)
[assertRedirectContains](#assert-redirect-contains)
[assertRedirectToSignedRoute](#assert-redirect-to-signed-route)
[assertSee](#assert-see)
[assertSeeInOrder](#assert-see-in-order)
[assertSeeText](#assert-see-text)
[assertSeeTextInOrder](#assert-see-text-in-order)
[assertSessionHas](#assert-session-has)
[assertSessionHasInput](#assert-session-has-input)
[assertSessionHasAll](#assert-session-has-all)
[assertSessionHasErrors](#assert-session-has-errors)
[assertSessionHasErrorsIn](#assert-session-has-errors-in)
[assertSessionHasNoErrors](#assert-session-has-no-errors)
[assertSessionDoesntHaveErrors](#assert-session-doesnt-have-errors)
[assertSessionMissing](#assert-session-missing)
[assertStatus](#assert-status)
[assertSuccessful](#assert-successful)
[assertUnauthorized](#assert-unauthorized)
[assertUnprocessable](#assert-unprocessable)
[assertValid](#assert-valid)
[assertInvalid](#assert-invalid)
[assertViewHas](#assert-view-has)
[assertViewHasAll](#assert-view-has-all)
[assertViewIs](#assert-view-is)
[assertViewMissing](#assert-view-missing)

</div>

<a name="assert-cookie"></a>
#### assertCookie

レスポンスに指定するクッキーが含まれていることを宣言します。

    $response->assertCookie($cookieName, $value = null);

<a name="assert-cookie-expired"></a>
#### assertCookieExpired

レスポンスに指定するクッキーが含まれており、有効期限が切れていることを宣言します。

    $response->assertCookieExpired($cookieName);

<a name="assert-cookie-not-expired"></a>
#### assertCookieNotExpired

レスポンスに指定するクッキーが含まれており、有効期限が切れていないことを宣言します。

    $response->assertCookieNotExpired($cookieName);

<a name="assert-cookie-missing"></a>
#### assertCookieMissing

レスポンスに指定するクッキーが含まれていないことを宣言します。

    $response->assertCookieMissing($cookieName);

<a name="assert-created"></a>
#### assertCreated

レスポンスに201 HTTPステータスコードがあることを宣言します。

    $response->assertCreated();

<a name="assert-dont-see"></a>
#### assertDontSee

指定する文字列が、アプリケーションが返すレスポンスに含まれていないことを宣言します。このアサートは、２番目の引数に`false`を渡さない限り、指定する文字列を自動的にエスケープします。

    $response->assertDontSee($value, $escaped = true);

<a name="assert-dont-see-text"></a>
#### assertDontSeeText

指定する文字列がレスポンステキストに含まれていないことを宣言します。このアサートは、２番目の引数に`false`を渡さない限り、指定する文字列を自動的にエスケープします。このメソッドは、アサートを作成する前に、レスポンスコンテンツを`strip_tags` PHP関数へ渡します。

    $response->assertDontSeeText($value, $escaped = true);

<a name="assert-download"></a>
#### assertDownload

レスポンスが「ダウンロード」であることを宣言します。通常、`Response::download`レスポンス、`BinaryFileResponse`、または`Storage::download`レスポンスを返すルートが起動されたことを意味します。

    $response->assertDownload();

お望みであれば、ダウンロード可能なファイルに指定のファイル名が付与されていることを宣言することもできます。

    $response->assertDownload('image.jpg');

<a name="assert-exact-json"></a>
#### assertExactJson

レスポンスに、完全一致する指定JSONデータが含まれていることを宣言します。

    $response->assertExactJson(array $data);

<a name="assert-forbidden"></a>
#### assertForbidden

レスポンスにForbidden（403）HTTPステータスコードがあることを宣言します。

    $response->assertForbidden();

<a name="assert-header"></a>
#### assertHeader

指定するヘッダと値がレスポンスに存在することを宣言します。

    $response->assertHeader($headerName, $value = null);

<a name="assert-header-missing"></a>
#### assertHeaderMissing

指定するヘッダがレスポンスに存在しないことを宣言します。

    $response->assertHeaderMissing($headerName);

<a name="assert-json"></a>
#### assertJson

レスポンスに指定するJSONデータが含まれていることを宣言します。

    $response->assertJson(array $data, $strict = false);

`assertJson`メソッドはレスポンスを配列に変換し、`PHPUnit::assertArraySubset`を利用して、指定する配列がアプリケーションが返すJSONレスポンス内に存在しているかを確認します。したがって、JSONレスポンスに他のプロパティがある場合でも、指定するフラグメントが存在する限り、このテストは合格します。

<a name="assert-json-count"></a>
#### assertJsonCount

レスポンスJSONに、指定するキーに予想されるアイテム数の配列があることを宣言します。

    $response->assertJsonCount($count, $key = null);

<a name="assert-json-fragment"></a>
#### assertJsonFragment

レスポンス中のどこかに指定JSONデータが含まれていることを宣言します。

    Route::get('/users', function () {
        return [
            'users' => [
                [
                    'name' => 'Taylor Otwell',
                ],
            ],
        ];
    });

    $response->assertJsonFragment(['name' => 'Taylor Otwell']);

<a name="assert-json-missing"></a>
#### assertJsonMissing

レスポンスに指定するJSONデータが含まれていないことを宣言します。

    $response->assertJsonMissing(array $data);

<a name="assert-json-missing-exact"></a>
#### assertJsonMissingExact

レスポンスに完全一致するJSONデータが含まれていないことを宣言します。

    $response->assertJsonMissingExact(array $data);

<a name="assert-json-missing-validation-errors"></a>
#### assertJsonMissingValidationErrors

指定するキーのJSONバリデーションエラーがレスポンスにないことを宣言します。

    $response->assertJsonMissingValidationErrors($keys);

> **Note**
> より一般的な[assertValid](#assert-valid)メソッドを使用して、JSONで返されたレスポンスにバリデーションエラーがなく、**かつ**セッションストレージにエラーが一時保存されていないことを宣言できます。

<a name="assert-json-path"></a>
#### assertJsonPath

レスポンスに、指定するパスの指定するデータが含まれていることを宣言します。

    $response->assertJsonPath($path, $expectedValue);

たとえば、アプリケーションが以下のJSONレスポンスを返す場合：

```js
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

`user`オブジェクトの`name`プロパティが次のように指定する値に一致すると宣言することができます。

    $response->assertJsonPath('user.name', 'Steve Schoger');

<a name="assert-json-missing-path"></a>
#### assertJsonMissingPath

レスポンスに、指定するパスが含まれないことを宣言します。

    $response->assertJsonMissingPath($path);

たとえば、アプリケーションが以下のJSONレスポンスを返す場合：

```js
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

`user`オブジェクトに`email`プロパティが含まれていないことを宣言できるでしょう。

    $response->assertJsonMissingPath('user.email');

<a name="assert-json-structure"></a>
#### assertJsonStructure

レスポンスが指定のJSON構造を持っていることを宣言します。

    $response->assertJsonStructure(array $structure);

たとえば、アプリケーションが返すJSONレスポンスに次のデータが含まれている場合:

```js
{
    "user": {
        "name": "Steve Schoger"
    }
}
```

次のように、JSON構造がエクスペクテーションに一致すると宣言できます。

    $response->assertJsonStructure([
        'user' => [
            'name',
        ]
    ]);

時々、アプリケーションが返すJSONレスポンスには、オブジェクトの配列が含まれている可能性があります。

```js
{
    "user": [
        {
            "name": "Steve Schoger",
            "age": 55,
            "location": "Earth"
        },
        {
            "name": "Mary Schoger",
            "age": 60,
            "location": "Earth"
        }
    ]
}
```

この状況では、`*`文字を使って配列内のすべてのオブジェクトの構造に対して宣言できます。

    $response->assertJsonStructure([
        'user' => [
            '*' => [
                 'name',
                 'age',
                 'location'
            ]
        ]
    ]);

<a name="assert-json-validation-errors"></a>
#### assertJsonValidationErrors

レスポンスへ、指定するキーに対する指定するJSONバリデーションエラーがあることを宣言します。このメソッドは、バリデーションエラーがセッションに一時保存されているのではなく、バリデーションエラーをJSON構造として返すレスポンスに対して宣言する場合に使用する必要があります。

    $response->assertJsonValidationErrors(array $data, $responseKey = 'errors');

> **Note**
> より一般的な[assertInvalid](#assert-invalid)メソッドを使用して、JSONで返されたレスポンスにバリデーションエラーが存在した、**もしくは**エラーがセッションストレージに一時保存されたことを宣言できます。

<a name="assert-json-validation-error-for"></a>
#### assertJsonValidationErrorFor

レスポンスが指定キーに対するJSONバリデーションエラーを持っていることを宣言します。

    $response->assertJsonValidationErrorFor(string $key, $responseKey = 'errors');

<a name="assert-location"></a>
#### assertLocation

レスポンスの`Location`ヘッダに指定するURI値があることを宣言します。

    $response->assertLocation($uri);

<a name="assert-no-content"></a>
#### assertNoContent

レスポンスに指定するHTTPステータスコードがあり、コンテンツがないことを宣言します。

    $response->assertNoContent($status = 204);

<a name="assert-not-found"></a>
#### assertNotFound

レスポンスにNot Found（404）HTTPステータスコードがあることを宣言します。

    $response->assertNotFound();

<a name="assert-ok"></a>
#### assertOk

レスポンスに200 HTTPステータスコードがあることを宣言します。

    $response->assertOk();

<a name="assert-plain-cookie"></a>
#### assertPlainCookie

レスポンスに指定する暗号化されていないクッキーが含まれていることを宣言します。

    $response->assertPlainCookie($cookieName, $value = null);

<a name="assert-redirect"></a>
#### assertRedirect

レスポンスが指定するURIへのリダイレクトであることを宣言します。

    $response->assertRedirect($uri);

<a name="assert-redirect-contains"></a>
#### assertRedirectContains

レスポンスが指定文字列を含むURIへリダイレクトされることを宣言する。

    $response->assertRedirectContains($string);

<a name="assert-redirect-to-signed-route"></a>
#### assertRedirectToSignedRoute

Assert that the response is a redirect to the given [signed route](/docs/{{version}}/urls#signed-urls):

    $response->assertRedirectToSignedRoute($name = null, $parameters = []);

<a name="assert-see"></a>
#### assertSee

指定する文字列がレスポンスに含まれていることを宣言します。このアサートは、２番目の引数に`false`を渡さない限り、指定する文字列を自動的にエスケープします。

    $response->assertSee($value, $escaped = true);

<a name="assert-see-in-order"></a>
#### assertSeeInOrder

指定する文字列がレスポンス内に順番に含まれていることを宣言します。このアサートは、2番目の引数へ`false`を渡さない限り、指定する文字列を自動的にエスケープします。

    $response->assertSeeInOrder(array $values, $escaped = true);

<a name="assert-see-text"></a>
#### assertSeeText

指定する文字列がレスポンステキストに含まれていることを宣言します。このアサートは、2番目の引数に`false`を渡さない限り、指定する文字列を自動的にエスケープします。アサートが作成される前に、レスポンスの内容が`strip_tags`PHP関数に渡されます。

    $response->assertSeeText($value, $escaped = true);

<a name="assert-see-text-in-order"></a>
#### assertSeeTextInOrder

指定する文字列がレスポンステキスト内に順番に含まれていることを宣言します。このアサートは、２番目の引数に`false`を渡さない限り、指定する文字列を自動的にエスケープします。アサートが作成される前に、レスポンスの内容が`strip_tags`PHP関数に渡されます。

    $response->assertSeeTextInOrder(array $values, $escaped = true);

<a name="assert-session-has"></a>
#### assertSessionHas

セッションに指定するデータが含まれていることを宣言します。

    $response->assertSessionHas($key, $value = null);

必要であれば、クロージャを`assertSessionHas`メソッドの第２引数へ指定できます。クロージャが`true`を返せば、アサートは成功します。

    $response->assertSessionHas($key, function ($value) {
        return $value->name === 'Taylor Otwell';
    });

<a name="assert-session-has-input"></a>
#### assertSessionHasInput

セッションの[一時保存されている入力配列](/docs/{{version}}/response#redirecting-with-flashed-session-data)に指定する値があることを宣言します。

    $response->assertSessionHasInput($key, $value = null);

必要であれば、クロージャを`assertSessionHasInput`メソッドの第２引数へ指定できます。クロージャが`true`を返せば、アサートは成功します。

    $response->assertSessionHasInput($key, function ($value) {
        return Crypt::decryptString($value) === 'secret';
    });

<a name="assert-session-has-all"></a>
#### assertSessionHasAll

セッションにキー／値ペアの指定配列が含まれていることを宣言します。

    $response->assertSessionHasAll(array $data);

たとえば、アプリケーションのセッションに`name`キーと`status`キーが含まれている場合、両方が存在し、以下のような指定値を持っていると宣言できます。

    $response->assertSessionHasAll([
        'name' => 'Taylor Otwell',
        'status' => 'active',
    ]);

<a name="assert-session-has-errors"></a>
#### assertSessionHasErrors

指定する`$keys`のエラーがセッションに含まれていることを宣言します。`$keys`が連想配列の場合、セッションに各フィールド(キー)の特定のエラーメッセージ(値)が含まれていることを宣言します。このメソッドは、バリデーションエラーをJSON構造として返すのではなく、セッションに一時保存するルートをテストするときに使用する必要があります。

    $response->assertSessionHasErrors(
        array $keys, $format = null, $errorBag = 'default'
    );

たとえば、`name`フィールドと`email`フィールドにセッションへ一時保存された検証エラーメッセージがあることを宣言するには、以下のように`assertSessionHasErrors`メソッドを呼び出すことができます。

    $response->assertSessionHasErrors(['name', 'email']);

または、特定のフィールドに特定のバリデーションエラーメッセージがあると宣言することもできます。

    $response->assertSessionHasErrors([
        'name' => 'The given name was invalid.'
    ]);

> **Note**
> より汎用的な[assertInvalid](#assert-invalid)メソッドは、レスポンスがJSONとして返されたバリデーションエラーを持つか、**または**エラーがセッションストレージに一時保存されたことを宣言するため使用できます。

<a name="assert-session-has-errors-in"></a>
#### assertSessionHasErrorsIn

指定[エラーバッグ](/docs/{{version}}/validation#named-error-bags)内に指定する`$keys`のエラーがセッションに含まれていることを宣言します。`$keys`が連想配列の場合、セッションにはエラーバッグ内の各フィールド(キー)に特定のエラーメッセージ(値)が含まれていることを宣言します。

    $response->assertSessionHasErrorsIn($errorBag, $keys = [], $format = null);

<a name="assert-session-has-no-errors"></a>
#### assertSessionHasNoErrors

セッションにバリデーションエラーがないことを宣言します。

    $response->assertSessionHasNoErrors();

<a name="assert-session-doesnt-have-errors"></a>
#### assertSessionDoesntHaveErrors

指定するキーのバリデーションエラーがセッションにないことを宣言します。

    $response->assertSessionDoesntHaveErrors($keys = [], $format = null, $errorBag = 'default');

> **Note**
> より汎用的な[assertValid](#assert-valid)メソッドは、レスポンスにJSON形式で返された結果にバリデーションエラーがなく、**かつ**セッションストレージにエラーが保存されていないことを宣言するために使用できます。

<a name="assert-session-missing"></a>
#### assertSessionMissing

セッションに指定するキーが含まれていないことを宣言します。

    $response->assertSessionMissing($key);

<a name="assert-status"></a>
#### assertStatus

レスポンスに指定HTTPステータスコードがあることを宣言します。

    $response->assertStatus($code);

<a name="assert-successful"></a>
#### assertSuccessful

レスポンスに成功した(>=200および<300)HTTPステータスコードがあることを宣言します。

    $response->assertSuccessful();

<a name="assert-unauthorized"></a>
#### assertUnauthorized

レスポンスに不正な(401)HTTPステータスコードがあることを宣言します。

    $response->assertUnauthorized();

<a name="assert-unprocessable"></a>
#### assertUnprocessable

レスポンスに未処理のエンティティ(422)HTTPステータスコードがあることを宣言します。

    $response->assertUnprocessable();

<a name="assert-valid"></a>
#### assertValid

レスポンスに指定キーのバリデーションエラーがないことを宣言します。このメソッドは、バリデーションエラーをJSON構造として返す、もしくはバリデーションエラーをセッションへ一時保存しているレスポンスに対する宣言に使用します。

    // バリデーションエラーが発生していないことを宣言
    $response->assertValid();

    // 指定キーのバリデーションエラーがないことを宣言
    $response->assertValid(['name', 'email']);

<a name="assert-invalid"></a>
#### assertInvalid

レスポンスに指定キーのバリデーションエラーがあることを宣言します。このメソッドは、バリデーションエラーをJSON構造として返す、もしくはバリデーションエラーをセッションへ一時保存しているレスポンスに対する宣言に使用します。

    $response->assertInvalid(['name', 'email']);

特定のキーに特定のバリデーションエラーメッセージがあることを宣言することもできます。その場合、メッセージ全体またはメッセージのごく一部だけを指定できます。

    $response->assertInvalid([
        'name' => 'The name field is required.',
        'email' => 'valid email address',
    ]);

<a name="assert-view-has"></a>
#### assertViewHas

指定するデータの部分がレスポンスビューに含まれていることを宣言します。

    $response->assertViewHas($key, $value = null);

`assertViewHas`メソッドの２番目の引数にクロージャを渡すと、ビューデータの特定部分に対して検査し、アサートできます。

    $response->assertViewHas('user', function (User $user) {
        return $user->name === 'Taylor';
    });

さらに、ビューデータはレスポンスの配列変数としてアクセスできるため、次のように簡単に検査できます。

    $this->assertEquals('Taylor', $response['name']);

<a name="assert-view-has-all"></a>
#### assertViewHasAll

レスポンスビューに指定するデータのリストがあることを宣言します。

    $response->assertViewHasAll(array $data);

このメソッドは、ビューに指定するキーに一致するデータが含まれていることを宣言するために使用できます。

    $response->assertViewHasAll([
        'name',
        'email',
    ]);

または、ビューデータが存在し、特定の値を持っていると宣言することもできます。

    $response->assertViewHasAll([
        'name' => 'Taylor Otwell',
        'email' => 'taylor@example.com,',
    ]);

<a name="assert-view-is"></a>
#### assertViewIs

指定するビューがルートによって返されたことを宣言します。

    $response->assertViewIs($value);

<a name="assert-view-missing"></a>
#### assertViewMissing

指定するデータキーが、アプリケーションのレスポンスで返されたビューで使用可能になっていないことを宣言します。

    $response->assertViewMissing($key);

<a name="authentication-assertions"></a>
### 認証のアサート

Laravelは、アプリケーションの機能テストで利用できるさまざまな認証関連のアサートも提供します。これらのメソッドは、`get`や`post`などのメソッドによって返される`Illuminate\Testing\TestResponse`インスタンスではなく、テストクラス自体で呼び出されることに注意してください。

<a name="assert-authenticated"></a>
#### assertAuthenticated

ユーザーが認証済みであることを宣言します。

    $this->assertAuthenticated($guard = null);

<a name="assert-guest"></a>
#### assertGuest

ユーザーが認証されていないことを宣言します。

    $this->assertGuest($guard = null);

<a name="assert-authenticated-as"></a>
#### assertAuthenticatedAs

特定のユーザーが認証済みであることを宣言します。

    $this->assertAuthenticatedAs($user, $guard = null);
