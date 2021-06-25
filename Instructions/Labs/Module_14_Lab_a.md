---
lab:
    title: '14A: ステージング スロットを使用した Azure App Service Web アプリの実装'
    module: 'モジュール 14: アプリケーション インフラストラクチャの実装'
---

# ラボ: ステージング スロットを使用した Azure App Service Web アプリの実装
# 受講生用ラボ マニュアル

## ラボ シナリオ

Adatum Corporation には、比較的頻繁に更新される多数の Web アプリがあります。Adatum はまだ DevOps の原則を完全には採用していませんが、バージョン管理として Git に依存しており、アプリの更新を合理化するオプションを模索しています。Adatum は一部のワークロードを Azure に移行しているため、Adatum エンタープライズ アーキテクチャ チームは、この目的を達成するために Azure App Service とそのデプロイ スロットの使用を評価することを決定しました。 

デプロイ スロットは、固有のホスト名を持つライブ アプリです。アプリのコンテンツと構成要素は、運用スロットを含む 2 つのデプロイ スロットの間にスワップできます。アプリケーションを非運用スロットにデプロイすると、次の利点があります。

- 運用スロットでスワップする前に、ステージング デプロイ スロットでアプリの変更を確認することはできます。

- スロットにアプリをデプロイした後に運用サイトにスワップすると、運用サイトへのスワップ前にスロットのすべてのインスタンスが準備されます。これにより、アプリのデプロイ中のダウンタイムがなくなります。トラフィックのリダイレクションはシームレスであり、スワップ操作のためにドロップされる要求がありません。事前スワップ検証が不要な場合に、自動スワップを構成することでこのワークフローを自動化できます。

- スワップ後、以前にステージングされたアプリ搭載のスロットには、前の運用アプリがあります。運用スロットに入れ替えられた変更を元に戻す必要がある場合、最後の既知の良好な状態に戻すには、別のスワップを直ちに実行するだけです。

デプロイ スロットは、blue/green と A/B テストという 2 つの一般的なデプロイ パターンを支援します。lue-green のデプロイでは、ライブ アプリケーションとは別の運用環境に更新プログラムをデプロイします。デプロイが検証されると、トラフィック ルーティングが更新されたバージョンに切り替わります。A/B テストでは、新しいバージョンのアプリをテストするために、トラフィックの一部をステージング サイトに徐々にルーティングします。

Adatum Architecture チームは、次の 2 つのデプロイ パターンをテストするためにデプロイ スロットを使用した Azure App Service Web アプリを使用したいと考えています。

-  Blue/Green デプロイ 

-  A/B テスト 


## 目標
  
このラボを完了すると、次のことができるようになります。

-  Azure App Service Web アプリのデプロイ スロットを使用して Blue/Green デプロイ パターンを実装する

-  Azure App Service Web アプリのデプロイ スロットを使用して A/B テストを実行する


## ラボ環境
  
推定時間: 60 分


## ラボ ファイル

None

## 説明

### 演習 1: Azure App Service Web アプリを実装する

1. Azure App Service Web アプリをデプロイする

1. App Service Web アプリのデプロイ スロットを作成する

#### タスク 1: Azure App Service Web アプリをデプロイする

1. ラボのコンピューターから Web ブラウザーを起動し、 [Azure portal](https://portal.azure.com) に移動し、このラボで使用するサブスクリプションの所有者の役割を持つユーザーアカウントの認証情報を提供してサインインします。

1. Azure で、検索テキストボックスのすぐ右にあるツールバーアイコンを選択して、**Cloud Shell** ペインを表示します。

1. **Bash** または **PowerShell** のいずれかを選択するためのプロンプトが表示されたら、**「Bash」** を選択します。 

    >**注**: **Cloud Shell** を初めて起動し、「**ストレージがマウントされていません**」というメッセージが表示された場合は、このラボで使用しているサブスクリプションを選択し、「**ストレージの作成**」を選択します。 

1. Cloud Shell ペインから次のコマンドを実行して、**az30314a1** という名前の新しいディレクトリを作成し、それを現在のディレクトリとして設定します。

   ```sh
   mkdir az30314a1
   cd ~/az30314a1/
   ```

1. Cloud Shell ペインから次のコマンドを実行して、サンプル アプリ リポジトリを **az30314a1** ディレクトリにクローンします。

   ```sh
   REPO=https://github.com/Azure-Samples/html-docs-hello-world.git
   git clone $REPO
   cd html-docs-hello-world
   ```

1. Cloud Shell ペインから次のコマンドを実行して、デプロイ ユーザーを構成します。

   ```sh
   USERNAME=az30314user$RANDOM
   PASSWORD=az30314pass$RANDOM
   az webapp deployment user set --user-name $USERNAME --password $PASSWORD 
   echo $USERNAME
   echo $PASSWORD
   ```
1. デプロイ ユーザーが正常に作成されたことを確認します。競合を示すエラー メッセージが表示された場合は、前の手順を繰り返します。

    >**注**: ユーザー名の値と対応するパスワードを必ず記録してください。

1. Cloud Shell ペインから、次のコマンドを実行して、App Service Web アプリをホストするリソース グループを作成します (`<location>` プレースホルダーを、サブスクリプションで使用可能で、ラボ コンピューターの場所に最も近い Azure リージョンの名前に置き換えます)。

   ```sh
   LOCATION='<location>'
   RGNAME='az30314a-labRG'
   az group create --location $LOCATION --resource-group $RGNAME
   ```

1. Cloud Shell ペインから、次のコマンドを実行して、新しい App Service プランを作成します。

   ```sh
   SPNAME=az30314asp$LOCATION$RANDOM
   az appservice plan create --name $SPNAME --resource-group $RGNAME --location $LOCATION --sku S1
   ```

1. Cloud Shell ペインから、次のコマンドを実行して、Git 対応の新しい App Service Web アプリを作成します。

   ```sh
   WEBAPPNAME=az30314$RANDOM$RANDOM
   az webapp create --name $WEBAPPNAME --resource-group $RGNAME --plan $SPNAME --deployment-local-git
   ```

    >**注**: デプロイが完了するのを待ちます。 

1. Cloud Shell ペインから、次のコマンドを実行して、新しく作成された App Service Web アプリの公開 URL を取得します。

   ```sh
   URL=$(az webapp deployment list-publishing-credentials --name $WEBAPPNAME --resource-group $RGNAME --query scmUri --output tsv)
   ```

1. Cloud Shell ペインから、次のコマンドを実行して、Git 対応の Azure App Service Web アプリを表す git リモート エイリアスを設定します。

   ```sh
   git remote add azure $URL
   ```

1. Cloud Shell ペインから次のコマンドを実行して、git push azure master を使用して Azure リモートにプッシュします。

   ```sh
   git push azure master
   ```

    >**注**: デプロイが完了するのを待ちます。 

1. Cloud Shell ペインから、次のコマンドを実行して、新しくデプロイされた App Service Web アプリの FQDN を識別します。 

   ```sh
   az webapp show --name $WEBAPPNAME --resource-group $RGNAME --query defaultHostName --output tsv
   ```

1. 「Cloud Shell」 ペインを閉じます。


#### タスク 2: App Service Web アプリのデプロイ スロットを作成する

1. Azure portal で、**App Services** を検索して選択し、「**App Services**」 ブレードで、新しく作成した App Service Web アプリを選択します。

1. Azure portal で、新しくデプロイされた App Service Web アプリを表示するブレードに移動し、「**URL**」 リンクを選択して、「**Azure App Service - Sample Static HTML Site**」が表示されていることを確認します。ブラウザー タブは開いたままにしておきます。

1. App Service Web アプリ ブレードの 「**デプロイ**」 セクションで、「**デプロイ スロット**」 を選択してから、「**+ スロットの追加**」 を選択します。

1. 「**スロットの追加**」 ブレードで、次の設定を指定し、「**追加**」 を選択してから、「**閉じる**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | 名前 | **staging** |
    | 設定の複製先 | 設定を複製する |


### 演習 2: App Service Web アプリのデプロイ スロットを管理する
  
この演習の主なタスクは次のとおりです。

1. App Service Web アプリのステージング スロットに Web コンテンツをデプロイする

1. App Service Web アプリのステージング スロットをスワップする

1. A/B テストを構成する

1. ラボにデプロイした Azure リソースを削除する


#### タスク 1: App Service Web アプリのステージング スロットに Web コンテンツをデプロイする

1. Azure で、検索テキストボックスのすぐ右にあるツールバーアイコンを選択して、**Cloud Shell** ペインを表示します。

1. Cloud Shell ペインから次のコマンドを実行して、現在のセット **az30314a1/html-docs-hello-world** が現在のディレクトリであることを確認します。

   ```sh
   cd ~/az30314a1/html-docs-hello-world
   ```

1. 「Cloud Shell」 ウィンドウで次のコマンドを実行して、組み込みエディターを立ち上げます。

   ```sh
   code index.html
   ```
1. 「Cloud Shell」 ウィンドウのコード エディターで、次の行を置き換えます。

   ```html
   <h1>Azure App Service - Sample Static HTML Site</h1>
   ```

   置き換えると、次のようになります。

   ```html
   <h1>Azure App Service - Sample Static HTML Site v1.0.1</h1>
   ```

1. 変更を保存(**Ctrl + s**)し、エディター ウィンドウ(**Ctrl + q**)を閉じます。 

1. Cloud Shell ペインから次のコマンドを実行して、必要なグローバル git 構成設定を指定します。

   ```sh
   git config --global user.email "user@az30314.com"
   git config --global user.name "user az30314"
   ```

1. Cloud Shell ペインから次のコマンドを実行して、ローカルに適用した変更をマスター分岐にコミットします。

   ```sh
   git add index.html
   git commit -m 'v1.0.1'
   ```

1. Cloud Shell ペインから次のコマンドを実行して、App Service Web アプリの新しく作成されたステージング スロットの公開 URL を取得します。

   ```sh
   RGNAME='az30314a-labRG'
   WEBAPPNAME=$(az webapp list --resource-group $RGNAME --query "[?starts_with(name,'az30314')]".name --output tsv)
   SLOTNAME='staging'
   URLSTAGING=$(az webapp deployment list-publishing-credentials --name $WEBAPPNAME --slot $SLOTNAME --resource-group $RGNAME --query scmUri --output tsv)
   ```

1. Cloud Shell ペインから次のコマンドを実行して、Git 対応の Azure App Service Web アプリのステージング スロットを表す git リモート エイリアスを設定します。

   ```sh
   git remote add azure-staging $URLSTAGING
   ```

1. Cloud Shell ペインから次のコマンドを実行して、git push azure master を使用して Azure リモートにプッシュします。

   ```sh
   git push azure-staging master
   ```

    >**注**: デプロイが完了するのを待ちます。 

1. 「Cloud Shell」 ペインを閉じます。

1. Azure portal で、App Service Web アプリのデプロイ スロットを表示しているブレードに移動し、ステージング スロットを選択します。

1. ステージング スロットの概要を表示しているブレードで、 **URL** リンクをクリックして、表示されるページが変更されていることを確認します。


#### タスク 2: App Service Web アプリのステージング スロットをスワップする

1. Azure portal で、App Service Web アプリを表示しているブレードに戻り、「**デプロイ スロット**」 を選択します。

1. デプロイ スロット ブレードで、 「**スワップ**」 を選択します。

1. 「**スワップ**」 ブレードで、「**スワップ**」 を選択し、ソースとターゲットのスロットを入れ替えます。「**閉じる**」 を選択します。

1. App Service Web アプリが表示されているブラウザー タブに切り替えて、ブラウザーの画面を更新します。ステージング スロットにデプロイした変更が表示されることを確認します。

1. App Service Web アプリのステージング スロットを示すブラウザー タブに切り替え、ブラウザーの画面を更新します。元のデプロイメントに含まれる元の Web ページが表示されることを確認します。 


#### タスク 3: A/B テストを構成する

1. Azure portal で、App Service Web アプリの運用スロットを表示しているブレードに戻ります。

1. Web アプリの運用スロットページで **デプロイ スロット** を選択し、ステージング スロットを表示している行の 「**トラフィック ％**」 列の値を 50 に設定します。これにより、本番スロットを表す行の **トラフィック ％** の値が自動的に 50 に設定されます。

1. デプロイ スロットを表示しているブレードで、**保存**を選択します。 

1. Azure で、検索テキストボックスのすぐ右にあるツールバーアイコンを選択して、**Cloud Shell** ペインを表示します。

1. Cloud Shell ペインから次のコマンドを実行して、ターゲット Web アプリの名前を変数に設定します。

   ```sh
   RGNAME='az30314a-labRG'
   WEBAPPNAME=$(az webapp list --resource-group $RGNAME --query "[?starts_with(name,'az30314')]".name --output tsv)
   ```

1. Cloud Shell ペインから次を数回実行して、2 つのスロット間のトラフィック分布を特定します。

   ```sh
   curl -H 'Cache-Control: no-cache' https://$WEBAPPNAME.azurewebsites.net --stderr - | grep '<h1>Azure App Service - Sample Static HTML Site'
   ```

    >**注**: トラフィック分布は完全に確定的ではありませんが、各目標サイトからいくつかの応答が表示されるはずです。

#### タスク 4: ラボにデプロイした Azure リソースを削除する

1. 「Cloud Shell」 ウィンドウから次のコマンドを実行して、この演習で作成したリソース グループを一覧表示します。

   ```sh
   az group list --query "[?starts_with(name,'az30314')]".name --output tsv
   ```

    > **注**: このラボで作成したリソース グループのみが出力に含まれていることを確認します。このグループは、このタスクで削除されます。

1. 「Cloud Shell」 ウィンドウから次を実行して、このラボで作成したリソース グループを削除します

   ```sh
   az group list --query "[?starts_with(name,'az30314')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

1. 「Cloud Shell」ペインから次を実行して、**az30314a1** ディレクトリを削除します。

   ```sh
   rm -r -f ~/az30314a1
   ```
   
1. 「Cloud Shell」 ペインを閉じます。
