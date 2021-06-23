---
lab:
    title: '05: 高可用性 Azure IaaS コンピューティング アーキテクチャの実装'
    module: 'モジュール 05: 負荷分散とネットワーク セキュリティの実装'
---

# ラボ: 高可用性 Azure IaaS コンピューティング アーキテクチャの実装
# 受講生用ラボ マニュアル

## ラボ シナリオ
  
Adatum Corporation には、物理サーバーと仮想マシンが混在して実行されているオンプレミスのワークロードがいくつかあります。ほとんどのワークロードには、高可用性 SLA の範囲を含む、ある程度の回復性が必要です。ほとんどのワークロードは、クラスターノード間の同期レプリケーションで、Windows サーバー フェールオーバー クラスタリングまたは Linux Corosyncクラスターと Pacemaker Resource Manager のいずれかを利用します。Adatumは、同等の機能を Azure に実装する方法を決定しようとしています。特に、Adatum エンタープライズ アーキテクチャ チームは、同じデータ センター内および同じリージョンのデータセンター間の高可用性要件に対応する Azure プラットフォームの機能を調査しています。

さらに、Adatum エンタープライズ アーキテクチャ チームは、回復性だけではビジネス オペレーションで期待されるレベルの可用性を提供するには不十分である可能性があることを認識しています。一部のワークロードには、非常に動的な使用パターンがあり、現在、継続的な監視とカスタム スクリプト ソリューションに基づいて対処されており、追加のクラスターノードのプロビジョニングとプロビジョニング解除が自動的に行われます。他のユーザーの使用パターンはより予測可能ですが、ディスク スペース、メモリ、または処理リソースに対する需要の増加に対応するために、時々調整する必要があります。

これらの目標を達成するために、アーキテクチャ チームは、次のような高可用性 IaaS コンピューティング デプロイの範囲をテストしたいと考えています。

-  Azure Load Balancer Basic の背後にある Azure VM の可用性セット ベースのデプロイ

-  Azure Load Balancer Standard の背後にある Azure VM のゾーン冗長デプロイ

-  Azure Application Gateway の背後にある Azure VM スケール セットのゾーン冗長デプロイ

-  Azure VM Scale Sets の自動水平スケーリング (自動スケーリング) 

-  Azure VM Scale Sets の手動バーティカル スケーリング (計算とストレージ)

可用性セットは、同じ Azure データセンター内の物理的な配置を制御する Azure VM の論理グループを表します。Azure は、可用性セット内に配置する VM が複数の物理サーバー、コンピューティング ラック、ストレージ ユニット、ネットワーク スイッチで実行されるようにします。ハードウェアまたはソフトウェアの障害が発生した場合、影響を受けるのは VM のサブセットだけであり、ソリューション全体は動作状態を維持します。可用性セットは、信頼性の高いクラウド ソリューションの構築に不可欠です。可用性セットにより、Azure は 99.95％ の VM アップタイム SLA を提供します。

Availability Zones は、単一の Azure リージョン内の一意の物理的な場所を表します。それぞれのゾーンは、独立した電源、冷却手段、ネットワークを備えた 1 つまたは複数のデータセンターで構成されています。可用性ゾーンはリージョン内で物理的に分離されているため、データセンターで障害が発生してもアプリケーションとデータは保護されます。ゾーン冗長サービスによってアプリケーションとデータが可用性ゾーン全体にレプリケートされ、単一障害点から保護されます。可用性セットにより、Azure は 99.99％ の VM アップタイム SLA を提供します。

Azure Virtual Machine Scale Sets では、負荷分散が行われる同一の VM のグループを作成して管理できます。需要または定義されたスケジュールに応じて、VM インスタンスの数を自動的に増減させることができます。スケール セットは、アプリケーションの高可用性を実現します。また、多数の VM の一元的な管理、構成、更新を可能にします。仮想マシン スケール セットを使用すると、コンピューティング、ビッグ データ、コンテナー ワークロードなどの分野で大規模なサービスを構築できます。

## 目標
  
このラボを完了すると、次のことができるようになります。

-  Azure Load Balancer Basic の背後にある同じ可用性セットにある高可用性 Azure VM の特性を説明する

-  Azure Load Balancer Standard の背後にあるさまざまな Availability Zones に存在する高可用性 Azure VM の特性を説明する

-  Azure VM Scale Sets の自動の水平スケーリングの特性を説明する

-  Azure VM Scale Sets の手動のバーティカル スケーリングの特性を説明する


## ラボ環境
  
Windows Server 管理者の資格情報

-  User Name (ユーザー名): **Student**

-  パスワード: **Pa55w.rd1234**

推定時間: 120 分


## ラボ ファイル

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305suba.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rga.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rga.parameters.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgb.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgb.parameters.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgc.json

-  \\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgc.parameters.json

-  \\AZ303\\AllFiles\\Labs\\05\\az30305e-configure_VMSS_with_data_disk.ps1


## 説明

### 演習 1: 可用性セットと Azure Load Balancer Basic を使用して高可用性 Azure VM デプロイメントを実装および分析する
  
この演習の主なタスクは次のとおりです。

1. Azure Resource Manager テンプレートを使用して、高可用性 Azure VM を Azure Load Balancer Basic の背後にある可用性セットにデプロイする

1. Azure Load Balancer Basic の背後にある可用性セットにデプロイされた高可用性 Azure VM を分析する

1. 演習でデプロイされた Azure リソースを削除する


#### タスク 1: Azure Resource Manager テンプレートを使用して、高可用性 Azure VM を Azure Load Balancer Basic の背後にある可用性セットにデプロイする

1. ラボのコンピューターから Web ブラウザーを起動し、 [Azure portal](https://portal.azure.com) に移動し、このラボで使用するサブスクリプションの所有者の役割を持つユーザーアカウントの認証情報を提供してサインインします。

1. Azure portal で、検索テキストボックスのすぐ右にあるツールバー アイコンを選択して、「**Cloud Shell**」ペインを表示します。

1. **Bash** または **PowerShell** のいずれかを選択するためのプロンプトが表示されたら、**「Bash」** を選択します。 

    > **注**: **Cloud Shell** を初めて起動し、「**ストレージがマウントされていません**」というメッセージが表示された場合は、このラボで使用しているサブスクリプションを選択し、「**ストレージの作成**」を選択します。 
    
1. Cloud Shell ペインから、以下を実行して、このラボでの今後の演習に備えて Microsoft.Insights リソース プロバイダーを登録します。

   ```Bash
   az provider register --namespace 'Microsoft.Insights'
   ```

1. Cloud Shell ペインのツールバーで、「**ファイルのアップロード/ダウンロード**」アイコンを選択し、ドロップダウン メニューで「**アップロード**」を選択して、ファイル **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305suba.json** を Cloud Shell ホーム ディレクトリにアップロードします。

1. Cloud Shell ペインから、以下を実行して、このラボで使用する Azure リージョンを指定します (`<Azure region>` プレースホルダーを、サブスクリプションで Azure VM の展開に使用でき、ラボ コンピューターの場所に最も近い Azure リージョンの名前に置き換えます ※ **astus** をお勧めします)：

   ```Bash
   LOCATION='<Azure region>'
   ```
   
      > **注記**: Azure VM をプロビジョニングできる Azure リージョンを識別するには、[**https://azure.microsoft.com/ja-jp/regions/offers/**](https://azure.microsoft.com/ja-jp/regions/offers/) を参照してください。

      > **注記**: **LOCATION** 変数の値を設定するときに使用する Azure リージョンの名前を特定するには、`az account list-locations --query "[].{name:name}" -o table` を実行します。スペースを含まない表記を使用してください (例: **US East** ではなく **eastus**)。

1. Cloud Shell ペインから、以下を実行して、このラボでの今後の演習に備えて Network Watcher のインスタンスを作成します。

   ```Bash
   az network watcher configure --resource-group NetworkWatcherRG --locations $LOCATION --enabled -o table
   ```

1. Cloud Shell ペインから、以下を実行して、指定された Azure リージョンにリソース グループを作成します。
 
   ```Bash
   az deployment sub create \
   --location $LOCATION \
   --template-file azuredeploy30305suba.json \
   --parameters rgName=az30305a-labRG rgLocation=$LOCATION
   ```
      
1. Cloud Shell ペインから、Azure Resource Manager テンプレート **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rga.json** をアップロードします。

1. Cloud Shell ペインから、Azure Resource Manager パラメーター ファイル **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rga.parameters.json** をアップロードします。

1. Cloud Shell ペインから次のコマンドを実行して、Windows Server 2019 Datacenter Core をホストする Azure VM のペアで構成されるバックエンド プールを備えた Azure Load Balancer Basic を同じ可用性セットにデプロイします。

   ```Bash
   az deployment group create \
   --resource-group az30305a-labRG \
   --template-file azuredeploy30305rga.json \
   --parameters @azuredeploy30305rga.parameters.json
   ```

    > **注**: 次のタスクに進む前に、デプロイが完了するまで待機します。これにはおよそ 10 分かかります。

1. Azure portal で、**Cloud Shell** ウィンドウを閉じます。 


#### タスク 2: Azure Load Balancer Basic の背後にある可用性セットにデプロイされた高可用性 Azure VM を分析する

1. Azure portal で、「**Network Watcher**」を検索して選択し、「**Network Watcher**」ブレードで、 「**トポロジ**」 を選択します。

1. 「**Network Watcher \| トポロジ**」ブレードで、次の設定を指定します。

    | 設定 | 値 | 
    | --- | --- |
    | 定期売買 | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30305a-labRG** |
    | 仮想ネットワーク | **az30305a-vnet** |

1. トポロジ図を確認し、パブリック IP アドレス、ロードバランサー、およびバックエンドプール内の Azure VM のネットワークアダプター間の接続に注意を払います。

1. 「**Network Watcher**」ブレードで、「**有効なセキュリティルール**」 を選択します。

1. 「**Network Watcher** \| 「**有効なセキュリティルール**」 ブレードで、次の設定を指定します。

    | 設定 | 値 | 
    | --- | --- |
    | 定期売買 | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30305a-labRG** |
    | 仮想マシン | **az30305aが-VM0** |
    | ネットワーク インターフェイス | **az30305a-nic0** |
    
    > **注** NIC が表示されない場合は、一度別の VM1 を選択し、再度 VM0 を選択しなおしてみてください。

1. 関連するネットワーク セキュリティ グループと、RDP および HTTP を介した受信接続を許可する 2 つのカスタム ルールを含む有効なセキュリティ ルールを確認します。  

    > **注**: 「**有効なセキュリティ ルール**」 は、
    - **az30305a-nic0** の「ネットワーク インターフェイス」ブレードから表示することもできます。
    - **az30305a-web-nsg** の「ネットワーク セキュリティ グループ」ブレード 
    
1. **Network Watcher** ブレードで、「**接続の問題解決**」 を選択します。

    > **注**: 同じ可用性セット内の 2 つの Azure VM の近接性 (ネットワークの観点から) を確認することが目的です。

1. 「**Network Watcher**」 \| 「**接続のトラブルシューティング**」 ブレードで、次の設定を指定して、「**確認**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | 定期売買 | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30305a-labRG** |
    | ソース タイプ | **仮想マシン** |
    | 仮想マシン | **az30305a-VM0** |
    | 宛先 | **仮想マシンの選択** |
    | リソース グループ | **az30305a-labRG** |
    | 仮想マシン | **az30305a-vm1** |
    | プロトコル | **TCP** |
    | Destination port (依存先ポート)| **80** |

    > **注**: **Azure Network Watcher エージェント** VM 拡張機能を Azure VM にインストールするには数分かかります。

1. 結果を確認し、Azure VM 間のネットワーク接続の遅延を確認します。

    > **注**: 両方の VM が同じ可用性セット内（同じ Azureデータセンター内）にあるため、遅延時間は 1 ミリ秒程度のはずです。

1. Azureポータルで、 **az30305a-labRG** リソース グループ ブレードのリソースリストで、 **az30305a-avset** 可用性セット エントリを選択し、**az30305a-avset** ブレードで、2 つの Azure VM に割り当てられた障害ドメインと更新ドメインの値を確認します。

1. Azure ポータルで、 **az30305a-labRG** リソース グループ ブレードのリソース リストで、 **az30305a-lb** 負荷分散装置を選択し、および **az30305a-lb** ブレードで、パブリック IP アドレスのエントリをメモします。

1. Azure portal の Cloud Shell ペインで **Bash** セッションを開始します。 

1. Cloud Shell ペインから次のコマンドを実行して、Azure Load Balancer のバックエンド プールにある Azure VM への HTTP トラフィックの負荷分散をテストします (`<lb_IP_address>` プレースホルダーを以前に特定したロード バランサーのフロントエンドの IP アドレスに置き換えます)。

   ```Bash
   for i in {1..4}; do curl <lb_IP_address>; done
   ```

    > **注**: 返されたメッセージが、要求が任意のバックエンド Azure VM に配信されていることを示していることを確認します

1. 「**az30305a-lb**」 ブレードで、「**負荷分散規則**」 エントリを選択し、HTTP トラフィックを処理する負荷分散規則を表す 「**az303005a-lbruletcp80**」 エントリを選択します。 

1. 「**az303005a-lbruletcp80**」 ブレードの 「**セッション永続化**」 ドロップダウン リストで、「**クライアント IP**」 を選択し、「**保存**」 を選択します。

1. 更新が完了するのを待ち、Cloud Shell ペインから以下を再実行して、セッション永続性がない Azure Load Balancer のバックエンド プール内にある Azure VM に対する HTTP トラフィックの負荷分散をテストします (`<lb_IP_address>` プレースホルダーを、前に特定した負荷分散装置のフロントエンド IP アドレスに置き換えます)。

   ```Bash
   for i in {1..4}; do curl <lb_IP_address>; done
   ```

    > **注**: 返されたメッセージでは、同じバックエンドの Azure VM にリクエストが配信されていることを示していることを確認してください

1. Azure portal で、「**az30305a-lb**」 ブレードに戻り、「**インバウンド NAT 規則**」 エントリを選択して、TCP ポート 33890 および 33891 を介してリモート デスクトップ経由でそれぞれバックエンド プールに接続できるように設定されていることを確認します。フロントエンドIPアドレスと、2 つのポート番号をメモします。 

1. Cloud Shell ペインから、次のコマンドを実行して、Azure Load Balancer のバックエンド プールにある最初の Azure VM への NAT 経由のリモート デスクトップの接続をテストします (`<lb_IP_address>` プレースホルダーを、前に特定した負荷分散装置のフロントエンド IP アドレスに置き換えます)。

   ```Bash
   curl -v telnet://<lb_IP_address>:33890
   ```

    > **注**: 返されたメッセージが接続成功を示していることを確認します。 

1. **Ctrl+C** キーを押して Bash シェル プロンプトに戻り、次を実行して、Azure Load Balancer のバックエンド プールにある 2 番目の Azure VM への NAT 経由のリモート デスクトップの接続をテストします (`<lb_IP_address>` プレースホルダーを、前に特定した負荷分散装置のフロントエンド IP アドレスに置き換えます) 。

   ```Bash
   curl -v telnet://<lb_IP_address>:33891
   ```

    > **注**: 返されたメッセージが接続成功を示していることを確認します。 

1. **Ctrl+C** キーの組み合わせを押して、Bash シェル プロンプトに戻ります。


#### タスク 3: 演習でデプロイされた Azure リソースを削除する

1. 「Cloud Shell」 ウィンドウから次のコマンドを実行して、この演習で作成したリソース グループを一覧表示します。

   ```Bash
   az group list --query "[?starts_with(name,'az30305a-')]".name --output tsv
   ```

    > **注**: このラボで作成したリソース グループのみが出力に含まれていることを確認します。このグループは、このタスクで削除されます。

1. 「Cloud Shell」 ウィンドウから次を実行して、このラボで作成したリソース グループを削除します

   ```sh
   az group list --query "[?starts_with(name,'az30305a-')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

1. 「Cloud Shell」 ペインを閉じます。


### 演習 2: Availability Zones と Azure Load Balancer Standard を使用して高可用性 Azure VM デプロイを実装および分析する
  
この演習の主なタスクは次のとおりです。

1. Azure Resource Manager テンプレートを使用して、高可用性 Azure VM を Azure Load Balancer Standard の背後にある Availability zones ゾーンにデプロイする

1. Azure Load Balancer Standard の背後にある Availability Zones 全体にデプロイされた高可用性 Azure VM を分析する

1. 演習でデプロイされた Azure リソースを削除する


#### タスク 1: Azure Resource Manager テンプレートを使用して、高可用性 Azure VM を Azure Load Balancer Standard の背後にある Availability zones ゾーンにデプロイする

1. 必要に応じて、Azure portal で、検索テキストボックスの右にあるツールバー アイコンを直接選択して、「**Cloud Shell**」 ペインを開きます。

1. **Bash** または **PowerShell** のいずれかを選択するためのプロンプトが表示されたら、**「Bash」** を選択します。 

1. Cloud Shell ペインのツールバーで、「**ファイルのアップロード/ダウンロード**」アイコンを選択し、ドロップダウン メニューで「**アップロード**」を選択して、ファイル **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305subb.json** を Cloud Shell ホーム ディレクトリにアップロードします。

1. Cloud Shell ペインから次のコマンドを実行してリソース グループを作成します (`<Azure region>` プレースホルダーを、サブスクリプションで使用可能で、ラボ コンピューターの場所に最も近い Azure リージョンの名前に置き換えます)。

   ```Bash
   LOCATION='<Azure region>'
   ```
   
   ```Bash
   az deployment sub create \
   --location $LOCATION \
   --template-file azuredeploy30305subb.json \
   --parameters rgName=az30305b-labRG rgLocation=$LOCATION
   ```

1. Cloud Shell ペインから、Azure Resource Manager テンプレート **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgb.json** をアップロードします。

1. Cloud Shell ペインから、Azure Resource Manager パラメーター ファイル **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgb.parameters.json** をアップロードします。

1. Cloud Shell ペインから次のコマンドを実行して、2 つの Availability Zones にわたって Windows Server 2019 Datacenter Core をホストする Azure VM のペアで構成されるバックエンド プールを備えた Azure Load Balancer Standard をデプロイします。

   ```Bash
   az deployment group create \
   --resource-group az30305b-labRG \
   --template-file azuredeploy30305rgb.json \
   --parameters @azuredeploy30305rgb.parameters.json
   ```

    > **注**: 次のタスクに進む前に、デプロイが完了するまで待機します。これにはおよそ 10 分かかります。

1. Azure portal で「Cloud Shell」ペインを閉じます。 


#### タスク 2: Azure Load Balancer Standard の背後にある Availability Zones 全体にデプロイされた高可用性 Azure VM を分析する

1. Azure portal で、「**Network Watcher**」を検索して選択し、「**Network Watcher**」ブレードで、「**トポロジ**」を選択します。

1. 「**Network Watcher \| トポロジ**」 ブレードで、次の設定を指定します。

    | 設定 | 値 | 
    | --- | --- |
    | 定期売買 | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30305b-labRG** |
    | 仮想ネットワーク | **az30305b-vnet** |

1. トポロジ図を確認し、パブリック IP アドレス、ロードバランサー、およびバックエンドプール内の Azure VM のネットワークアダプター間の接続に注意を払います。

    > **注**: このダイアグラムは、前の演習で表示した図とほぼ同じです。これは、異なるゾーン (実際には Azure データ センター) に存在しても、Azure VM は同じサブネット上にあるためです。
    
1. 「**Network Watcher**」ブレードで、「**有効なセキュリティルール**」 を選択します。

1. 「**Network Watcher」** \| 「**有効なセキュリティルール**」 ブレードで、次の設定を指定します。

    | 設定 | 値 | 
    | --- | --- |
    | 定期売買 | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30305b-labRG** |
    | 仮想マシン | **az30305b-vm0** |
    | ネットワーク インターフェイス | **az30305b-nic0** |

1. 関連するネットワーク セキュリティ グループと、RDP および HTTP を介した受信接続を許可する 2 つのカスタム ルールを含む有効なセキュリティ ルールを確認します。 

    > **注**: この登録情報は前の演習で表示したものと実質的に同じですが、ネットワークレベルの保護は、両方の Azure VM が接続されているサブネットに関連付けられたネットワーク セキュリティ グループを使用して実装されています。ただし、この場合、Azure Load Balancer Standard SKU を使用するため、HTTP および RDP トラフィックがバックエンド プールの Azure VM に到達するためにネットワーク セキュリティ グループが必要です (Basic SKU を使用する場合、NSG はオプションです)。  
    
    > **注**: 「**有効なセキュリティ ルール**」は、
    - **az30305a-nic0** の「ネットワーク インターフェイス」ブレードから表示することもできます。
    - **az30305a-web-nsg** の「ネットワーク セキュリティ グループ」ブレード 

1. **Network Watcher** ブレードで、「**接続のトラブルシューティング**」 を選択します。

    > **注**: 同じ可用性セット内の 2 つの Azure VM の近接性 (ネットワークの観点から) を確認することが目的です。

1. 「**Network Watcher**」 \| 「**接続のトラブルシューティング**」 ブレードで、次の設定を指定して、「**確認**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | 定期売買 | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30305b-labRG** |
    | ソース タイプ | **仮想マシン** |
    | 仮想マシン | **az30305b-vm0** |
    | 宛先 | **仮想マシンを選択する** |
    | リソース グループ | **az30305b-labRG** |
    | 仮想マシン | **az30305b-vm1** |
    | プロトコル | **TCP** |
    | Destination port (依存先ポート)| **80** |

    > **注**: **Azure Network Watcher エージェント** VM 拡張機能を Azure VM にインストールするには数分かかります。

1. 結果を確認し、Azure VM 間のネットワーク接続の遅延を確認します。

    > **注**: 2 つの VM は異なるゾーン (異なる Azure データセンター内) にあるため、待ち時間が前の演習で確認した時間よりわずかに長くなる可能性があります。

1. Azure portal で、「**az30305b-labRG**」 リソース グループ ブレードに移動し、リソース リストで 「**az30305b-vm0**」 仮想マシン エントリを選択し、「**az30305b-vm0**」 ブレードで 「**場所**」 および 「**可用性ゾーン**」 エントリを確認します。 

1. Azure portal で、「**az30305b-labRG**」 リソース グループ ブレードに移動し、リソース リストで 「**az30305b-vm1**」 仮想マシン エントリを選択し、「**az30305b-vm1**」 ブレードで 「**場所**」 および 「**可用性ゾーン**」 エントリを確認します。 

    > **注**: 確認したエントリにより、各 Azure VM が異なる可用性ゾーンにあることが確認されます。

1. Azure portal で、「**az30305b-labRG**」 リソース グループ ブレードに移動し、リソース リストで 「**az30305b-lb**」 負荷分散装置エントリを選択し、「**az30305b-lb**」 ブレードでパブリック IP アドレス エントリを確認します。

1. Azure portal の Cloud Shell ペインで、新しい **Bash** セッションを起動します。 

1. Cloud Shell ペインから次のコマンドを実行して、Azure Load Balancer のバックエンド プールにある Azure VM への HTTP トラフィックの負荷分散をテストします (`<lb_IP_address>` プレースホルダーを以前に特定したロード バランサーのフロントエンドの IP アドレスに置き換えます)。

   ```Bash
   for i in {1..4}; do curl <lb_IP_address>; done
   ```

    > **注**: 返されたメッセージが、要求が任意のバックエンド Azure VM に配信されていることを示していることを確認します

1. 「**az30305b-lb**」 ブレードで、「**負荷分散規則**」 エントリを選択し、「**az30305b-lb \| 負荷分散規則**」 ブレードで、HTTP トラフィックを処理する負荷分散規則を表す 「**az303005b-lbruletcp80**」 エントリを選択します。 

1. 「**az303005b-lbruletcp80**」 ブレードの 「**セッション永続化**」 ドロップダウン リストで、「**クライアント IP**」 を選択し、「**保存**」 を選択します。

1. 更新が完了するのを待ち、Cloud Shell ペインから以下を再実行して、セッション永続性がない Azure Load Balancer のバックエンド プール内にある Azure VM に対する HTTP トラフィックの負荷分散をテストします (`<lb_IP_address>` プレースホルダーを、前に特定した負荷分散装置のフロントエンド IP アドレスに置き換えます)。

   ```Bash
   for i in {1..4}; do curl <lb_IP_address>; done
   ```

    > **注**: 返されたメッセージでは、同じバックエンドの Azure VM にリクエストが配信されていることを示していることを確認してください

1. Azure portal で、「**az30305b-lb**」 ブレードに戻り、「**インバウンド NAT 規則**」 エントリを選択して、TCP ポート 33890 および 33891 を介してリモート デスクトップ経由でそれぞれバックエンド プール VM の最初と 2 番目に接続できるようにする 2 つの規則をメモします。 

1. Cloud Shell ペインから、次のコマンドを実行して、Azure Load Balancer のバックエンド プールにある最初の Azure VM への NAT 経由のリモート デスクトップの接続をテストします (`<lb_IP_address>` プレースホルダーを、前に特定した負荷分散装置のフロントエンド IP アドレスに置き換えます)。

   ```Bash
   curl -v telnet://<lb_IP_address>:33890
   ```

    > **注**: 返されたメッセージが接続成功を示していることを確認します。 

1. **Ctrl+C** キーを押して Bash シェル プロンプトに戻り、次を実行して、Azure Load Balancer のバックエンド プールにある 2 番目の Azure VM への NAT 経由のリモート デスクトップの接続をテストします (`<lb_IP_address>` プレースホルダーを、前に特定した負荷分散装置のフロントエンド IP アドレスに置き換えます) 。

   ```Bash
   curl -v telnet://<lb_IP_address>:33891
   ```

    > **注**: 返されたメッセージが接続成功を示していることを確認します。 

1. **Ctrl+C** キーを押して、Bash シェル プロンプトに戻り、Cloud Shell ペインを閉じます。

1. 「**az30305b-lb**」 ブレードで、「**負荷分散規則**」 エントリを選択し、「**az30305b-lb \| 負荷分散規則**」 ブレードで、HTTP トラフィックを処理する負荷分散規則を表す 「**az303005b-lbruletcp80**」 エントリを選択します。 

1. **az303005b-lbruletcp80** ブレードの「**アウトバウンドの送信元ネットワーク アドレス変換 (SNAT)**」セクションで、「**(推奨) アウトバウンド ルールを使用してバックエンドのプール メンバーにインターネットにアクセスできるようにします**」を選択し、「**保存**」を選択します。

1. 「**az30305b-lb**」 ブレードに戻り、「**送信規則**」 エントリを選択し、「**+ 追加**」 を選択します。

1. 「**アウトバウンド規則を追加**」 ブレードで、次の設定を指定し、「**追加**」 を選択します (他のすべての設定は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | 名前 | **az303005b-obrule** | 
    | フロントエンド IP アドレス | **az30305b-lb** 負荷分散装置の既存のフロントエンド IP アドレスの名前 |
    | バックエンド プール | **az30305b-bepool** |
    | ポートの割り当て | **送信ポートの数を手動で選択する** |
    | 選択基準 | **バックエンド インスタンスの最大数** |
    | バックエンド インスタンスの最大数 | **3** |

    > **注**: Azure Load Balancer Standard では、送信トラフィック専用のフロントエンド IP アドレスを指定できます (複数のフロントエンド IP アドレスが割り当てられている場合)。

1. Azure portal で、**az30305b-labRG** リソース グループ ブレードに移動し、リソースのリストで仮想マシン **az30305b-vm0** のエントリを選択し、**az30305b-vm0** ブレードの**操作**ブレードで 「**実行コマンド**」 を選択します。

1. 「**az30305b-vm0 \| 実行コマンド**」 ブレードで、**RunPowerShellScript** を選択します。 

1. **実行コマンド** ブレードの **PowerShell スクリプト** テキスト ボックスに次のように入力し、「**実行**」 を選択します。

   ```powershell
   (Invoke-RestMethod -Uri "http://ipinfo.io").IP
   ```

    > **注**: このコマンドで、Web 要求の発信元のパブリック IP アドレスを返します。

1. 出力を確認し、アウトバウンド負荷分散規則に割り当てた Azure Load Balancer Standard のフロントエンドに割り当てられたパブリック IP アドレスと一致することを確認します。


#### タスク 3: 演習でデプロイされた Azure リソースを削除する

1. Azure portal の Cloud Shell ペインで、新しい **Bash** セッションを起動します。 

1. 「Cloud Shell」 ウィンドウから次のコマンドを実行して、この演習で作成したリソース グループを一覧表示します。

   ```Bash
   az group list --query "[?starts_with(name,'az30305b-')]".name --output tsv
   ```

    > **注**: このラボで作成したリソース グループのみが出力に含まれていることを確認します。このグループは、このタスクで削除されます。

1. 「Cloud Shell」 ウィンドウから次を実行して、このラボで作成したリソース グループを削除します

   ```Bash
   az group list --query "[?starts_with(name,'az30305b-')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```

1. 「Cloud Shell」 ペインを閉じます。


### 演習 3: Availability Zones とAzure Application Gateway を使用して、高可用性 Azure VM スケール セットのデプロイを実装および分析します。
  
この演習の主なタスクは次のとおりです。

1. Azure Resource Manager テンプレートを使用して、高可用性 Azure VM スケールセットを Azure Application Gateway の背後にある Availability Zones にデプロイする

1. Azure Application Gateway の背後にある Availability Zones 全体にデプロイされた高可用性 Azure VM スケールセットを分析する

1. 演習でデプロイされた Azure リソースを削除する


#### タスク 1: Azure Resource Manager テンプレートを使用して、高可用性 Azure VM スケールセットを Azure Application Gateway の背後にある Availability Zones にデプロイする

1. 必要に応じて、Azure portal で、検索テキストボックスの右にあるツールバー アイコンを直接選択して、「**Cloud Shell**」 ペインを開きます。

1. **Bash** または **PowerShell** のいずれかを選択するためのプロンプトが表示されたら、**「Bash」** を選択します。 

1. Cloud Shell ペインのツールバーで、「**ファイルのアップロード/ダウンロード**」アイコンを選択し、ドロップダウン メニューで「**アップロード**」を選択して、ファイル **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305subc.json** を Cloud Shell ホーム ディレクトリにアップロードします。

1. Cloud Shell ペインから次のコマンドを実行してリソース グループを作成します (`<Azure region>` プレースホルダーを、サブスクリプションで使用可能で、ラボ コンピューターの場所に最も近い Azure リージョンの名前に置き換えます)。

   ```Bash
   az deployment sub create --location '<Azure region>' --template-file azuredeploy30305subc.json --parameters rgName=az30305c-labRG rgLocation='<Azure region>'
   ```

1. Cloud Shell ペインから、Azure Resource Manager テンプレート **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgc.json** をアップロードします。

1. Cloud Shell ペインから、Azure Resource Manager パラメーター ファイル **\\\\AZ303\\AllFiles\\Labs\\05\\azuredeploy30305rgc.parameters.json** をアップロードします。

1. Cloud Shell ペインから次のコマンドを実行して、Azure Application Gateway をデプロイします。そのバックエンド プールは、さまざまな Availability Zones にわたって Windows Server 2019 Datacenter Core をホストする Azure VM のペアで構成されています。

   ```
   az deployment group create --resource-group az30305c-labRG --template-file azuredeploy30305rgc.json --parameters @azuredeploy30305rgc.parameters.json
   ```

    > **注**: 次のタスクに進む前に、デプロイが完了するまで待機します。これにはおよそ 10 分かかります。

1. Azure portal で「Cloud Shell」ペインを閉じます。 


#### タスク 2: Azure Application Gateway の背後にある Availability Zones 全体にデプロイされた高可用性 Azure VM スケールセットを分析する

1. Azure portal で、「**Network Watcher**」を検索して選択し、「**Network Watcher**」ブレードで、「**トポロジ**」を選択します。

1. 「**Network Watcher \| トポロジ**」ブレードで、次の設定を指定します。

    | 設定 | 値 | 
    | --- | --- |
    | 定期売買 | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | **az30305c-labRG** |
    | 仮想ネットワーク | **az30305c-vnet** |

1. 結果のトポロジ ダイアグラムを確認し、パブリック IP アドレス、負荷分散装置、および Azure Virtual Machine Scale Set 内の Azure VM インスタンスのネットワーク アダプター間の接続をバックエンド プールに記録します。 

    > **注**: さらに、Azure Application Gateway のデプロイには、ダイアグラムに含まれている専用のサブネットが必要です (ただし、ゲートウェイは表示されません)。

    > **注**: この構成では、Network Watcher を使用して有効なネットワーク セキュリティ ルールを表示することはできません (Azure VM と Azure VM Scale Set のインスタンスの違いの 1 つ)。同様に、Azure Application Gateway からの接続をテストすることは可能ですが、**接続のトラブルシューティング**を使用して Azure VM Scale Set インスタンスからのネットワーク接続をテストすることはできません。

1. Azure portal で、リソースリストから **az30305c-labRG** リソース グループ ブレードに移動し、**az30305c-vmss** 仮想マシン スケール セットのエントリを選択します。 

1. **az30305c-vmss** ブレードで、**場所**と**障害ドメイン**のエントリを確認します。 

    > **注**: Azure VM とは異なり、Azure VM Scale Sets の個々のインスタンスは、同じゾーンにデプロイされたインスタンスを含め、別々の障害ドメインにデプロイされます。さらに、5 つの障害ドメインをサポートします (使用できる障害ドメインが最大で 3 つである Azure VM とは異なります)。 

1. 「**az30305c-vmss**」 ブレードで 「**インスタンス**」 を選択し、「**az30305c-vmss \| インスタンス**」 ブレードの最初のインスタンスを選択し、「**場所**」 プロパティの値を確認して可用性ゾーンを識別します。 

1. 「**az30305c-vmss**」 に戻ります | **インスタンス**ブレードに戻り、2 番目のインスタンスを選択し、「**場所**」 プロパティの値を確認して可用性ゾーンを識別します。 

    > **注**: 各インスタンスが異なる可用性ゾーンにあることを確認します。

1. Azure portal で、「**az30305c-labRG**」リソース グループ ブレードに移動し、リソースのリストで、「**az30305c-appgw**」ロード バランサー エントリを選択し、「**az30305c-appgw**」ブレードで、パブリック IP アドレスのエントリをメモします。

1. Azure portal の Cloud Shell ペインで、新しい **Bash** セッションを起動します。 

1. Cloud Shell ペインから、次のコマンドを実行して、Azure Application Gateway のバックエンド プールにある Azure VM スケール セット インスタンスへの HTTP トラフィックの負荷分散をテストします (`<lb_IP_address>` プレースホルダーを以前に特定したゲートウェイのフロントエンドの IP アドレスに置き換えます)。

   ```Bash
   for i in {1..4}; do curl <appgw_IPaddress>; done
   ```

    > **注**: 返されたメッセージが、要求がラウンドロビン方式でバックエンド Azure VM に配信されていることを示していることを確認します

1. 「**az30305c-appgw**」ブレードで、「**HTTP 設定**」エントリを選択し、「**az30305c-appgw**」で **| 「HTTP 設定」** ブレードで、HTTP トラフィックを処理する負荷分散規則を表す 「**appGwBackentHttpSettings**」エントリを選択します。 

1. 「**appGwBackentHttpSettings**」ブレードで、変更を加えずに既存の設定を確認し、「**Cookie ベースのアフィニティ**」を有効にできることに注意します。

    > **注**: この機能では、クライアントが Cookie の使用をサポートしている必要があります。

    > **注**: Azure VM スケール セットのインスタンスへの RDP 接続に NAT を実装するために Azure Application Gateway を使用できません。Azure Application Gateway は HTTP/HTTPS トラフィックのみをサポートします。


### 演習 4: Availability Zones とAzure Application Gateway を使用して Azure VM スケール セットの自動スケーリングを実装する。
  
この演習の主なタスクは次のとおりです。

1. Azure VM スケール セットの自動スケーリングの構成

1. Azure VM スケール セットの自動スケールのテスト

#### タスク 1: Azure VM スケール セットの自動スケーリングを構成する

1. Azure portal で、「**az30305c-labRG**」リソース グループ ブレードに移動し、リソースのリストで、「**az30305c-vmss**」仮想マシン スケール セット エントリを選択し、「**az30305c-vmss**」ブレードで、「**スケーリング**」を選択します。 

1. 「**az30305c-vmss \| スケーリング**」ブレードで、「**カスタム自動スケーリング**」オプションを選択します。

1. 「**カスタム自動スケーリング**」 セクションで、次の設定を指定します (他は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | スケーリング モード | **メトリックに基づくスケール** |
    | インスタンス制限最小 | **1** |
    | インスタンス制限最大 | **3** |
    | インスタンス制限既定値 | **1** |

1. 「**+ 規則の追加**」 を選択します。

1. 「**スケール ルール**」 ブレードで、次の設定を指定し、「**追加**」を選択します (他は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | 時間の集計 | **最大値** |
    | メトリック名前空間 | **仮想マシン ホスト** |
    | メトリック名 | **Percentage CPU** |
    | VMName 演算子 | **=** |
    | ディメンション値 | **az30305c-vmss_0** |
    | インスタンス数によりメトリックの除算を有効にする | **Enabled** |
    | 演算子 | **次の値より大きい** |
    | スケール操作をトリガーするメトリックのしきい値 | **1** |
    | 期間 (分) | **1** |
    | 時間グレインの統計 | **最大値** |
    | 操作 | **カウントを増やす量** |
    | インスタンス数:  | **1** |
    | クールダウン (分) | **5** |

    > **注**: ラボの進行上、これらの値を厳密に選択し、できるだけ早くスケーリングをトリガーさせます。Azure VM スケール セットのスケーリングに関するガイダンスについては、 [Microsoft Docs](https://docs.microsoft.com/ja-jp/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-autoscale-overview) を参照してください。 

1. **az30305c-vmss** に戻る **| スケーリング** ブレードに戻り、**ルールを追加**を選択します。

1. 「**スケール ルール**」 ブレードで、次の設定を指定し、「**追加**」を選択します (他は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | 時間の集計 | **平均** |
    | メトリック名前空間 | **仮想マシン ホスト** |
    | メトリック名 | **Percentage CPU** |
    | VMName 演算子 | **=** |
    | ディメンション値 | **az30305c-vmss_0 と az30305c-vmss_1 を選択** |
    | インスタンス数によるメトリックの除算を有効にする | **Enabled** |
    | 演算子 | **次の値より小さい** |
    | スケール アクションをトリガーするメトリックのしきい値 | **1** |
    | 期間 (分) | **1** |
    | 時間グレインの統計 | **最小** |
    | オペレーション | **カウントを減らす量** |
    | インスタンス数:  | **1** |
    | クールダウン (分) | **5** |

1. **az30305c-vmss | スケーリング** ブレード、「**保存**」 を選択します。


#### タスク 2: Azure VM スケール セットの自動スケーリングをテストする

1. Azure portal の Cloud Shell ペインで、新しい **Bash** セッションを起動します。 

1. Cloud Shell ペインから次を実行して、Azure Application Gateway のバックエンド プール内の Azure VM スケール セット インスタンスの自動スケーリングをトリガーします (`<lb_IP_address>`プレースホルダーを、先ほど特定したゲートウェイのフロントエンドの IP アドレスに置き換えます):

   ```Bash
   for (( ; ; )); do curl -s <lb_IP_address>?[1-10]; done
   ```
1. Azure portal の **az30305c-vmss** の**概要**ブレードで**監視**タブを選択し、**CPU (平均)** チャートを確認し、Application Gateway の CPU 使用率が十分に増加して、スケール アウトをトリガーできることを確認します。

    > **注**: 数分待つ必要があるかもしれません。

1. **az30305c-vmss** ブレードで、**インスタンス** エントリを選択し、インスタンスの数が増加していることを確認します。

    > **注**: 場合によっては、「**az30305c-vmss** を更新する必要があります **| インスタンス**」 ブレードを更新する必要があります。

    > **注**: インスタンスの数が (1 ではなく) 2 増加するのを確認する場合があります。これは、実行中のインスタンスの最終的な数が 3 である限り予想されることです。 

1. Azure portal で、**Cloud Shell** ウィンドウを閉じます。 

1. Azure portal の **az30305c-vmss** ブレードで、**CPU (平均)** チャートを確認し、Application Gateway の CPU 使用率が十分に低下して、スケール インをトリガーできることを確認します。 

    > **注**: 数分待つ必要があるかもしれません。

1. **az30305c-vmss** ブレードで、**インスタンス** エントリを選択し、インスタンスの数が 2 に減少したことを確認します。

    > **注**: 場合によっては、「**az30305c-vmss** を更新する必要があります **| インスタンス**」 ブレードを更新する必要があります。

1. **az30305c-vmss** ブレードで、「**スケーリング**」 を選択します。 

1. 「**az30305c-vmss**」 \| 「**スケーリング**」 ブレード、「**手動スケール**」 オプションを選択し、「**保存**」 を選択します。

    > **注**: これにより、次の演習中に不要な自動スケーリングが行われなくなります。 


### 演習 5: Azure VM スケール セットのバーティカル スケーリングの実装

この演習の主なタスクは次のとおりです。

1. Azure Virtual Machine スケール セット インスタンスのコンピューティング リソースのスケーリング。

1. Azure Virtual Machine スケール セットのインスタンスのストレージ リソースのスケーリング。


#### タスク 1: Azure 仮想マシン スケール セット インスタンスのコンピューティング リソースをスケーリングします。

1. Azure portal の 「**az30305c-vmss**」 ブレードで、「**サイズ**」 を選択します。

1. 使用可能なサイズのリストで、現在構成されていない使用可能なサイズ(DS1_v2 など)を選択し、「**サイズ変更**」 を選択します。

1. 「**az30305c-vmss**」 ブレードで 「**インスタンス**」 エントリを選択し、「**az30305c-vmss \| インスタンス**」 ブレードで、既存のインスタンスを目的のサイズの新しいインスタンスに置き換えるプロセスを監視します。

    > **注**: 場合によっては、「**az30305c-vmss**」 \| を更新する必要があります 「**インスタンス**」 ブレードを更新する必要があります。

1. インスタンスが更新されて実行されるまで待機します。


#### タスク 2: Azure 仮想マシン スケール セット インスタンスのストレージ リソースをスケーリングします。

1. 「**az30305c-vmss**」 ブレードで、「**ディスク**」 を選択し、「**+ データ ディスクの追加**」 を選択して、次の設定で新しいマネージド ディスクを接続し (他は既定値のままにします)、「**保存**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | LUN | **0** |
    | サイズ | **32** |
    | ストレージ アカウント タイプ | **標準 HDD** |

1. 「**az30305c-vmss**」 ブレードで 「**インスタンス**」 エントリを選択し、「**az30305c-vmss**」 \| 「**インスタンス**」 ブレードで、既存のインスタンスを更新するプロセスを監視します。

    > **注**: 前の手順で接続されたディスクは未フォーマット ディスクです。使用する前に、パーティションを作成してフォーマットし、マウントする必要があります。これを行うには、カスタム スクリプト拡張機能を介して PowerShell スクリプトを Azure VM スケール セット インスタンスにデプロイします。ただし、最初に、それを削除する必要があります。

1. 「**az30305c-vmss**」 ブレードで、「**拡張機能**」 を選択し、「**az30305c-vmss \| 拡張機能**」 で 「**customScriptExtension**」 エントリを選択し、「**拡張機能**」 ブレードで 「**アンインストール**」 を選択します。

    > **注**: アンインストールが完了するまで待機します。

1. Azure portal で、「**az30305c-labRG**」 リソース グループ ブレードに移動し、リソースのリストでストレージ アカウント リソースを選択します。 

1. ストレージ アカウント ブレードで、「**Containers**」 を選択し、「**+ コンテナー**」 を選択します。 

1. 「**新しいコンテナー**」 ブレードで、次の設定を指定し (他は既定値のままにします)、「**作成**」 を選択します。

    | 設定 | 値 | 
    | --- | --- |
    | 名前 | **scripts** |
    | パブリック アクセス レベル | **非公開 (匿名アクセスなし**) |
    
1. コンテナーのリストを表示しているストレージ アカウント ブレードに戻り、「**scripts**」 を選択します。

1. 「**scripts**」 ブレードで、「**アップロード**」 を選択します。

1. 「**BLOB のアップロード**」 ブレードでフォルダー アイコンを選択し、「**開く**」 ダイアログ ボックスで **\\\\AZ303\\AllFiles\\Labs\\05** フォルダーに移動して、「**az30305e-configure_VMSS_with_data_disk.ps1**」 を選択します。「**開く**」 を選択し、「**BLOB のアップロード**」 ブレードに戻り、「**アップロード**」 を選択します。 

1. Azure portal で、「**az30305c-vmss**」 仮想マシン スケール セット ブレードに戻ります。 

1. 「**az30305c-vmss**」 ブレードで、「**拡張機能**」 を選択し、「**az30305c-vmss \| 拡張機能**」 ブレードで 、「**+ 追加**」 を選択し、「**拡張機能**」 ブレードで 「**customScriptExtension**」 エントリを選択します。

1. 「**新しいリソース**」 ブレードで、「**カスタム スクリプト拡張機能**」 を選択してから、「**作成**」 を選択します。

1. 「**拡張機能のインストール**」 ブレードから、「**参照**」 を選択します。 

1. 「**ストレージ アカウント**」 ブレードで、**az30305e-configure_VMSS_with_data_disk.ps1** スクリプトをアップロードしたストレージ アカウントの名前を選択し、「**コンテナ**」 ブレードで 「**スクリプト**」 を選択し、「**スクリプト**」 ブレードで 「**az30305e-configure_VMSS_with_data_disk.ps1**」 を選択してから、「**選択**」 を選択します。 

1. 「**拡張機能のインストール**」 ブレードに戻り、「**OK**」 を選択します。

1. 「**az30305c-vmss**」 ブレードで 「**インスタンス**」 エントリを選択し、「**az30305c-vmss**」 で **|** 「**インスタンス**」 ブレードで、既存のインスタンスを更新するプロセスを監視します。

    > **注**: 場合によっては、「**az30305c-vmss**」 \| を更新する必要があります 「**インスタンス**」 ブレードを更新する必要があります。


#### タスク 3: 演習でデプロイされた Azure リソースを削除する

1. 「Cloud Shell」 ウィンドウから次のコマンドを実行して、この演習で作成したリソース グループを一覧表示します。

   ```Bash
   az group list --query "[?starts_with(name,'az30305c-')]".name --output tsv
   ```

    > **注**: このラボで作成したリソース グループのみが出力に含まれていることを確認します。このグループは、このタスクで削除されます。

1. 「Cloud Shell」 ウィンドウから次を実行して、このラボで作成したリソース グループを削除します

   ```Bash
   az group list --query "[?starts_with(name,'az30305c-')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
   ```
   
1. 「Cloud Shell」 ペインを閉じます。
