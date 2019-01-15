---

# Flutter command

- Flutter勉強会 in 福岡 #3 & Fukuoka.dart #1 2019/01/23
- osamu_arita (github: noeloasis, twitter: @osamu_arita)

---

## 発表者について

- あああ

---

## Serverlessとは？

- ステートレスなコンピューティングコンテナ上で動作 |
- イベント駆動型である |
- Ephemaral（非永続的）である |
- Function as Service (FaaS) |
- AWS Labmda, Google Cloud Functions, Azure Fnctions, etc. |

---

## AWS Lambda

- AWS Lambdaは下記のイベントをトリガーとして、Java, Python, Node.js, C#で記述された関数を実行する。
    - API Gateway
    - S3へのPUT, POST, COPY, DELETE
    - Dynamo DBのtrigger
    - Kinesis Streams(AWS版Apache Kafka)のpoll
    - SNS(PubSub), SES(Eメール), 
    - Cognito(認証)
    - CloudWatch
    - CodeCommit, Alexa, Lex, などなど
- AWS Lambdaコンソール |

--- 

## Lambdaの適用例

- 月次処理でCSVデータからExcelファイルを生成
- [incanter](https://github.com/incanter/incanter)でデータ処理
- [mjul/docjure](https://github.com/mjul/docjure)でExcelファイルを生成
![sample](doc/img/2017-10-13-cigar-tax-item-details.png) |

---

## Lambdaのコスト
- 95ファイルを生成
- 先月の明細 |
![aws-bill](doc/img/aws-bill.png) |
- ひと月あたり、100万リクエスト、メモリ1GBで約111コンピューティング時間まで無料！（12ヶ月の無料利用枠期間終了後も！） |
    
---

## AWS Gateway

- API GatewayはAWS Cloud上でRESTfulエンドポイントを提供する手段
- CloudFrontと連携し、edge networkとcacheによる低レイテンシー 
- 複数バージョンAPIの同時稼働によるテストとデプロイの効率化 
- Cognito, OAuthと連携したセキュリティ 
- AWS API Gatewayコンソール | 

---
## clj/cljsをLambdaに対応させるアプローチ

- cljをAOTして、com.amazonaws.services.lambda.runtime.RequestHandlerの実装を生成し、Javaプラットフォームにデプロイ
- cljsをGoogle Closureコンパイラでjsに変換し、node.jsプラットフォームにデプロイ
- [Lambda上でのclj/cljsパフォーマンス比較](https://numergent.com/2016-01/AWS-Lambda-Clojure-and-ClojureScript.html)
- コールドインスタンス問題はCloud Watch Schedulerから定期的にリクエストを送ることで回避可能
- [AWS Lambdaの制限](http://docs.aws.amazon.com/ja_jp/lambda/latest/dg/limits.html)

---
## uswitch/Lambada 

- [uswitch/lambada](https://github.com/uswitch/lambada)
- com.amazonaws.services.lambda.runtime.RequestHandlerの実装を生成するマクロ

```clojure
(ns cigar-tax-report-aggregator.core
  (:require [clojure.data.json :as json]
            [clojure.java.io :as io]
            [uswitch.lambada.core :refer [deflambdafn]]))

(defn handle-event
  [event]
  (log/debug "Got the following event: " (pr-str event))
  ;; parse the event and dispatch based on the event
  {:status "ok"})

(deflambdafn cigar-tax-report-aggregator.core.LambdaFn
  [in out ctx]
  (try (let [event (json/read (io/reader in))
             res (handle-event event)]
         (with-open [w (io/writer out)]
           (json/write res w)))
       (catch Throwable e
         ;; send error to SNS topic
         (throw e))))
```

@[4]
@[12-20]
@[6-10]

- ctxは[Context](http://docs.aws.amazon.com/ja_jp/lambda/latest/dg/java-context-object.html)オブジェクト
- エラーはCloudWatchログに出力されるが、[Rollbar](https://rollbar.com/)などに出力したほうが管理しやすい

---

## Lambdaへのデプロイメント

- [mhjort/clj-lambda-utils](https://github.com/mhjort/clj-lambda-utils)
    - JarをS3にアップロードし、Lambdaにデプロイするソリューション。API Gatewayへのデプロイも可能。

```clojure
(defproject cigar-tax-report-generator "0.1.0-SNAPSHOT"
...
  :lambda {"dev" [{:handler "cigar-tax-report-aggregator.core.LambdaFn"
                   :memory-size 1024
                   :timeout 240
                   :function-name "aggregate-sales"
                   :region "ap-northeast-1"
                   :s3 {:bucket "dev.lambda-jars"
                        :object-key "cigar-tax-report-aggregator.jar"}}]
           "release" [{:handler "cigar-tax-report-aggregator.core.LambdaFn"
                   :memory-size 1024
                   :timeout 240
                   :function-name "aggregate-sales"
                   :region "ap-northeast-1"
                   :s3 {:bucket "lambda-jars"
                        :object-key "cigar-tax-report-aggregator.jar"}}]}
...
                :plugins [[com.jakemccrary/lein-test-refresh "0.15.0"]
                          [lein-clj-lambda "0.4.0"]]}}
```

@[3-9]
@[10-16]
@[19]

---

## cljsからAPI Gateway+Labmdaへデプロイ
- [nervous-systems/cljs-lambda](https://github.com/nervous-systems/cljs-lambda) 
    - cljsをAWS Lambdaにデプロイするソリューション。API Gatewayへのデプロイはなし 
- [nervous-systems/serverless-cljs-plugin](https://github.com/nervous-systems/serverless-cljs-plugin)
    - プラットフォームを抽象化し、AWS Lambda, Azure Functions, Google CloudFunctionsをサポートするServerlessにcljsをデプロイするプラグイン 
    - ServerlessはLambdaとAPI Gatewayまでの登録を一気に行う 

---

## serverless + serverless-cljs-plugin

- servlerss.yml

```yml
service: serverless-cljs

provider:
  name: aws
  runtime: nodejs6.10
  region: ap-northeast-1

functions:
  echo:
    cljs: serverless-cljs.core/echo
    events:
      - http:
          path: echo
          method: post

plugins:
  - serverless-cljs-plugin
```

---

- servless-cljs.core

```clojure
(ns serverless-cljs.core
  (:require [cljs-lambda.macros :refer-macros [defgateway]]))

(defn count-input [s]
  (count s))

(defgateway echo [event ctx]
  {:status  200
   :headers {:content-type (-> event :headers :content-type)}
   :body    (let [body (:body event)] 
              (str "body:" body " count:" (count-input body)))})
```

- デモ |
    - `serverless --profile default deploy`
    - `http POST https://l15ym7sje7.execute-api.ap-northeast-1.amazonaws.com/dev/echo body=hi`

---

## １エンドポイント毎に１関数を割り当てるのは面倒...

- [mhjort/ring-apigw-lambda-proxy](https://github.com/mhjort/ring-apigw-lambda-proxy)
    - Lambadaとringを連携させるring middleware
- まずは普通通りRing handlerを定義

```clojure
(ns lambda-api-demo.handler
  (:require [compojure.core :refer :all]
            [compojure.route :as route]
            [ring.middleware.defaults :refer [wrap-defaults api-defaults]]
            [ring.util.http-response :refer :all]
            [ring.middleware.json :refer [wrap-json-response]]))

(defroutes app-routes
  (GET "/hello" [name]
    (ok {:message (format "Hello World, %s" name)}))

  (GET "/error" []
    (bad-request {:message "Test error"}))

  (route/not-found
   (not-found {:message "Not Found"})))

(def app
  (-> app-routes
      (wrap-json-response)
      (wrap-defaults api-defaults)))

```

@[1-6]
@[8-16]
@[18-21]

---

- 次に、appをLambdaにマウント

```clojure
(ns lambda-api-demo.lambda
  (:require [uswitch.lambada.core :refer [deflambdafn]]
            [clojure.java.io :as io]
            [ring.middleware.apigw :refer [wrap-apigw-lambda-proxy]]
            [cheshire.core :as cheshire]
            [lambda-api-demo.handler :refer [app]]))

(def lambda-handler (wrap-apigw-lambda-proxy app {:scheduled-event-route "/warmup"}))

(deflambdafn lambda-api-demo.lambda.LambdaFn 
  [in out ctx]
  (with-open [writer (io/writer out)]
    (-> in
        (io/reader :encoding "UTF-8")
        (cheshire/parse-stream true)
        (lambda-handler)
        (cheshire/generate-stream writer))))
```

@[1-6]
@[8]
@[10-17]


---

## aws-lambdaでデプロイ

- 今度はlein-clj-lambdaプラグインの代わりに、lein-lambdaを使用してみる。
- [paulbutcher/lein-lambda](https://github.com/paulbutcher/lein-lambda)

```clojure
(defproject lambda-api-demo "0.1.0-SNAPSHOT"
...
  :dependencies [[org.clojure/clojure "1.8.0"]
                 [compojure "1.6.0"]
                 [metosin/ring-http-response "0.9.0"]
                 [ring/ring-defaults "0.3.1"]
                 [ring/ring-json "0.4.0"]
                 [uswitch/lambada "0.1.2"]
                 [cheshire "5.7.1"]
                 [ring-apigw-lambda-proxy "0.3.0"]]
  :plugins [[lein-ring "0.9.7"]
            [lein-lambda "0.2.0"]]
  :ring {:handler lambda-api-demo.handler/app}
  :profiles
  {:dev {:dependencies [[javax.servlet/servlet-api "2.5"]
                        [ring/ring-mock "0.3.1"]]}
   :uberjar {:aot :all}}
  :lambda {:credentials {:profile "default"} 
           :function {:name "lambda-api-demo"
                      :handler "lambda-api-demo.lambda.LambdaFn"}
           :api-gateway {:name "lambda-api-demo"}
           :stages {"production" {:warmup {:enable true}}
                    "staging"    {}}})

```
@[18-23]

    - `lein lambda deploy staging`
    - `http https://jmihnbxsf6.execute-api.ap-northeast-1.amazonaws.com/staging/hello name==Shibuya`
    - `http https://jmihnbxsf6.execute-api.ap-northeast-1.amazonaws.com/staging/error`

---

## 多機能にするとJarが肥大する...

- Lambdaの制限として、プログラムJarの上限は圧縮時50MB, 展開時250MB
    - Dependenciesにexclusionを入れて依存関係を静的に減らす。 |
    - 手作業で大変。Jar単位での制御では限界がある。 |

---

## portkey-cloud/portkey

- [portkey](https://github.com/portkey-cloud/portkey)は[O'reillyのClojure Programming](http://shop.oreilly.com/product/0636920013754.do)の共著者、Christophe Grandらによるプロジェクト。
- Apache Sparkクラスタ上でライブコーディングする[powderkeg](https://github.com/HCADatalab/powderkeg)から移植
- REPLから直接デプロイできる。
    - プラグインも、serverlessのようなデプロイ用のフレームワークも必要なし
    - データ => プログラム => デプロイ の流れがプログラムで実現可能 (Infrastructure as Code)
- 実行時にコードを動的に解析し、必要な依存関係のみを抽出し、動的にバイトコードを生成している。
    - 詳細は[スライド](https://github.com/portkey-cloud/portkey-clojutre-2017/blob/master/Portkey%20ClojuTRE%202017.pdf)と[動画](https://www.youtube.com/watch?v=qJXqQATJNTk&list=PLetHPRQvX4a9iZk-buMQfdxZm72UnP3C9&index=6)で。
- デモ |
- まだアルファクオリティ。Clojarにリリースされていない。 | 
- 既存アプリのRingハンドラを渡したら解析に8分以上 |

---

## その他

- [apex](http://apex.run/)
    - AWS Lambdaへのデプロイ管理をするフレームワーク
    - Node.jsから子プロセスを起動し、Go言語など、標準でサポートされていない言語でLambdaを記述可能
    - Clojureのサポートも入っているようだが、デモを稼働させることができず 😭

---

### ご清聴ありがとうございました

