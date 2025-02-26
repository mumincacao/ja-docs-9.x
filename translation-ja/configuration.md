# 設定

- [イントロダクション](#introduction)
- [環境設定](#environment-configuration)
    - [環境変数タイプ](#environment-variable-types)
    - [環境設定の取得](#retrieving-environment-configuration)
    - [現在環境の決定](#determining-the-current-environment)
- [設定値へのアクセス](#accessing-configuration-values)
- [設定キャッシュ](#configuration-caching)
- [デバッグモード](#debug-mode)
- [メンテナンスモード](#maintenance-mode)

<a name="introduction"></a>
## イントロダクション

Laravelフレームワークの全設定ファイルは、`config`ディレクトリに保存されています。各オプションには詳しいコメントが付いているので、各ファイルを一読し、使用できるオプションを把握しておきましょう。

これら設定ファイルを使用すると、データベース接続情報、メールサーバ情報、およびアプリケーションのタイムゾーンや暗号化キーなどの他のさまざまなコア設定値などを設定できます。

<a name="application-overview"></a>
#### アプリケーションの概要

お急ぎですか？`about` Artisanコマンドで、アプリケーションの設定、ドライバ、環境の概要を簡単に確認できます。

```shell
php artisan about
```

アプリケーション概要の出力のうち、特定のセクションのみ興味がある場合は、`--only`オプションを使用してそのセクションをフィルタリングすることができます。

```shell
php artisan about --only=environment
```

<a name="environment-configuration"></a>
## 環境設定

アプリケーションを実行している環境にもとづき、別の設定値に切り替えられると便利です。たとえば、ローカルと実働サーバでは、異なったキャッシュドライバを使いたいことでしょう。

これを簡単に実現するために、Laravelは[DotEnv](https://github.com/vlucas/phpdotenv) PHPライブラリを利用しています。Laravelの新規インストールでは、アプリケーションのルートディレクトリに、多くの一般的な環境変数を定義する`.env.example`ファイルが含まれます。Laravelのインストールプロセス中に、このファイルは自動的に`.env`へコピーされます。

Laravelのデフォルトの`.env`ファイルには、アプリケーションがローカルで実行されているか本番Webサーバで実行されているかにより異なる可能性のある、一般的な設定値が含まれています。これらの値は、Laravelの`env`関数を使用して`config`ディレクトリ内のさまざまなLaravel設定ファイルから取得されています。

チームで開発している場合は、アプリケーションに`.env.example`ファイルを含め続けることをお勧めします。サンプル設定ファイルにプレースホルダー値を配置することにより、チームの他の開発者は、アプリケーションを実行するために必要な環境変数を明確に確認できます。

> **Note**
> `.env`ファイルにあるすべての変数は、サーバレベルやシステムレベルで定義されている、外部の環境変数によってオーバーライドすることができます。

<a name="environment-file-security"></a>
#### 環境ファイルのセキュリティ

アプリケーションを使用する開発者/サーバごとに異なる環境設定が必要になる可能性があるため、`.env`ファイルをアプリケーションのソース管理にコミットしないでください。さらに、機密性の高い資格情報が公開されるため、侵入者がソース管理リポジトリにアクセスした場合のセキュリティリスクになります。

<a name="additional-environment-files"></a>
#### 追加の環境ファイル

アプリケーションの環境変数を読み込む前に、Laravelは`APP_ENV`環境変数が外部から提供されているか、もしくは`--env` CLI引数が指定されているかを判断します。その場合、Laravelは`.env.[APP_ENV]`ファイルが存在すれば、それを読み込もうとします。存在しない場合は、デフォルトの`.env`ファイルを読み込みます。

<a name="environment-variable-types"></a>
### 環境変数タイプ

通常、`.env`ファイル内のすべての変数は文字列として解析されるため、`env()`関数からより広範囲の型を返せるように、いくつかの予約値が作成されています。

| `.env`値     | `env()`値      |
|--------------|---------------|
| true         | (bool) true   |
| (true)       | (bool) true   |
| false        | (bool) false  |
| (false)      | (bool) false  |
| empty        | (string) ''   |
| (empty)      | (string) ''   |
| null         | (null) null   |
| (null)       | (null) null   |

スペースを含む値で環境変数を定義する必要がある場合は、値をダブルクォーテーションで囲むことによって定義できます。

```ini
APP_NAME="My Application"
```

<a name="retrieving-environment-configuration"></a>
### 環境設定の取得

このファイルでリストされているすべての変数は、アプリケーションがリクエストを受信すると、`$_ENV` PHPスーパーグローバルへロードされます。ただし、`env`ヘルパを使用して、設定ファイル内のこれらの変数から値を取得することができます。実際、Laravel設定ファイルを確認すると、多くのオプションがすでにこのヘルパを使用していることがわかります。

    'debug' => env('APP_DEBUG', false),

`env`関数に渡す２番目の値は「デフォルト値」です。指定されたキーの環境変数が存在しない場合、この値が返されます。

<a name="determining-the-current-environment"></a>
### 現在環境の決定

現在のアプリケーション環境は、`.env`ファイルの`APP_ENV`変数により決まります。`APP`[ファサード](/docs/{{version}}/facades)の`environment`メソッドにより、この値へアクセスできます。

    use Illuminate\Support\Facades\App;

    $environment = App::environment();

`environment`メソッドに引数を渡して、環境が特定の値と一致するかどうかを判定することもできます。環境が指定された値のいずれかに一致する場合、メソッドは`true`を返します。

    if (App::environment('local')) {
        // 環境はlocal
    }

    if (App::environment(['local', 'staging'])) {
        // 環境はlocalかstaging
    }

> **Note**
> 現在のアプリケーション環境の検出は、サーバレベルの`APP_ENV`環境変数を定義することで上書きできます。

<a name="accessing-configuration-values"></a>
## 設定値へのアクセス

アプリケーションのどこからでもグローバルの`config`ヘルパ関数を使用し、設定値へ簡単にアクセスできます。設定値はファイルとオプションの名前を含む「ドット」記法を使いアクセスします。デフォルト値も指定でき、設定オプションが存在しない場合に、返されます。

    $value = config('app.timezone');

    // 設定値が存在しない場合、デフォルト値を取得する
    $value = config('app.timezone', 'Asia/Seoul');

実行時に設定値をセットするには、`config`ヘルパへ配列で渡してください。

    config(['app.timezone' => 'America/Chicago']);

<a name="configuration-caching"></a>
## 設定キャッシュ

アプリケーションの速度を上げるには、`config:cache` Artisanコマンドを使用してすべての設定ファイルを1つのファイルへキャッシュする必要があります。これにより、アプリケーションのすべての設定オプションが１ファイルに結合され、フレームワークによってすばやくロードされます。

通常、本番デプロイメントプロセスの一部として`php artisan config:cache`コマンドを実行する必要があります。アプリケーションの開発中は設定オプションを頻繁に変更する必要があるため、ローカル開発中はコマンドを実行しないでください。

> **Warning**
> 開発過程の一環として`config:cache`コマンド実行を採用する場合は、必ず`env`関数を設定ファイルの中だけで使用してください。設定ファイルがキャッシュされると、`.env`ファイルはロードされません。したがって、`env`関数は外部システムレベルの環境変数のみを返すだけです。

<a name="debug-mode"></a>
## デバッグモード

`config/app.php`設定ファイルの`debug`オプションは、エラーに関する情報が実際にユーザーに表示される量を決定します。デフォルトでは、このオプションは、`.env`ファイルに保存されている`APP_DEBUG`環境変数の値を尊重するように設定されています。

ローカル開発の場合は、`APP_DEBUG`環境変数を`true`に設定する必要があります。**実稼働環境では、この値は常に`false`である必要があります。本番環境で変数が`true`に設定されていると、機密性の高い設定値がアプリケーションのエンドユーザーに公開されるリスクがあります。**

<a name="maintenance-mode"></a>
## メンテナンスモード

アプリケーションをメンテナンスモードにすると、アプリケーションに対するリクエストに対し、すべてカスタムビューが表示されるようになります。アプリケーションのアップデート中や、メンテナンス中に、アプリケーションを簡単に「停止」状態にできます。メンテナンスモードのチェックは、アプリケーションのデフォルトミドルウェアスタックに含まれています。アプリケーションがメンテナンスモードの時、ステータスコード503で`Symfony\Component\HttpKernel\Exception\HttpException`インスタンスを投げます。

メンテナンスモードにするには、`down` Artisanコマンドを実行します。

```shell
php artisan down
```

メンテナンスモードのすべてのレスポンスで、`Refresh`のHTTPヘッダを送信したい場合は、`down`コマンドを実行する際に、`refresh`オプションを指定します。`Refresh`ヘッダは、指定した秒数後にページを自動的に更新するようブラウザに指示します。

```shell
php artisan down --refresh=15
```

また、`down`コマンドに`retry`オプションを指定し、HTTPヘッダの`Retry-After`値を設定できますが、通常ブラウザはこのヘッダを無視します。

```shell
php artisan down --retry=60
```

<a name="bypassing-maintenance-mode"></a>
#### メンテナンスモードをバイパスする

メンテナンスモード状態でも`secret`オプションを使い、メンテナンスモードパイパストークンを指定できます。

```shell
php artisan down --secret="1630542a-246b-4b66-afa1-dd72a4c43515"
```

アプリケーションをメンテナンスモードにしたあとで、このトークンと同じURLによりブラウザでアプリケーションにアクセスすると、メンテナンスモードバイパスクッキーがそのブラウザへ発行されます。

```shell
https://example.com/1630542a-246b-4b66-afa1-dd72a4c43515
```

この隠しルートへアクセスすると、次にアプリケーションの`/`ルートへリダイレクトされます。ブラウザへこのクッキーが一度発行されると、メンテナンスモードでない状態と同様に、アプリケーションへ普通にブラウズできます。

> **Note**
> メンテナンスモードのシークレットは、通常、英数字とオプションでダッシュで構成されるべきです。URLの中で特別な意味を持つ文字、例えば `?` の使用は避けるべきです。

<a name="pre-rendering-the-maintenance-mode-view"></a>
#### Viewメンテナンスモードビューの事前レンダリング

開発時に`php artisan down`コマンドを使うと、Composerの依存パッケージやその他の基盤コンポーネントのアップデート中に、アプリケーションへユーザーがアクセスすると、エラーが発生することがあります。この理由は、アプリケーションがメンテナンスモードであると判断することやテンプレートエンジンによりメンテナンスモードビューをレンダするには、Laravelフレームワークのかなりの部分が起動している必要があるからです。

このため、Laravelはリクエストサイクルの最初に返されるメンテナンスモードビューを事前レンダできます。このビューは、アプリケーションの依存関係が読み込まれる前にレンダされます。`down`コマンドの`render`オプションで、選んだテンプレートを事前レンダできます。

```shell
php artisan down --render="errors::503"
```

<a name="redirecting-maintenance-mode-requests"></a>
#### メンテナンスモードのリクエストのリダイレクト

URI:メンテナンスモード中、Laravelはユーザーがアクセスしてきたアプリケーションの全URLに対し、メンテナンスモードビューを表示します。お望みならば、全リクエストを特定のＵＲＬへリダイレクトすることも可能です。`redirect`オプションを使用してください。例として、全リクエストを`/`のＵＲＩへリダイレクトするとしましょう。

```shell
php artisan down --redirect=/
```

<a name="disabling-maintenance-mode"></a>
#### メンテナンスモードの無効化

メンテナンスモードから抜けるには、`up`コマンドを使います。

```shell
php artisan up
```

> **Note**
> `resources/views/errors/503.blade.php`を独自に定義することにより、メンテナンスモードのデフォルトテンプレートをカスタマイズできます。

<a name="maintenance-mode-queues"></a>
#### メンテナンスモードとキュー

アプリケーションがメンテナンスモードの間、[キューされたジョブ](/docs/{{version}}/queues)は実行されません。メンテナンスモードから抜け、アプリケーションが通常状態へ戻った時点で、ジョブは続けて処理されます。

<a name="alternatives-to-maintenance-mode"></a>
#### メンテナンスモードの代替

メンテナンスモードではアプリケーションに数秒のダウンタイムが必要なため、Laravelを使う開発においてはダウンタイムゼロを達成するために、[Laravel Vapor](https://vapor.laravel.com)や[Envoyer](https://envoyer.io)などの代替手段を検討してください。
