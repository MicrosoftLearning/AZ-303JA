---
lab:
    title: '10: Azure ロールベースのアクセス制御 (RBAC) を管理する'
    module: 'モジュール 10: Azure のガバナンスの導入と管理'
---

# ラボ: Azure ロールベースのアクセス制御 (RBAC) を管理する
# 受講生用ラボ マニュアル

## ラボ シナリオ

Azure Active Directory (Azure AD) が ID 管理環境の不可欠な部分になると、Adatum エンタープライズ アーキテクチャ チームも最適な承認アプローチを決定する必要があります。Azure リソースへのアクセスを制御するコンテキストでは、そのようなアプローチには Azure ロールベースのアクセス制御 (RBAC) を使用する必要があります。RBAC は、Azure リソースのきめ細かいアクセス管理を提供する Azure Resource Manager 上に構築された承認システムです。

Azure RBAC の主要な概念は、ロールの割り当てです。ロールの割り当ては、セキュリティ プリンシパル、ロールの定義、スコープの 3 つの要素で構成されています。セキュリティ プリンシパルは、Azure リソースへのアクセスを要求するユーザー、グループ、サービス プリンシパル、またはマネージド ID を表すオブジェクトです。ロールの定義は、読み取り、書き込み、削除など、ロールの割り当てによって付与される操作のコレクションです。ロールは、一般的またはリソース固有にすることができます。Azure には、4 つの組み込みの一般的なロール (所有者、共同作成者、閲覧者、およびユーザー アクセス管理者) と、非常に多くの組み込みのリソース固有のロール (たとえば、Azure 仮想マシンを作成および管理する権限を含む仮想マシン共同作成者など) が含まれています。カスタム ロールを定義することも可能です。スコープは、アクセスが適用されるリソース セットです。スコープは複数のレベル (管理グループ、サブスクリプション、リソース グループ、リソース) で設定できます。スコープは親子関係で構造化されています。

Adatum エンタープライズ アーキテクチャ チームは、カスタムのロールベースのアクセス制御ロールを使用して、Azure 管理の委任をテストしたいと考えています。評価を開始するために、チームは Azure 仮想マシンへの制限付きアクセスを提供するカスタム ロールを作成する予定です。 
  

## 目標
  
このラボを完了すると、次のことができるようになります。

-  カスタム RBAC ロールの定義 

-  カスタム RBAC ロールの割り当て


## ラボ環境
  
Windows Server 管理者の資格情報

-  User Name (ユーザー名): **Student**

-  パスワード: **Pa55w.rd1234**

推定時間: 60 分


## ラボ ファイル

-  \\\\AZ303\\AllFiles\\Labs\\10\\azuredeploy30310suba.json

-  \\\\AZ303\\AllFiles\\Labs\\10\\azuredeploy30310rga.json

-  \\\\AZ303\\AllFiles\\Labs\\10\\azuredeploy30310rga.parameters.json

-  \\\\AZ303\\AllFiles\\Labs\\10\\roledefinition30310.json


## 説明

### 演習 0: ラボ環境の準備

この演習の主なタスクは次のとおりです。

1. Azure Resource Manager テンプレートを使用して Azure VM をデプロイする

1. Azure Active Directory ユーザーを作成する


#### タスク 1: Azure Resource Manager テンプレートを使用して Azure VM をデプロイする

1. ラボのコンピューターから Web ブラウザーを起動し、 [Azure portal](https://portal.azure.com) に移動し、このラボで使用するサブスクリプションの所有者の役割を持つユーザーアカウントの認証情報を提供してサインインします。

1. Azure で、検索テキストボックスのすぐ右にあるツールバーアイコンを選択して、**Cloud Shell** ペインを表示します。

1. **Bash** や **PowerShell** のどちらかを選択するためのプロンプトが表示されたら、**PowerShell** を選択します。 

    >**注**: **Cloud Shell** を初めて起動し、「**ストレージがマウントされていません**」というメッセージが表示された場合は、このラボで使用しているサブスクリプションを選択し、「**ストレージの作成**」を選択します。 

1. Cloud Shell ペインのツールバーで、 **ファイルのアップロード/ダウンロード** アイコンを選択し、ドロップダウン メニューで **アップロード**を選択して、ファイル **\\\\\AZ303\\AllFiles\Labs\\10\\azuredeploy30310suba.json** を Cloud Shell ホーム ディレクトリにアップロードします。

1. Cloud Shell ペインから次のコマンドを実行してリソースグループを作成します（ `<Azure region>`プレースホルダーを、サブスクリプションでの Azure VM のデプロイに使用可能で、ラボのコンピューターの場所に最も近い Azure リージョンの名前に置き換えます ）:

   ```powershell
   $location = '<Azure region>'
   New-AzSubscriptionDeployment `
     -Location $location `
     -Name az30310subaDeployment `
     -TemplateFile $HOME/azuredeploy30310suba.json `
     -rgLocation $location `
     -rgName 'az30310a-labRG'
   ```

      >**注記**: Azure VM をプロビジョニングできる Azure リージョンを識別するには、[**https://azure.microsoft.com/ja-jp/regions/offers/**](https://azure.microsoft.com/ja-jp/regions/offers/) を参照してください。

1. Cloud Shell ペインから、Azure Resource Manager テンプレート **\\\\AZ303\\AllFiles\Labs\\10\\azuredeploy30310rga.json** をアップロードします。

1. Cloud Shell ペインから、Azure Resource Manager パラメーター ファイル **\\\\ AZ303 \\ AllFilesLabs \\ 10 \\ azuredeploy30310rga.parameters.json** をアップロードします 。

1. Cloud Shell ペインから次を実行して、このラボで使用する Windows Server 2019 を実行する Azure VM をデプロイします:

   ```powershell
   New-AzResourceGroupDeployment `
     -Name az30310rgaDeployment `
     -ResourceGroupName 'az30310a-labRG' `
     -TemplateFile $HOME/azuredeploy30310rga.json `
     -TemplateParameterFile $HOME/azuredeploy30310rga.parameters.json `
     -AsJob
   ```

    > **注**: デプロイが完了するのを待たずに、代わりに次のタスクに進みます。デプロイにかかる時間は 5 分未満です。


#### タスク 2: Azure Active Directory ユーザーを作成する

1. Azure portal で、Cloud Shell ペインの PowerShell セッションから次のコマンドを実行して、Azure サブスクリプションに関連付けられているAzure AD テナントに対して認証します:

   ```powershell
   Connect-AzureAD
   ```
   
1. 「Cloud Shell」ペインで以下を実行して、Azure AD DNS ドメイン名を識別します。

   ```powershell
   $domainName = ((Get-AzureAdTenantDetail).VerifiedDomains)[0].Name
   ```

1. Cloud Shell ウィンドウから、次の操作を実行して、新しい Azure AD ユーザーを作成します。

   ```powershell
   $passwordProfile = New-Object -TypeName Microsoft.Open.AzureAD.Model.PasswordProfile
   $passwordProfile.Password = 'Pa55w.rd1234'
   $passwordProfile.ForceChangePasswordNextLogin = $false
   New-AzureADUser -AccountEnabled $true -DisplayName 'az30310aaduser1' -PasswordProfile $passwordProfile -MailNickName 'az30310aaduser1' -UserPrincipalName "az30310aaduser1@$domainName"
   ```

1. Cloud Shell ウィンドウから、次の操作を実行して、新しく作成された Azure AD ユーザーのユーザー プリンシパル名を識別します。

   ```powershell
   (Get-AzureADUser -Filter "MailNickName eq 'az30310aaduser1'").UserPrincipalName
   ```

      > **注記**: 新しく作成した Azure AD ユーザーのユーザー プリンシパル名を記録します。これは、このラボで後ほど必要になります。

1. 「Cloud Shell」 ペインを閉じます。


### 演習 1: カスタム RBAC ロールの定義
  
この演習の主なタスクは次のとおりです。

1. RBAC を使用して委託するアクションを特定する

1. Azure AD テナントにカスタム RBAC ロールを作成する


#### タスク 1: RBAC を使用して委託するアクションを特定する

1. Azure portal で、**「az30310a-LabRG」** ブレードに移動します。

1. **「az30310a-LabRG」** ブレードで、「**Access Control (IAM)**」 をクリックします。

1. 「**az30310a-labRG - Access Control (IAM)**」 ブレードで、「**ロール（クラシック）**」 をクリックします。

1. 「**ロール（クラシック）**」 ブレードで、「**所有者**」 を選択します。

1. 「**所有者**」 ブレードで、「**アクセス許可**」 を選択します。

1. 「**アクセス許可 (プレビュー)**」 ブレードで、「**Microsoft Compute**」 を選択します。

1. 「**Microsoft Compute**」 ブレードで、「**仮想マシン**」 を選択します。

1. 「**仮想マシン**」ブレードで、RBAC を使用して委任できる管理アクションのリストを確認します。これには、**仮想マシンの割り当て解除**と **仮想マシンの起動**アクションが含まれます。


#### タスク 2: Azure AD テナントにカスタム RBAC ロールを作成する

1. ファイル **\\\\AZ303\\AllFiles\\Labs\\10\\roledefinition30310.json**を開き、その内容を確認します。SUBSCRIPTION_ID には現在使用しているサブスクリプションの ID が入りますが、これはこの後のステップで行います。このロールには、「すべてのリソースの閲覧」、「仮想マシンの再起動」そして「仮想マシンの起動」の権限が含まれています。「仮想マシンの停止」は含まれていません。

   ```json
   {
      "Name": "Virtual Machine Operator (Custom)",
      "Id": null,
      "IsCustom": true,
      "Description": "Allows to start/restart Azure VMs",
      "Actions": [
          "Microsoft.Compute/*/read",
          "Microsoft.Compute/virtualMachines/restart/action",
          "Microsoft.Compute/virtualMachines/start/action"
      ],
      "NotActions": [
      ],
      "AssignableScopes": [
          "/subscriptions/SUBSCRIPTION_ID"
      ]
   }
   ```

1. ラボ コンピューターで、Azure portal を表示しているブラウザーの画面で、 **Cloud Shell** 内の **PowerShelll** セッションを起動します。 

1. Cloud Shell ペインから、Azure Resource Manager テンプレート **\\\\AZ303\\AllFiles\\Labs\\10\\roledefinition30310.json** をホーム ディレクトリにアップロードします。

1. Cloud Shell ペインから、`SUBSCRIPTION_ID` プレースホルダーを Azure サブスクリプションの ID 値に置き換えるには、次の操作を実行します。

   ```powershell
   $subscription_id = (Get-AzContext).Subscription.id
   (Get-Content -Path $HOME/roledefinition30310.json) -Replace 'SUBSCRIPTION_ID', "$subscription_id" | Set-Content -Path $HOME/roledefinition30310.json
   ```

1. Cloud Shell ペインから、次を実行して、 `SUBSCRIPTION_ID` プレースホルダーが Azure サブスクリプションの ID 値に置き換えられたことを確認します。

   ```powershell
   Get-Content -Path $HOME/roledefinition30310.json
   ```

1. Cloud Shell ウィンドウから、次の操作を実行して、カスタム ロール定義を作成します。

   ```powershell
   New-AzRoleDefinition -InputFile $HOME/roledefinition30310.json
   ```

1. Cloud Shell ウィンドウから、次の操作を実行して、ロールが正常に作成されたことを確認します。

   ```powershell
   Get-AzRoleDefinition -Name 'Virtual Machine Operator (Custom)'
   ```

1. 「Cloud Shell」 ペインを閉じます。


### 演習 2: カスタム RBAC ロールの割り当てとテスト
  
この演習の主なタスクは次のとおりです。

1. RBAC ロールの割り当てを作成する

1. RBAC ロール割り当てのテスト


#### タスク 1: RBAC ロールの割り当てを作成する
 
1. Azure portal で、**「az30310a-LabRG」** ブレードに移動します。

1. **「az30310a-LabRG」** ブレードで、「**Access Control (IAM)**」 をクリックします。

1. **「az30310a-labRG - Access Control (IAM)」** ブレードで、**「+ 追加」** をクリックして、**「ロールの割り当てを追加」** オプションを選択します。

1. **「ロールの割り当てを追加」** ブレード、次の設定を指定し（他の設定は既存の値のままにします）、「**保存**」選択します:

    | 設定 | 値 | 
    | --- | --- |
    | Role | **Virtual Machine Operator (Custom)** |
    | アクセスの割り当て | **ユーザー、グループ、またはサービス プリンシパル** |
    | OK | **az30310aaduser1** |


#### タスク 2: RBAC ロール割り当てのテスト

1. ラボ コンピューターから新しいプライベート Web ブラウザー セッションを開始し、[Azure portal](https://portal.azure.com) に移動し、**「az30310aaduser1」** ユーザー アカウントと **「Pa55w.rd1234」** パスワードを使用してログインします。

    > **注**: このラボの前半で記録した **「az30310aaduser1」** ユーザー アカウントのユーザー プリンシパル名を必ず使用してください。

1. Azure portal で、「**リソース グループ**」 ブレードに移動します。リソース グループは表示されないことに注意してください。 

1. Azure portal で、**すべてのリソース**ブレードに移動します。**az30310a-vm0**とそのマネージド ディスクのみが表示されることに注意してください。

1. Azure portal で、**「az30310a-vm0」** ブレードに移動します。仮想マシンを停止してみてください。通知領域のエラー メッセージを確認し、現在のユーザーが実行を許可されていないため、この操作が失敗したことに留意してください。

1. 仮想マシンを再起動し、アクションが正常に完了したことを確認します。

1. 非公開の Web ブラウザー セッションを閉じます。


#### タスク 3: ラボにデプロイした Azure リソースを削除する

1. ラボ コンピューターの Azure portal を表示している既存のブラウザーの画面で、Cloud Shell ペイン内の PowerShell セッションを起動します。

1. 「Cloud Shell」 ウィンドウから次のコマンドを実行して、この演習で作成したリソース グループを一覧表示します。

   ```powershell
   Get-AzResourceGroup -Name 'az30310*'
   ```

    > **注**: このラボで作成したリソース グループのみが出力に含まれていることを確認します。このグループは、このタスクで削除されます。

1. Cloud Shell ペインから、以下を実行して、このラボで作成したリソースグループを削除します。

   ```powershell
   Get-AzResourceGroup -Name 'az30310*' | Remove-AzResourceGroup -Force -AsJob
   ```

1. Cloud Shell ペインから、以下を実行して、このラボで以前にアップロードしたラボ ファイルを削除します。

   ```powershell
   Get-ChildItem -Path . -Filter 'az30310*.json' | Remove-Item -Force
   Get-ChildItem -Path . -Filter 'roledefinition30310.json' | Remove-Item -Force
   ```

1. 「Cloud Shell」 ペインを閉じます。

1. Azure portal で、Azure サブスクリプションに関連付けられている Azure Active Directory テナントの「**ユーザー**」ブレードに移動します。

1. ユーザー アカウントの一覧で **az30310aaduser1** ユーザー アカウントを表すエントリを選択し、ツールバーの省略記号アイコンを選択して「**ユーザーの削除**」を選択し、確認のプロンプトが表示されたら「**はい**」を選択します。  

1. Azure portal で、Azure サブスクリプションのプロパティを表示しているブレードに移動し、「**アクセス制御 (IAM)**」エントリを選択してから、「**ロール**」を選択します。

1. ロールのリストで、「**仮想マシンオペレーター (カスタム)**」エントリを選択し、「**削除**」を選択し、確認を求められたら、「**はい**」を選択します。
