---
lab:
    title: '06: Azure Storage File および Blob Services の実装と構成'
    module: 'モジュール 06: ストレージ アカウントの実装'
---

# ラボ: Azure Storage File および Blob Services の実装と構成
# 受講生用ラボ マニュアル

## ラボ シナリオ
 
Adatum Corporation は、オンプレミス ストレージで大量の非構造化データと半構造化データをホストしています。そのメンテナンスはますます複雑になり、コストがかかります。一部のデータは、データ保持要件に対処するために長期間保存されます。Adatum エンタープライズ アーキテクチャ チームは、Tiered Storage をサポートすると同時に、データ流出の可能性を最小限に抑える安全なアクセスを許可する安価な代替手段を探しています。チームは Azure Storage によって提供される実質的に無制限の容量を認識していますが、対応するストレージ アカウントのコンテンツ全体への無制限のアクセスを許可するアカウント キーの使用について懸念しています。キーは順番に回転させることができますが、そのような操作は適切な計画で実行する必要があります。また、アクセス キーは承認メカニズムのみを構成し、その使用を適切に監査する機能を制限します。

これらの欠点に対処するため、アーキテクチャ チームは Shared Access Signatures の使用を検討することを決定しました。Shared Access Signature（SAS）は、ストレージ アカウントのリソースへの安全な委任アクセスを提供すると同時に、意図しないデータ漏洩の可能性を最小限に抑えます。SAS は、BLOB などの個々のストレージ オブジェクトへのアクセスを制限する機能、カスタム タイム ウィンドウへのアクセスを制限する機能、指定された IP アドレス範囲へのネットワーク アクセスのフィルタリングなど、データ アクセスをきめ細かく制御します。さらに、アーキテクチャ チームは、Azure Storage と Azure Active Directory の統合レベルを評価して、監査要件に対処することを望んでいます。アーキテクチャチームは、一部のオンプレミス ファイル共有の代替として、Azure Files の適合性を判断することも決定しました。

これらの目標を達成するために、Adatum Corporation は、次を含む Azure Storage リソースのさまざまな認証および許可メカニズムをテストします。

-  アカウント、コンテナー、オブジェクトレベルで Shared Access Signatures を使用する

-  BLOB のアクセス レベルの構成 

-  Azure Active Directory ベースの許可の実装

-  ストレージ アカウント アクセス キーの使用


## 目標
  
このラボを完了すると、次のことができるようになります。

-  Shared Access Signatures を活用して Azure Storage BLOB の承認を実装する

-  Azure Active Directory を活用して Azure Storage BLOB の承認を実装する

-  アクセス キーを活用して Azure Storage ファイル共有の承認を実装する


## ラボ環境
  
Windows Server 管理者の資格情報

-  User Name (ユーザー名): **Student**

-  パスワード: **Pa55w.rd1234**

推定時間: 90 分


## ラボ ファイル

-  \\\\AZ303\\AllFiles\\Labs\\06\\azuredeploy30306suba.json

-  \\\\AZ303\\AllFiles\\Labs\\06\\azuredeploy30306rga.json

-  \\\\AZ303\\AllFiles\\Labs\\06\\azuredeploy30306rga.parameters.json


### 演習 0: ラボ環境の準備

この演習の主なタスクは次のとおりです。

 1. Azure Resource Manager テンプレートを使用して Azure VM をデプロイする


#### タスク 1: Azure Resource Manager テンプレートを使用して Azure VM をデプロイする

1. ラボのコンピューターから Web ブラウザーを起動し、 [Azure portal](https://portal.azure.com) に移動し、このラボで使用するサブスクリプションの所有者の役割を持つユーザーアカウントの認証情報を提供してサインインします。

1. Azure で、検索テキストボックスのすぐ右にあるツールバーアイコンを選択して、**Cloud Shell** ペインを表示します。

1. **Bash** や **PowerShell** のどちらかを選択するためのプロンプトが表示されたら、**PowerShell** を選択します。 

    > **注**: **Cloud Shell** を初めて起動し、「**ストレージがマウントされていません**」というメッセージが表示された場合は、このラボで使用しているサブスクリプションを選択し、「**ストレージの作成**」を選択します。 

1. Cloud Shell ペインのツールバーで、 **ファイルのアップロード/ダウンロード** アイコンを選択し、ドロップダウン メニューで **アップロード**を選択して、ファイル **\\\\\AZ303\\AllFiles\Labs\\06\\azuredeploy30306suba.json** を Cloud Shell ホーム ディレクトリにアップロードします。

1. Cloud Shell ペインから次のコマンドを実行してリソースグループを作成します（ `<Azure region>`プレースホルダーを、サブスクリプションでの Azure VM のデプロイに使用可能で、ラボのコンピューターの場所に最も近い Azure リージョンの名前に置き換えます ）:

   ```powershell
   $location = '<Azure region>'
   ```
   
   ```powershell
   New-AzSubscriptionDeployment `
     -Location $location `
     -Name az30306subaDeployment `
     -TemplateFile $HOME/azuredeploy30306suba.json `
     -rgLocation $location `
     -rgName 'az30306a-labRG'
   ```

      > **注記**: Azure VM をプロビジョニングできる Azure リージョンを識別するには、[**https://azure.microsoft.com/ja-jp/regions/offers/**](https://azure.microsoft.com/ja-jp/regions/offers/) を参照してください。

1. Cloud Shell ペインから、Azure Resource Manager テンプレート **\\\\AZ303\\AllFiles\Labs\\06\\azuredeploy30306rga.json** をアップロードします。

1. Cloud Shell ペインから、Azure Resource Manager パラメーター ファイル **\\\\ AZ303 \\ AllFilesLabs \\ 06 \\ azuredeploy30306rga.parameters.json** をアップロードします 。

1. Cloud Shell ペインから次を実行して、このラボで使用する Windows Server 2019 を実行する Azure VM をデプロイします:

   ```powershell
   New-AzResourceGroupDeployment `
     -Name az30306rgaDeployment `
     -ResourceGroupName 'az30306a-labRG' `
     -TemplateFile $HOME/azuredeploy30306rga.json `
     -TemplateParameterFile $HOME/azuredeploy30306rga.parameters.json `
     -AsJob
   ```

    > **注**: デプロイが完了するのを待たず、代わりに次の演習に進みます。デプロイにかかる時間は 5 分未満です。

1. Azure portal で、**Cloud Shell** ウィンドウを閉じます。 


### 演習 1: Shared Access Signatures を使用して Azure Storage アカウントの承認を構成します。
  
この演習の主なタスクは次のとおりです。

1. Azure Storage アカウントを作成する

1. Storage Explorer をインストールする

1. アカウント レベルの Shared Access Signature を生成する

1. Azure Storage Explorer を使用してBLOB コンテナーを作成する

1. AzCopy を使用してファイルを BLOB コンテナーにアップロードする

1. BLOB レベルの Shared Access Signature を使用して BLOB にアクセスする


#### タスク 1: Azure Storage アカウントを作成する

1. Azure portalで、「**ストレージ アカウント**」 を検索して選択し、「**ストレージ アカウント**」 ブレードで、「**+ 追加**」 を選択します。

1. 「**ストレージ アカウントの作成**」 ブレードの 「**基本**」 タブで 、次の設定を指定します (他の設定は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | 定期売買 | このラボで使用する Azure サブスクリプションの名前 |
    | リソース グループ | 新しいリソース グループ **az30306a-LabRG** の名前 |
    | ストレージ アカウント名 | 文字と数字で構成される、長さ 3 文字から 24 文字までのグローバルに一意な名前 |
    | 場所 | Azure ストレージ アカウントを作成できる Azure リージョンの名前  |
    | パフォーマンス | **Standard** |
    | アカウントの種類 | **StorageV2 (汎用 v2)** |
    | レプリケーション | **ローカル冗長ストレージ (LRS)** |

1. 「**次へ:**」を選択します。 「**ネットワーク>**」、「**ストレージ アカウントの作成**」 ブレードの 「**ネットワーク**」 タブで、利用可能なオプションを確認し、デフォルトのオプション 「**パブリック エンドポイント (すべてのネットワーク}**」 を承諾し、「**次へ:**」 を選択します「**データ保護 >**」 をクリックします。

1. 「**ストレージ アカウントの作成**」 ブレードの 「**データ保護**」 タブで、利用可能なオプションを確認し、デフォルトを承諾し、「**次へ:**」 を選択します「**詳細 >**」 をクリックします。

1. 「**ストレージ アカウントの作成**」 ブレードの 「**詳細**」 タブで、利用可能なオプションを確認し、デフォルトを承諾し、 「**Review + create**」 を選択し、検証プロセスが完了するのを待って 「**作成**」 を選択します。

    > **注**: ストレージ アカウントが作成されるのを待ちます。これにはおよそ 2 分かかります。


#### タスク 2: Storage Explorer をインストールする

   > **注**: 続行する前に、このラボの最初に開始した Azure VM のデプロイが完了していることを確認してください。 

1. Azure portal で、「**仮想マシン**」 を検索して選択し、「**仮想マシン**」 ブレードで、仮想マシンのリスト内の 「**az30306a-vm0**」 を選択します。

1. 「**az30306a-vm0**」 ブレードで、「**接続**」 を選択し、ドロップダウン メニューで、 「**RDP**」、次に 「**RDP ファイルをダウンロード**」 を選択します。

1. プロンプトが表示されたら、次の認証情報を入力します。

    | 設定 | 値 | 
    | --- | --- |
    | 「ユーザー名」 | **Student** |
    | パスワード | **Pa55w.rd1234** |

1. **az30306a-vm0** へのリモート デスクトップ セッション内にある 「サーバー マネージャー」 ウィンドウで、「**ローカル サーバー**」 を選択し、「**IE 強化されたセキュリティ構成**」 ラベルの横にある 「**オン**」 のリンクを選択し、「**IE 強化されたセキュリティ構成**」 ダイアログ ボックスで、両方の 「**オフ**」 オプションを選択します。

1. 「**az30306a-vm0**」 へのリモート デスクトップ セッション内で、Internet Explorer を起動し、[Microsoft Edge](https://www.microsoft.com/ja-jp/edge/business/download) のダウンロードページに移動し、Microsoft Edge インストーラーをダウンロードして、インストールを実行します。 

1. 「**az30306a-vm0**」 へのリモート デスクトップ セッション内で、Microsoft Edge で、[Azure Storage Explorer](https://azure.microsoft.com/ja-jp/features/storage-explorer/) のダウンロード ページに移動します

1. 「**az30306a-vm0**」 へのリモート デスクトップ セッション内で、既定の設定で Azure Storage Explorer をダウンロードしてインストールします。 


#### タスク 3: アカウント レベルの Shared Access Signature を生成する

1. 「**az30306a-vm0**」 へのリモート デスクトップ セッション内で、Microsoft Edge を起動し、[Azure portal](https://portal.azure.com) に移動し、このラボで使用しているサブスクリプションの所有者の役割を持つユーザー アカウントの資格情報を提供してサインインします。

1. 新しく作成したストレージ アカウントのブレードに移動し、「**アクセス キー**」 を選択して、ターゲット ブレードの設定を確認します。

    > **注**: 各ストレージ アカウントには、個別に再生成できる 2 つのキーがあります。ストレージ アカウント名と 2 つのキーのいずれかがわかっていると、ストレージ アカウント全体に完全にアクセスできます。 

1. ストレージ アカウント ブレードで、「**Shared Access Signature**」 を選択します。

1. 次の設定を指定します (他は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | 使用できるサービス | **BLOB** |
    | 許可されるリソースの種類 | **サービス**および**コンテナー** |
    | 与えられているアクセス許可 | **読み取り**、 **リスト** および **作成** |
    | BLOB のバージョン管理のアクセス許可 | 無効にする |
    | 開始 | 現在のタイム ゾーンの現在時刻の 24 時間前 | 
    | 終了 | 現在のタイム ゾーンの現在時刻から 24 時間後 |
    | 許可されているプロトコル | **HTTPS のみ** |
    | 署名キー | **key1** |

1. 「**SAS および接続文字列を生成する**」 を選択します。

1. 「**Blob service の SAS URL**」 の値をクリップボードにコピーします。


#### タスク 4: Azure Storage Explorer を使用してBLOB コンテナーを作成する

1. **az30306a-vm0** へのリモート デスクトップ セッション内で、Azure Storage Explorer を起動します。 

1. 「Azure Storage Explorer」 ウィンドウを最初に起動すると、自動的に **「Connect to Azure Storage」** ウィンドウが開きます。**「Select Resource」** で、**「Storage Account or Server」** を選択します。

1. **「Select Connection Method」** タブで、**「Shared Access Signature (SAS)」** を選択し **Next** をクリックします。

1. **「Enter Connection Info」** タブで、**「Display Name」** テキスト ボックスに「**az30306a-blobs**」を入力し、**「Service URL」** に、クリップボードにコピーした値を貼り付け、**「Next」** を選択します。 

1. **「Summary」** ウィンドウで、**「Connect」** を選択します。 

1. Azure Storage Explorer ウィンドウの 「**Explorer**」 ペインで、「**az30306a-blobs**」 エントリに移動して展開し、「**BLOB コンテナー**」 エンドポイントのみにアクセスできることを確認します（Table、Queue、Files は表示されません）。 

1. 「**az30306a-blobs**」 エントリを右クリックし、右クリック メニューで 「**Create BLOB Container**」 を選択し、空のテキストボックスを使用してコンテナー名を「**container1**」に設定します。

1. **「Container1」** を選択して、メイン画面で **「Upload」** を選択し、ドロップダウン リストで、**「Upload Files」** を選択します。

1. 「**Upload Files**」 ウィンドウで、**C:\Windows\system.ini** を選択し、「**Upload**」 を選択します。

1. 「**ファイルのアップロード**」 ウィンドウに戻り、「**Activities**」 リストに表示されるエラー メッセージを確認します。 

    >**注**: Shared Access Signature はオブジェクトレベルのアクセス許可を提供しないため、これは正常な動作です。 

1. Azure Storage Explorer ウィンドウは開いたままにします。


#### タスク 5: AzCopy を使用してファイルを BLOB コンテナーにアップロードする

1. ブラウザーの画面の 「**Shared Access Signature**」 ブレードで、次の設定を指定します (他は既定値のままにします)。

    | 設定 | 値 | 
    | --- | --- |
    | 使用できるサービス | **BLOB** |
    | 許可されるリソースの種類 | **Object** |
    | 許可されるアクセス許可 | **読み取り**、 **作成** |
    | BLOB のバージョン管理権限 | 無効にする |
    | 開始 | 現在のタイム ゾーンの現在時刻の 24 時間前 | 
    | 終了 | 現在のタイム ゾーンの現在時刻から 24 時間後 |
    | 許可されているプロトコル | **HTTPS のみ** |
    | 署名キー | **key1** |

1. 「**SAS および接続文字列を生成する**」 を選択します。

1. **SAS トークン**の値をクリップボードにコピーします。

1. **Cloud Shell** を起動します。

1. **Bash** や **PowerShell** のどちらかを選択するためのプロンプトが表示されたら、**PowerShell** を選択します。 

1. Cloud Shell ペインから次のコマンドを実行してファイルを作成し、そこにテキスト行を追加します。

   ```powershell
   New-Item -Path './az30306ablob.html'

   Set-Content './az30306ablob.html' '<h3>Hello from az30306ablob via SAS</h3>'
   ```

1. Cloud Shell ペインから次のコマンドを実行し、新しく作成したファイルを Blob として、この演習の前半で作成した Azure Storage アカウントの container 1 にアップロードします（ このタスクの前半でクリップボードにコピーした Shared Access Signature の `<sas_token>` というプレースホルダ を差し替えます)。

   ```powershell
   $storageAccountName = (Get-AzStorageAccount -ResourceGroupName 'az30306a-labRG')[0].StorageAccountName

   azcopy cp './az30306ablob.html' "https://$storageAccountName.blob.core.windows.net/container1/az30306ablob.html<sas_token>"
   ```

1. AzCopy によって生成された出力を確認し、ジョブが正常に完了したことを確認します。

1. 「Cloud Shell」 ペインを閉じます。

1. ブラウザーの画面のストレージ アカウント ブレードで、「**コンテナー**」 を選択します。

1. 一覧で **container1** を選択します。

1. **container1** ブレードで、 blob のリストに **az30306ablob.html** が表示されていることを確認します。


#### タスク 6: BLOB レベルの Shared Access Signature を使用して BLOB にアクセスする

1. 「**az30306a-vm0**」 へのリモート デスクトップ セッション内の、ブラウザーの画面の 「**container1**」 ブレードで、「**アクセス レベルの変更**」 を選択し、「**プライベート (匿名アクセスはありません)**」 に設定されていることを確認し、「**キャンセル**」 を選択します。

    >**注**: 匿名アクセスを許可する場合は、パブリック アクセス レベルを次のように設定できます。 **Blob（Blob の匿名読み取りアクセスのみ）** または **コンテナー（コンテナーと Blobの匿名の読み取りアクセス）**。

1. **container1** ブレードで、**az30306ablob.html**を選択 します。

1. 「**az30306ablob.html**」 ブレードで、「**SAS の生成**」 を選択し、使用可能なオプションを変更せずに確認してから、「**SAS トークンと URL の生成**」 を選択します。

1. **BLOB SAS URL** の値をクリップボードにコピーします。

1. Microsoft Edge の新しいタブを開き、前の手順でクリップボードにコピーした URL に移動します。

1. **Hello from az30306ablob via SAS** というメッセージがブラウザーの画面に表示されることを確認します。


### 演習 2: Azure Active Directory を使用してAzure Storage BLOB サービスの承認を構成する
  
この演習の主なタスクは次のとおりです。

1. Azure AD ユーザーの作成

1. Azure Storage BLOB サービスの Azure Active Directory 承認を有効にする

1. AzCopy を使用してファイルを BLOB コンテナーにアップロードする


#### タスク 1: Azure AD ユーザーの作成

1. 「**az30306a-vm0**」 へのリモート デスクトップ セッション内の、ブラウザーの画面で、「**Cloud Shell**」 ペイン内の **PowerShell** セッションを開きます。

1. Cloud Shell ペインから次のコマンドを実行して、Azure AD テナントに対して明示的に認証します。

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
   New-AzureADUser -AccountEnabled $true -DisplayName 'az30306auser1' -PasswordProfile $passwordProfile -MailNickName 'az30306auser1' -UserPrincipalName "az30306auser1@$domainName"
   ```

1. Cloud Shell ウィンドウから、次の操作を実行して、新しく作成された Azure AD ユーザーのユーザー プリンシパル名を識別します。

   ```powershell
   (Get-AzureADUser -Filter "MailNickName eq 'az30306auser1'").UserPrincipalName
   ```

1. ユーザー プリンシパル名をメモします。これついては、演習の後半で取り上げます。 

1. 「Cloud Shell」 ペインを閉じます。


#### タスク 2: Azure Storage BLOB サービスの Azure Active Directory 承認を有効にする

1. 「**az30306a-vm0**」 へのリモート デスクトップ セッション内で、Azure portal を表示しているブラウザーの画面で、「**container1**」 ブレードに戻ります。

1. 「**container1**」 ブレードで、**認証方法** の横に表示されている「**Azure AD のユーザー アカウントに切り替える**」 を選択します。

1. BLOB コンテナー内のデータを一覧表示する権限がなくなったことを示すエラー メッセージに注意します。これは予測されていることです。

    >**注**: サブスクリプションに**所有者**の役割があるにもかかわらず、**ストレージ BLOB データ所有者**、**ストレージ BLOB データ共同作成者**、または **ストレージ BLOB データ閲覧者** など、ストレージ アカウントの BLOB コンテンツへのアクセスを提供する組み込みまたはカスタムの役割を割り当てる必要もあります。

1. Azure portal で、「**container1**」 をホストしているストレージ アカウントのブレードに戻り、「**アクセス制御 (IAM)**」 を選択し、「**+ 追加**」 を選択して、ドロップダウン リストで、「**ロールの割り当ての追加**」 を選択します。 

    >**注**: ストレージ アカウントの名前を書き留めます。この名前は、次のタスクで必要になります。

1. 「**ロールの割り当ての追加**」 ブレードの 「**ロール**」 ドロップダウン リストで、「**ストレージ BLOB データ所有者**」 を選択し、「**アクセスの割り当て**」ドロップダウン リスト エントリが 「**ユーザー、グループ、またはサービス プリンシパル**」 に設定されていることを確認し、「**選択**」 テキスト ボックスの下に表示されたリストからあなた自身のユーザー アカウントと前のタスクで作成したユーザー アカウントの両方を選択し、「**保存**」 を選択します。

1. 「**container1**」 ブレードに戻り、再度「**Azure AD のユーザーアカウントに切り替える**」をクリックします。エラーが出る場合は、一度ログインしなおしてみてください。

1. コンテナーのコンテンツが表示されることを確認します。   

#### タスク 3: AzCopy を使用してファイルを BLOB コンテナーにアップロードする

1. **「az30306a-vm0」** へのリモート デスクトップ セッション内で、Windows PowerShell を起動します。 

1. Windows PowerShell プロンプトから次のコマンドを実行して、**azcopy.zip** アーカイブをダウンロードし、そのコンテンツを抽出して、**azcopy.exe** を含む場所に切り替えます。

   ```powershell
   $url = 'https://aka.ms/downloadazcopy-v10-windows'
   $zipFile = '.\azcopy.zip'

   Invoke-WebRequest -Uri $Url -OutFile $zipFile

   Expand-Archive -Path $zipFile -DestinationPath '.\'

   Set-Location -Path 'azcopy*'
   ```

1. Windows PowerShell プロンプトから次のコマンドを実行し、この演習の最初のタスクで作成した Azure AD ユーザー アカウントを使用して AzCopy を認証します。 

   ```powershell
   .\azcopy.exe login
   ```

    >**注**: この目的では Microsoft アカウントを使用できないため、Azure AD ユーザー アカウントを最初に作成する必要がありました。

1. 前のステップで実行したコマンドで生成されたメッセージの指示に従って、**az30306auser1** ユーザー アカウントとして認証します。認証資格情報の入力を求められたら、この演習の最初のタスクでメモしたアカウントのユーザー プリンシパル名とそのパスワード **Pa55w.rd1234** を入力します。

1. 認証に成功したら、Windows PowerShell プロンプトから次のコマンドを実行して、**container1** へアップロードするファイルを作成します。

   ```powershell
   New-Item -Path './az30306bblob.html'

   Set-Content './az30306bblob.html' '<h3>Hello from az30306bblob via Azure AD</h3>'
   ```

1. Windows PowerShell プロンプトから次のコマンドを実行して、前の演習で作成した Azure Storage アカウントの **container1** へ、新しく作成したファイルを blob としてアップロードします (プレースホルダー `<storage_account_name>` を、前のタスクでメモしたストレージ アカウントの値に置き換えます)。

   ```powershell
   .\azcopy cp './az30306bblob.html' 'https://<storage_account_name>.blob.core.windows.net/container1/az30306bblob.html'
   ```

1. AzCopy によって生成された出力を確認し、ジョブが正常に完了したことを確認します。

1. Windows PowerShell プロンプトから次のコマンドを実行して、AzCopy ユーティリティで提供されるセキュリティ コンテキストの外部にある、アップロードされた blob にアクセスできないことを確認します (プレースホルダー `<storage_account_name>` を、前のタスクでメモしたストレージ アカウントの値に置き換えます)。

   ```powershell
   Invoke-WebRequest -Uri 'https://<storage_account_name>.blob.core.windows.net/container1/az30306bblob.html'
   ```

1. ブラウザーの画面の **az30306a-vm0** へのリモート デスクトップ セッション内で 、**container1** に戻ります。

1. **container1** ブレードで、 blob のリストに **az30306bblob.html** が表示されていることを確認します。

1. **container1** ブレードで、**アクセス レベルの変更** を選択し、パブリック アクセス レベルを **Blob （Blob のみの匿名読み取りアクセス）** に設定して、**OK** を選択します。 

1. Windows PowerShell プロンプトに戻り、次のコマンドを再実行して、アップロードされた blob に匿名でアクセスできることを確認します（ `<storage_account_name>`プレースホルダーを、前のタスクでメモしたストレージ アカウントの値に置き換えます）:

   ```powershell
   Invoke-WebRequest -Uri 'https://<storage_account_name>.blob.core.windows.net/container1/az30306bblob.html'
   ```


### 演習 3: Azure Files を実装します。
  
この演習の主なタスクは次のとおりです。

1. Azure Storage ファイル共有を作成する

1. Windows からドライブを Azure Storage ファイル共有にマップする

1. ラボにデプロイした Azure リソースを削除する


#### タスク 1: Azure Storage ファイル共有を作成する

1. **az30306a-vm0** へのリモート デスクトップ セッション内で、Azure portal が表示されているブラウザーの画面で、このラボの最初の演習で作成したストレージ アカウントのブレードに戻り、**ファイル サービス**セクションで、**ファイル共有**を選択します。

1. **+ファイル共有**を選択し、次の設定でファイル共有を作成します:

    | 設定 | 値 |
    | --- | --- |
    | 名前 | **az30306a-share** |
    | クォータ | **1024** |


#### タスク 2: Windows からドライブを Azure Storage ファイル共有にマップする

1. 新しく作成したファイル共有を選択し、 **接続**を選択します。

1. **接続**ブレードで、**Windows** タブが選択されていることを確認し、**クリップボードにコピー**を選択します。

    >**注**: Azure Storage のファイル共有マッピングでは、ターゲット共有にアクセスするために、ストレージ アカウント名と 2 つのストレージ アカウントキーの1 つを、それぞれユーザー名とパスワードに相当するものとして使用します。

1. **az30306a-vm0** へのリモート デスクトップ セッション内の PowerShell プロンプトで、コピーしたスクリプトを貼り付けて実行します。

1. スクリプトが正常に完了したことを確認します。 

1. ファイル エクスプローラーを起動し、**Z:** ドライブに移動して、マッピングが成功したことを確認します。 

1. ファイル エクスプローラーで、**Folder1** という名前のフォルダと、**File1.txt** という名前のフォルダ内にテキスト ファイルを作成します。

1. Azure portal を表示しているブラウザー ウィンドウに戻り、**az30306a-share** ブレードで、**更新**を選択し、**Folder1** がフォルダーのリストに表示されるのを確認します。 

1. **Folder1**を選択し、ファイルのリストに**File1.txtt**が表示されることを確認します。


#### タスク 3: ラボにデプロイした Azure リソースを削除する

1. **az30306a-vm0** へのリモート デスクトップ セッション内の Azure portal を表示しているブラウザーの画面で、Cloud Shell ペイン内で PowerShell セッションを起動します。

1. 「Cloud Shell」 ウィンドウから次のコマンドを実行して、この演習で作成したリソース グループを一覧表示します。

   ```powershell
   Get-AzResourceGroup -Name 'az30306*'
   ```

    > **注**: このラボで作成したリソース グループのみが出力に含まれていることを確認します。このグループは、このタスクで削除されます。

1. 「Cloud Shell」 ウィンドウから次を実行して、このラボで作成したリソース グループを削除します

   ```powershell
   Get-AzResourceGroup -Name 'az30306*' | Remove-AzResourceGroup -Force -AsJob
   ```

1. 「Cloud Shell」 ペインを閉じます。

1. Azure portal で、Azure サブスクリプションに関連付けられている Azure Active Directory テナントの「**ユーザー**」ブレードに移動します。

1. ユーザー アカウントの一覧で **az30306auser1** ユーザー アカウントを表すエントリを選択し、ツールバーの省略記号アイコンを選択して「**ユーザーの削除**」を選択し、確認のプロンプトが表示されたら「**はい**」を選択します。  
