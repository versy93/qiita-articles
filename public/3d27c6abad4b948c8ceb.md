---
title: DockerでLaravel×Elasticsearch環境を作る
tags:
  - PHP
  - MySQL
  - Elasticsearch
  - Docker
private: false
updated_at: '2022-01-01T14:36:52+09:00'
id: 3d27c6abad4b948c8ceb
organization_url_name: null
slide: false
ignorePublish: false
---
# 前提条件



## 環境

・ローカル

検証端末：MacOS Monterey (12.1)

docker：Docker version 20.10.11, build dea9396

docker-compose：v2.2.1



・Docker環境

PHP：8.0.13

MySQL：8.0.27

Elasticsearch：7.12.0



## 読むべき人

Elasticsearch導入したことない人。

家で検証してみたい人。

Elasticという響きがかっこいいなーと思ってる人。





# コンテナの構成

・nginx

リクエストを最初に受けるサーバー



・php

Laravelが動作するコンテナ。



・mysql

メインのデータストレージとして使用するDB。



・Elasticsearch

全文検索エンジン。Scout経由で使用する想定。



・node

画面実装にはないと困るので。



# Laravelのモジュールをクローンする

```
git clone https://github.com/laravel/laravel.git
```



# 今回作成する環境のディレクトリ構成

```
elasticsearch_test/   ←ディレクトリ名は任意のものに変更
├── CHANGELOG.md
├── README.md
├── app
├── artisan
├── bootstrap
├── composer.json
├── composer.lock
├── config
├── database
├── docker   ←[追加] 設定ファイルを追加
├── docker-compose.yml   ←[追加] 設定ファイルを追加
├── package.json
├── phpunit.xml
├── public
├── resources
├── routes
├── server.php
├── src
├── storage
├── tests
├── vendor
└── webpack.mix.js
```



# 任意のプロジェクト名に変更

今回は「elasticsearch_test」とする。

```
mv laravel elasticsearch_test
```





# Dockerの設定ファイルを追加していく

## nginxの設定ファイル

nginxコンテナ。

Laravelのrootディレクトリはpublic配下なので設定しておく。

listenするポートはコンテナの内側なのでwebサーバーのデフォルトの80でおk。



docker/nginx/default.conf

```nginx
server {
    listen 80;
        index index.php index.html;
        root /var/www/elasticsearch_test/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {

    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass php:9000;
    fastcgi_index index.php;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```



## phpコンテナのDockerfile

phpコンテナはimageではなくDockerfileで設定を指定する。

理由はコンテナ起動時にRUNで叩きたいコマンドがあるからだと思うけど、今回はその辺から拝借してきた記述をそのままにしてある。



docker/php/Dockerfile

```shell
FROM php:8.0-fpm
COPY php.ini /usr/local/etc/php/

RUN apt-get update \
    && apt-get install -y zlib1g-dev mariadb-client vim libzip-dev \
    && docker-php-ext-install zip pdo_mysql

#Composer install
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
RUN php composer-setup.php
RUN php -r "unlink('composer-setup.php');"
RUN mv composer.phar /usr/local/bin/composer

ENV COMPOSER_ALLOW_SUPERUSER 1

ENV COMPOSER_HOME /composer

ENV PATH $PATH:/composer/vendor/bin


WORKDIR /var/www

COPY . /var/www/

RUN composer global require "laravel/installer"
```



## php.ini

docker/php/php.ini

```ini
[Date]
date.timezone = "Asia/Tokyo"
[mbstring]
mbstring.internal_encoding = "UTF-8"
mbstring.language = "Japanese"
```



## docker-compose.yml

コンテナ達を1セットに扱うためのdocker-compose。

それの設定ファイルを定義しておきます。

```yaml
version: '3'

services:
  php:
    container_name: php_container
    build: ./docker/php
    volumes:
      - ./:/var/www/elasticsearch_test

  nginx:
    container_name: nginx_container
    image: nginx
    ports:
      - 81:80
    volumes:
      - ./:/var/www/elasticsearch_test
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php

  db:
    container_name: mysql_container
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: database
      MYSQL_PASSWORD: root
      TZ: 'Asia/Tokyo'
    command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    volumes:
      - ./docker/db/data:/var/lib/mysql
      - ./docker/db/my.cnf:/etc/mysql/conf.d/my.cnf
      - ./docker/db/sql:/docker-entrypoint-initdb.d
    ports:
      - 3306:3306

  es:
    container_name: elasticsearch_container
    image: elasticsearch:7.12.0
    ports:
      - "9200:9200"
    environment:
      - discovery.type=single-node
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1

  node:
    image: node:12.13-alpine
    tty: true
    volumes:
      - ./src:/var/www
    working_dir: /var/www

```



# コンテナ起動

起動！

```
docker-compose up -d
```



# コンテナで実行

vendor周りを作成。

composerの意味がわからずvendor配下を他プロジェクトからコピーしてきていた日が僕にもありました。

依存ライブラリはちゃんとcomposerでインストールしよう！

```
composer install
```



envファイルを作成

```
cp .env.example .env
```



keyを生成

これあんまり意味分かってない。謎の儀式。

```
php artisan key:generate
```



# 疎通確認



## phpコンテナ

ブラウザでアクセス

`localhost:81`



## mysqlコンテナ

コンテナにログイン

`docker exec -it mysql_container /bin/bash`



Mysqlにログイン

```
mysql -uroot -proot
```



Databaseを作成する

```
create database elasticsearch_test_db;
```



Mysql8系はデフォルトの接続方法が異なるため変更する

```
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY 'root';
```



.envファイルのDBホスト情報をdocker-composeで定義したサービス名に変更

```
DB_CONNECTION=mysql
DB_HOST=db // ←ここ
DB_PORT=3306
DB_DATABASE=elasticsearch_test_db
DB_USERNAME=root
DB_PASSWORD=root
```





## Elasticsearchコンテナ

とりあえずコンテナに入る

`docker exec -it elasticsearch_container /bin/bash`



Elasticserachコンテナの中からなのでhostは「localhost」

```json
[root@8970f3395096 elasticsearch]# curl -X GET localhost:9200
{
  "name" : "8970f3395096",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "JKMTJ9JYRDuTfq0o4phKIQ",
  "version" : {
    "number" : "7.12.0",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "78722783c38caa25a70982b5b042074cde5d3b3a",
    "build_date" : "2021-03-18T06:17:15.410153305Z",
    "build_snapshot" : false,
    "lucene_version" : "8.8.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```



.envにElasticsearchの設定を追加。

識別はdocker-compose.ymlのserviceに書いたservice名

```
SCOUT_DRIVER=elasticsearch
ELASTICSEARCH_HOST=es:9200
```





# サンプルとなるDBを用意する



## migration

migrationファイルを作成

```
php artisan make:migration create_scout_test_records_table --create=scout_test_records
```

database/migrations/2021_12_17_xxxxxx_create_scout_test_records_table.php のサンプル

```php:database/migrations/2021_12_17_xxxxxx_create_scout_test_records_table.php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreateScoutTestRecordsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('scout_test_records', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name', 255)->comment('名前');
            $table->string('hash_code', 255)->comment('ハッシュコード');
            $table->text('text')->nullable()->comment('テキスト');
            $table->double('latitude')->nullable()->comment('緯度');
            $table->double('longitude')->nullable()->comment('経度');
            $table->integer('type')->comment('タイプ');
            $table->tinyInteger('status')->comment('ステータス');
            $table->timestamps();
            $table->softDeletes();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('scout_test_records');
    }
}

```



## Model

modelを作成

```
php artisan make:model ScoutTestRecord
```



```php:Models/ScoutTestRecord.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class ScoutTestRecord extends Model
{
    use HasFactory;

    use SoftDeletes;

    public $table = 'scout_test_records';

    public $dates = ['deleted_at'];

    public $fillable = [
        'id',
        'name',
        'hash_code',
        'text',
        'latitude',
        'longitude',
        'type',
        'status',
    ];

    protected $casts = [
        'id' => 'integer',
        'name' => 'string',
        'hash_code' => 'string',
        'text' => 'string',
        'latitude' => 'double',
        'longitude' => 'double',
        'type' => 'integer',
        'status' => 'integer',
    ];

    public static $rules = [
        'name' => 'required|string',
        'hash_code' => 'required|string',
        'text' => 'string',
        'type' => 'integer',
        'status' => 'integer',
    ];

}

```



## Seeder

seederも作成しておく

```
php artisan make:seeder ScoutTestRecordsTableSeeder
```

```php:database/seeders/ScoutTestRecordsTableSeeder.php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\ScoutTestRecord;
use Faker\Factory as Faker;

class ScoutTestRecordsTableSeeder extends Seeder
{
    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        ScoutTestRecord::factory()->count(20)->create();
    }
}
```



DatabaseSeederにSeederファイルを登録する。
こうすると `php artisan migrate --seed` でseederが実行される。

```php:DatabaseSeeder.php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Database\Seeders\ScoutTestRecordsTableSeeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the application's database.
     *
     * @return void
     */
    public function run()
    {
        $this->call([
            ScoutTestRecordsTableSeeder::class,
        ]);
    }
}

```





## Factory

config/app.php　factoryの設定をするため一部変更する

```
    'fallback_locale' => 'en',
```

factoryを作成

```
php artisan make:factory ScoutTestRecordFactory --model=ScoutTestRecord
```


```php:database/factories/ScoutTestRecordFactory.php
<?php

namespace Database\Factories;

use App\Models\ScoutTestRecord;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Carbon;

class ScoutTestRecordFactory extends Factory
{
    protected $model = ScoutTestRecord::class;

    /**
     * Define the model's default state.
     *
     * @return array
     */
    public function definition()
    {
        return [
            'name' => $this->faker->name,
            'hash_code' => $this->faker->password(),
            'text' => $this->faker->realText($maxNbChars = 50, $indexSize = 2),
            'latitude' => $this->faker->latitude(-90, 90),
            'longitude' => $this->faker->longitude(0, 180),
            'type' => $this->faker->randomNumber($nbDigits = 1),
            'status' => $this->faker->randomNumber($nbDigits = 1),
            'created_at' => Carbon::now(),
            'updated_at' => Carbon::now(),
            'deleted_at' => null,
        ];
    }
}
```





DBにデータを登録する

```
php artisan migrate --seed
```



# Elasticsearchにデータを連携する

各モジュールの役割は下記の記事で解説してます。



https://qiita.com/yodai/items/c5ddf5ec3b1f0c3d1724



## Scoutの設定

まずはScoutをインストールする。

このあとインストールするエンジンがscoutの8.x系に依存しているため、バージョンを指定する。

バージョン指定のセマンティックバージョンの概念はこれがわかりやすかったです。

https://zenn.dev/kazma_13/articles/422ffe9026fd56

```
composer require laravel/scout:^8.0
```



設定ファイルをバージョン管理配下に配置する

```
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```



連携したいデータソースのModelにモデルオブザーバを登録する

```php:app/Models/ScoutTestRecord.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Laravel\Scout\Searchable; // 追加

class ScoutTestRecord extends Model
{
    use HasFactory;

    use SoftDeletes;
    
    use Searchable; // 追加

    public $table = 'scout_test_records';
・
・
・

```



## エンジンを設定



Elasticsearchのクライアントモジュールをインストール

```
composer require elasticsearch/elasticsearch
```



エンジンは先人達がけっこういっぱい用意してくれてるけど、今回はtamayoさんのやつをチョイス。

```
composer require tamayo/laravel-scout-elastic
```



エンジンの処理の挙動はカスタマイズしたい場合があるので、app配下にエンジンを継承したファイルを用意する。

```
mkdir app/Scout/ 

touch app/Scout/ElasticsearchEngine.php
```


ElasticsearchEngine.php。
若干挙動がおかしいところがあったので継承して修正。

```php:app/Scout/ElasticsearchEngine.php
<?php

namespace App\Scout;

use Laravel\Scout\Builder;
use Elasticsearch\Client;
use Tamayo\LaravelScoutElastic\Engines\ElasticsearchEngine as ScoutElasticsearchEngine;

class ElasticsearchEngine extends ScoutElasticsearchEngine
{
    protected $elastic;
    protected $index;
    private $query;

    public function __construct(Client $elastic, $index)
    {
        $this->elastic = $elastic;
        $this->index = $index;
    }

    protected function performSearch(Builder $builder, array $options = [])
    {
        $params = [
            'index' => $builder->model->searchableAs(),
            'type' => get_class($builder->model),
            'body' => [
                'query' => [
                    'bool' => [
                        'must' => [['query_string' => ['query' => "*{$builder->query}*"]]]
                    ]
                ]
            ]
        ];

        if ($sort = $this->sort($builder)) {
            $params['body']['sort'] = $sort;
        }

        if (isset($options['from'])) {
            $params['body']['from'] = $options['from'];
        }

        if (isset($options['size'])) {
            $params['body']['size'] = $options['size'];
        }

        if (isset($options['z']) && count($options['numericFilters'])) {
            $params['body']['query']['bool']['must'] = array_merge(
                $params['body']['query']['bool']['must'],
                $options['numericFilters']
            );
        }

        if ($builder->callback) {
            return call_user_func(
                $builder->callback,
                $this->elastic,
                $builder->query,
                $params
            );
        }

        return $this->elastic->search($params);
    }

    /**
     * Perform the given search on the engine.
     * @inheritDoc
     * @see ScoutElasticsearchEngine::paginate
     * @param  Builder  $builder
     * @param  int  $perPage　(limit)
     * @param  int  $page (offset)
     * @return mixed
     */
    public function paginate(Builder $builder, $perPage, $page)
    {
        $result = $this->performSearch($builder, [
            'numericFilters' => $this->filters($builder),
            'from' => (($page * $perPage) - $perPage),
            'size' => $perPage,
        ]);

        $result['nbPages'] = $result['hits']['total']['value'] / $perPage;

        return $result;
    }

    /**
     * Map the given results to instances of the given model.
     *
     * @inheritDoc
     * @see ScoutElasticsearchEngine::map
     * @param  \Laravel\Scout\Builder  $builder
     * @param  mixed  $results
     * @param  \Illuminate\Database\Eloquent\Model  $model
     * @return Collection
     */
    public function map(Builder $builder, $results, $model)
    {
        if ($results['hits']['total']['value'] === 0) {
            return $model->newCollection();
        }

        $keys = collect($results['hits']['hits'])->pluck('_id')->values()->all();

        $modelIdPositions = array_flip($keys);

        return $model->getScoutModelsByIds(
            $builder,
            $keys
        )->filter(function ($model) use ($keys) {
            return in_array($model->getScoutKey(), $keys);
        })->sortBy(function ($model) use ($modelIdPositions) {
            return $modelIdPositions[$model->getScoutKey()];
        })->values();
    }

    /**
     * Get the total count from a raw result returned by the engine.
     *
     * @inheritDoc
     * @see ScoutElasticsearchEngine::getTotalCount
     * @param  mixed  $results
     * @return int
     */
    public function getTotalCount($results)
    {
        return $results['hits']['total']['value'];
    }
}

```



## Providerを読み込む



configを設定。

```php:config/app.php
    'providers' => [

        /*
         * Laravel Framework Service Providers...
         */
        Illuminate\Auth\AuthServiceProvider::class,
        Illuminate\Broadcasting\BroadcastServiceProvider::class,
・
・
・

        // Elasticsearch
        Tamayo\LaravelScoutElastic\LaravelScoutElasticProvider::class, // ←これを追加
    ],
```



## 軽く動作確認

ここまで書いたらphpのコンテナにログインしてScoutが提供しているデータの取り込みスクリプトを実行する。

```
php artisan scout:import "App\Models\ScoutTestRecord"


Imported [App\Models\ScoutTestRecord] models up to ID: 20
All [App\Models\ScoutTestRecord] records have been imported.

// こうなれば成功
```



次はElasticsearchに検索リクエストを投げてみる。

コンテナ間の通信はdocker-composeのおかげでservice名で識別できる。

```
curl -X GET es:9200/scout_test_records/_search

{"took":9,"timed_out":false,"_shards":{"total":1,"successful":1,"skipped":0,"failed":0},"hits":{"total":{"value":20,"relation":"eq"},"max_score":1.0,"hits":[{"_index":"scout_test_records","_type":"App\\Models\\ScoutTestRecord","_id":"1","_score":1.0,"_source":{"id":1,"name":"山田 涼平","has...

// 結果が返却されればOK!
```



# まとめ

MySQL8.0とかPHP8を使ったせいで実プロジェクトで完コピして作業するには難しいかも。

でもおおまかな流れは全て一緒で、変わるとしたらcomposerを使って依存ライブラリを導入した時のバージョンが変わるくらいかなー。



あと使ったライブラリがあきらか個人名みたいな名前だけどMITライセンス、みたいな感じ、

これってどれくらい信用度あるんだろ。

フレームワーク本体とかよりは信頼度低い気がする。



そのまま使ったら普通にエラーになったし。



クエリの使い方とかちょいちょい上げていきます。



記載漏れとかあったら是非ご連絡ください！

