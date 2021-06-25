---
lab:
    title: '14B: メッセージベースの統合アーキテクチャの構成'
    module: 'モジュール 14: アプリケーション インフラストラクチャの実装'
---

# ラボ: メッセージベースの統合アーキテクチャの構成

# 受講生用ラボ マニュアル

## ラボ シナリオ

Adatum Corporation には、オンプレミスのファイル サーバーに定期的にアップロードされるファイルを処理するいくつかの Web アプリケーションがあります。ファイルのサイズはさまざまですが、最大 100 MB まで可能です。Adatum は、アプリケーションを Azure App Service または Azure Functions ベースのアプリに移行し、Azure Storage を使用してアップロードされたファイルのホストを検討しています。次の 2 つのシナリオをテストする予定です。

- Azure Functions を通して、Azure Storage コンテナーにアップロードされた新しい Blob を自動的に処理します。

  - Event Grid を使用して、Azure Storage コンテナーにアップロードされた新しい Blob を参照する Azure Storage キューメッセージを生成します。

これらのシナリオは、大きなメッセージを送信、受信、および操作するときに、メッセージングベースのアーキテクチャに共通の課題に対処することを目的としています。大きなメッセージをメッセージキューに直接送信することはお勧めしません。使用するリソースと消費する帯域幅が大きくなるからです。加えて、メッセージングプラットフォームは通常、大量の小さなメッセージを処理するように微調整されているため、処理速度に悪影響を与えてしまいます。さらに、メッセージングプラットフォームは通常、処理できる最大メッセージサイズを制限しています。

考えられるソリューションの 1 つは、Azure Blob Store、Azure SQL、Azure Cosmos DB などの外部サービスにメッセージペイロードを保存し、保存されたペイロードへの参照を取得してから、その参照だけをメッセージバスに送信する方法です。このアーキテクチャ パターンは、「クレームチェック」として知られています。その特定のメッセージの処理に関心があるクライアントは、必要に応じて、取得した参照を使ってペイロードを取得できます。

Azure では、このパターンはさまざまな方法により、さまざまなテクノロジーを使用して実装できますが、通常、クレーム チェックの生成を自動化し、メッセージ バスにプッシュしてクライアントが使用するか、ペイロード処理を直接トリガーするイベントに依存します。このような実装に含まれる一般的なコンポーネントの 1 つは Event Grid です。Event Grid は、構成可能な期間 (最大 24 時間) 以内にイベントを配信するイベント ルーティング サービスです。その後、イベントは破棄されるか配信不能にされます。イベントコンテンツのアーカイブまたはイベントストリームの再生が必要な場合は、Event Hub への Event Grid サブスクリプションまたはメッセージを長期間保持できる Azure Storage のキューをセットアップすることにより、この要件を容易にすることができます。

このラボでは、Azure Storage Blob サービスを使用して、処理するファイルを保存します。クライアントは、指定された Azure Blob コンテナーに共有するファイルをドロップするだけです。最初のエクササイズでは、ファイルは Azure Function によって直接消費され、サーバーレスの性質を活用します。また、Application Insights を利用してインストルメンテーションを提供し、ファイル処理の監視と分析を促進します。2番目のエクササイズでは、 Event Grid を使用して、リクエストチェックメッセージを自動的に生成し、Azure Storage キューに送信します。これにより、クライアントアプリケーションはキューをポーリングます。メッセージを取得してから、格納された参照データを使用して、Azure Blob Storage から直接ペイロードをダウンロードできます。

最初のエクササイズの Azure Function が Blob Storage トリガーに依存していることに注意するべきです。次の要件を処理する場合は、 Blob ストレージトリガーではなく Event Grid トリガーを選択する必要があります。

- Blob 専用ストレージアカウント： Blob ストレージアカウントは、 Blob の入力および出力のバインディングでサポートされていますが、 Blob トリガーでサポートされていません。Blob Storage のトリガーには、汎用のストレージ アカウントが必要です。

  - 高スケール。高スケールは、100,000 を超える Blob が含まれるコンテナー、または1秒あたり100を超える Blob 更新があるストレージアカウントとして大まかに定義できます。

  - 待機時間の短縮。機能アプリが消費計画にあった場合、機能アプリがアイドル状態になっていると、新しい Blob の処理に最大10分の遅延が生じる可能性があります。この遅延を回避するには、 Event Grid トリガーを使用するか、常にオンの設定を有効にして App Service プランに切り替えてください。

  - Blob 削除イベントの処理： Blob 削除イベントは、Blob ストレージトリガーではサポートされていません。

## 目標
  
このラボを完了すると、次のことができるようになります。

- Azure Function App Storage Blob トリガーを構成および検証

- Azure Event Grid サブスクリプションベースのキューメッセージングを構成および検証

## ラボ環境
  
推定時間: 60 分

## ラボ ファイル

None

## 説明

### 演習 1: Azure Function App Storage Blob トリガーを構成および検証
  
この演習の主なタスクは次のとおりです。

1. Azure Function App Storage Blob トリガーの構成

1. Azure Function App Storage Blob トリガーの検証

#### タスク 1: Azure Function App Storage Blob トリガーの構成

1. ラボのコンピューターから Web ブラウザーを起動し、 [Azure portal](https://portal.azure.com) に移動し、このラボで使用するサブスクリプションの所有者の役割を持つユーザーアカウントの認証情報を提供してサインインします。

1. Azure で、検索テキストボックスのすぐ右にあるツールバーアイコンを選択して、**Cloud Shell** ペインを表示します。

1. **Bash** または **PowerShell** のいずれかを選択するためのプロンプトが表示されたら、**「Bash」** を選択します。

    > **注**: **Cloud Shell** を初めて起動し、「**ストレージがマウントされていません**」というメッセージが表示された場合は、このラボで使用しているサブスクリプションを選択し、「**ストレージの作成**」を選択します。

1. Cloud Shell ペインから次を実行して、このエクササイズで指定するリソースの名前のプレフィックスとして使用される擬似ランダムな文字列を生成します。

   ```sh
   export PREFIX=$(echo `openssl rand -base64 5 | cut -c1-7 | tr '[:upper:]' '[:lower:]' | tr -cd '[[:alnum:]]._-'`)
   ```

1. Cloud Shell ペインから、次の操作を実行して、このラボでリソースをプロビジョニングする Azure リージョンを指定します (`<Azure region>`プレースホルダーを、サブスクリプションで使用できる、ラボの場所に最も近い Azure リージョンの名前に置き換えます)。

   ```sh
   export LOCATION='<Azure region>'
   ```

1. Cloud Shell ペインから次を実行して、このラボでプロビジョニングするすべてのリソースをホストするリソースグループを作成します。

   ```sh
   export RESOURCE_GROUP_NAME='az30314b-labRG'

   az group create --name "${RESOURCE_GROUP_NAME}" --location "$LOCATION"
   ```

1. Cloud Shell ペインから、次を実行して、Azure ストレージ アカウントと、Azure 関数によって処理される Blob をホストするコンテナーを作成します。

   ```sh
   export STORAGE_ACCOUNT_NAME="az30314b${PREFIX}"

   export CONTAINER_NAME="workitems"

   export STORAGE_ACCOUNT=$(az storage account create --name "${STORAGE_ACCOUNT_NAME}" --kind "StorageV2" --location "${LOCATION}" --resource-group "${RESOURCE_GROUP_NAME}" --sku "Standard_LRS")
   ```

    > **注**: 同じストレージアカウントは、独自の処理要件を容易にするために Azure 機能でも使用されます。実際のシナリオでは、この目的のために別のストレージアカウントを作成することを検討する必要があります。

1. Cloud Shell ペインから、次を実行して、Azure Storage アカウントの接続文字列 プロパティの値を格納する変数を作成します。

   ```sh
   export STORAGE_CONNECTION_STRING=$(az storage account show-connection-string --name "${STORAGE_ACCOUNT_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" -o tsv)
   ```

1. Cloud Shell ペインから、次を実行して、Azure 関数によって監視される Blob をホストするコンテナーを作成します。

   ```sh
   az storage container create --name "${CONTAINER_NAME}" --account-name "${STORAGE_ACCOUNT_NAME}" --connection-string "${STORAGE_CONNECTION_STRING}"
   ```

1. Cloud Shell ペインから次のコマンドを実行して、Azure Function によって監視するためのApplication Insights を作成し、そのキーを変数に保存します。

   ```sh
   export APPLICATION_INSIGHTS_NAME="az30314bi${PREFIX}"

   az resource create --name "${APPLICATION_INSIGHTS_NAME}" --location "${LOCATION}" --properties '{"Application_Type": "other", "ApplicationId": "function", "Flow_Type": "Redfield"}' --resource-group "${RESOURCE_GROUP_NAME}" --resource-type "Microsoft.Insights/components"

   export APPINSIGHTS_KEY=$(az resource show --name "${APPLICATION_INSIGHTS_NAME}" --query "properties.InstrumentationKey" --resource-group "${RESOURCE_GROUP_NAME}" --resource-type "Microsoft.Insights/components" -o tsv)
   ```

1. Cloud Shell ペインから、次を実行して、Azure Storage Blob の作成イベントを監視し、処理する Azure Function を作成します。

   ```sh
   export FUNCTION_NAME="az30314f${PREFIX}"

   az functionapp create --name "${FUNCTION_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --app-insights "$APPLICATION_INSIGHTS_NAME" --app-insights-key "$APPINSIGHTS_KEY" --storage-account "${STORAGE_ACCOUNT_NAME}" --consumption-plan-location "${LOCATION}" --runtime "dotnet" --functions-version 2
   ```

1. Cloud Shell ペインから、次を実行して、新しく作成された関数アプリに Application Insights および Azure Storage アカウントにリンクするための環境変数を定義します。

   ```sh
   az functionapp config appsettings set --name "${FUNCTION_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --settings "APPINSIGHTS_INSTRUMENTATIONKEY=$APPINSIGHTS_KEY" FUNCTIONS_EXTENSION_VERSION=~2

   az functionapp config appsettings set --name "${FUNCTION_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --settings "STORAGE_CONNECTION_STRING=$STORAGE_CONNECTION_STRING" FUNCTIONS_EXTENSION_VERSION=~2
   ```

1. Azure ポータルに切り替えて、このタスクの前半で作成した Azure 関数 アプリのブレードに移動します。

1. Azure 関数アプリ ブレードで、「**関数**」 をクリックし、「**+ 追加**」 をクリックします。

1. **関数の追加**ブレードで、**テンプレートの選択** から「**Azure Blob Storage Trigger**」 テンプレートを選択します。

1. 下にスクロールして**テンプレートの詳細**セクションをで以下を指定し、「**追加**」 を選択して Azure 関数内に新しい関数 を作成します。

    | 設定 | 値 |
    | --- | --- |
    | 新しい関数 | **BlobTrigger** |
    | Path | **workitems/{name}** |
    | Storage Account connection | **STORAGE_CONNECTION_STRING** |

1. Azure 関数アプリの **BlobTrigger**関数 をクリックして開き、「**コードとテスト**」 を選択し、run.csx ファイルのコンテンツを確認します。

   ```csharp
   public static void Run(Stream myBlob, string name, ILogger log)
   {
       log.LogInformation($"C# Blob trigger function Processed blob\n Name:{name} \n Size: {myBlob.Length} Bytes");
   }
   ```

    > **注**: デフォルトでは、この関数は、新しい Blob の作成に対応するイベントを単純に記録するように構成されています。追加の Blob 処理タスクを実行するには、このファイルのコンテンツを変更する必要があります。

#### タスク 2: Azure Function App Storage Blob トリガーの検証

1. 必要に応じて、 Cloud Shell で Bash セッションを再起動します。

1. Cloud Shell ペインから次を実行して、前のタスクで使用した変数を再入力します。

   ```sh
   export RESOURCE_GROUP_NAME='az30314b-labRG'

   export STORAGE_ACCOUNT_NAME="$(az storage account list --resource-group "${RESOURCE_GROUP_NAME}" --query "[0].name" --output tsv)"

   export CONTAINER_NAME="workitems"
   ```

1. Cloud Shell ペインから次を実行して、このタスクで以前に作成した Azure ストレージアカウントにテスト Blob をアップロードします。

   ```sh
   export STORAGE_ACCESS_KEY="$(az storage account keys list --account-name "${STORAGE_ACCOUNT_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --query "[0].value" --output tsv)"

   export WORKITEM='workitem1.txt'

   touch "${WORKITEM}"

   az storage blob upload --file "${WORKITEM}" --container-name "${CONTAINER_NAME}" --name "${WORKITEM}" --auth-mode key --account-key "${STORAGE_ACCESS_KEY}" --account-name "${STORAGE_ACCOUNT_NAME}"
   ```

1. Azure ポータルで、前のタスクで作成した Azure Function アプリが表示するブレードに戻ります。

1. 関数アプリで、「**関数**」をクリックします。

1. 関数の一覧で、**BlobTrigger** をクリックします。

1. 「BlobTrigger」ブレードで、「**モニター**」エントリをクリックします。

1. Blob のアップロードを表す単一のイベントエントリに注意してください。エントリを選択して、「**Invocation Details**」の詳細を表示します。

    > **注**: このエクササイズの Azure Function アプリは消費計画で実行されるため、 Blob のアップロードとトリガーされる関数の間に最大数分の遅延が生じる場合があります。App Service（Consumption ではなく）プランを使用して Function アプリを実装することにより、待ち時間を最小限に抑えることができます。

1. 「**Invocation Details**」で「**Application Insights」の「クエリの実行**」をクリックします。

1. Application Insights ポータルで、自動生成された Kusto クエリとその結果を確認します。

### 演習 2: Azure Event Grid サブスクリプションベースのキューメッセージングを構成および検証
  
この演習の主なタスクは次のとおりです。

1. Azure Event Grid サブスクリプションベースのキューメッセージングを構成

1. Azure Event Grid サブスクリプションベースのキューメッセージングを検証

1. ラボにデプロイした Azure リソースを削除する

#### タスク 1: Azure Event Grid サブスクリプションベースのキューメッセージングを構成

1. 必要に応じて、 Cloud Shell で Bash セッションを再起動します。

1. Cloud Shell ペインから、次の操作を実行して、サブスクリプションに eventgrid リソース プロバイダーを登録します。

   ```sh
   az provider register --namespace microsoft.eventgrid
   ```
  
1. Cloud Shell ペインから次を実行して、このエクササイズで指定するリソースの名前のプレフィックスとして使用される擬似ランダムな文字列を生成します。

   ```sh
   export PREFIX=$(echo `openssl rand -base64 5 | cut -c1-7 | tr '[:upper:]' '[:lower:]' | tr -cd '[[:alnum:]]._-'`)
   ```

1. Cloud Shell ペインから次を実行して、リソースグループとその Azure リージョンを特定します。

   ```sh
   export RESOURCE_GROUP_NAME_EXISTING='az30314b-labRG'

   export LOCATION=$(az group list --query "[?name == '${RESOURCE_GROUP_NAME_EXISTING}'].location" --output tsv)

   export RESOURCE_GROUP_NAME='az30314c-labRG'

   az group create --name "${RESOURCE_GROUP_NAME}" --location $LOCATION
   ```

1. Cloud Shell ペインから、次を実行して、このタスクで構成する Event Grid サブスクリプションで使用されるコンテナーをホストする Azure ストレージ アカウントを作成します。

   ```sh
   export STORAGE_ACCOUNT_NAME="az30314cst${PREFIX}"

   export CONTAINER_NAME="workitems"

   export STORAGE_ACCOUNT=$(az storage account create --name "${STORAGE_ACCOUNT_NAME}" --kind "StorageV2" --location "${LOCATION}" --resource-group "${RESOURCE_GROUP_NAME}" --sku "Standard_LRS")
   ```

1. Cloud Shell ペインから、次を実行して、Azure Storage アカウントの接続文字列 プロパティの値を格納する変数を作成します。

   ```sh
   export STORAGE_CONNECTION_STRING=$(az storage account show-connection-string --name "${STORAGE_ACCOUNT_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" -o tsv)
   ```

1. Cloud Shell ペインから、次を実行して、Event Grid サブスクリプションで使用するコンテナーを作成します。

   ```sh
   az storage container create --name "${CONTAINER_NAME}" --account-name "${STORAGE_ACCOUNT_NAME}" --connection-string "${STORAGE_CONNECTION_STRING}"
   ```

1. Cloud Shell ペインから、次を実行して、Azure Storage アカウントの Resource Id プロパティの値を格納する変数を作成します。

   ```sh
   export STORAGE_ACCOUNT_ID=$(az storage account show --name "${STORAGE_ACCOUNT_NAME}" --query "id" --resource-group "${RESOURCE_GROUP_NAME}" -o tsv)
   ```

1. Cloud Shell ペインから次を実行して、このタスクで構成する Event Grid サブスクリプションによって生成されたメッセージを保存するストレージアカウントキューを作成します。

   ```sh
   export QUEUE_NAME="az30314cq${PREFIX}"

   az storage queue create --name "${QUEUE_NAME}" --account-name "${STORAGE_ACCOUNT_NAME}" --connection-string "${STORAGE_CONNECTION_STRING}"
   ```

1. Cloud Shell ペインから、次を実行して、Azure Storage アカウントの指定されたコンテナーへの Blob アップロードに応じて Azure Storage キューでメッセージを生成する Event Grid サブスクリプションを作成します。

   ```sh
   export QUEUE_SUBSCRIPTION_NAME="az30314cqsub${PREFIX}"

   az eventgrid event-subscription create --name "${QUEUE_SUBSCRIPTION_NAME}" --included-event-types 'Microsoft.Storage.BlobCreated' --endpoint "${STORAGE_ACCOUNT_ID}/queueservices/default/queues/${QUEUE_NAME}" --endpoint-type "storagequeue" --source-resource-id "${STORAGE_ACCOUNT_ID}"
   ```

#### タスク 2: Azure Event Grid サブスクリプションベースのキューメッセージングを検証

1. Cloud Shell ペインから次を実行して、このタスクで以前に作成した Azure ストレージアカウントにテスト Blob をアップロードします。

   ```sh
   export AZURE_STORAGE_ACCESS_KEY="$(az storage account keys list --account-name "${STORAGE_ACCOUNT_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --query "[0].value" --output tsv)"

   export WORKITEM='workitem2.txt'

   touch "${WORKITEM}"

   az storage blob upload --file "${WORKITEM}" --container-name "${CONTAINER_NAME}" --name "${WORKITEM}" --auth-mode key --account-key "${AZURE_STORAGE_ACCESS_KEY}" --account-name "${STORAGE_ACCOUNT_NAME}"
   ```

1. Azure ポータルで、このエクササイズの前のタスクで作成した Azure ストレージアカウントが表示されているブレードに移動します。

1. Azure ストレージ アカウントのブレードで、「**キュー**」 を選択して、キューのリストを表示します。

1. この演習の前のタスクで作成したキューを表すエントリを選択します。

1. キューには１つのメッセージが含まれていることに注意してください。エントリをクリックして、**メッセージ プロパティ**ブレードを表示します。

1. **メッセージ本文** では、前のタスクでアップロードした Azure Storage Blob の URL を表す **URL** プロパティの値に注意してください。

#### タスク 3: ラボにデプロイした Azure リソースを削除する

1. 「Cloud Shell」 ウィンドウから次のコマンドを実行して、この演習で作成したリソース グループを一覧表示します。

   ```sh
   az group list --query "[?starts_with(name,'az30314')]".name --output tsv
   ```

    > **注**: このラボで作成したリソース グループのみが出力に含まれていることを確認します。このグループは、このタスクで削除されます。

1. 「Cloud Shell」 ウィンドウから次を実行して、このラボで作成したリソース グループを削除します

   ```sh
   az group list --query "[?starts_with(name,'az30314')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

1. 「Cloud Shell」 ペインを閉じます。
