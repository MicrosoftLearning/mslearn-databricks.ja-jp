---
lab:
  title: Azure Databricks と Azure DevOps または Azure Databricks と GitHub を使用して CI/CD パイプラインを実装する
---

# Azure Databricks と Azure DevOps または Azure Databricks と GitHub を使用して CI/CD パイプラインを実装する

Azure Databricks と Azure DevOps または Azure Databricks と GitHub を使用して継続的インテグレーション (CI) パイプラインと継続的デプロイ (CD) パイプラインを実装するには、コードの変更が統合され、テストされ、効率的にデプロイされるように、一連の自動化された手順を設定する必要があります。 通常、このプロセスには、Git リポジトリへの接続、Azure Pipelines を使用したジョブの実行、コードのビルドと単体テスト、Databricks ノートブックで使用するためのビルド成果物のデプロイが含まれます。 このワークフローにより、堅牢な開発サイクルが可能になり、最新の DevOps プラクティスに合わせた継続的インテグレーションとデリバリーが可能になります。

このラボは完了するまで、約 **40** 分かかります。

>**注:** この演習を完了するには、Github アカウントと Azure DevOps アクセス権が必要です。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。

2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示されたら、***PowerShell*** 環境を選んで、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#9723;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

     ```powershell
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
     ```

5. リポジトリをクローンした後、次のコマンドを入力して **setup.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

     ```powershell
    ./mslearn-databricks/setup.ps1
     ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。

7. スクリプトが完了するまで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの[Delta Lake の概要](https://docs.microsoft.com/azure/databricks/delta/delta-intro)に関する記事をご確認ください。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS 以降のランタイム バージョンを持つクラスターが既にある場合は、それを使ってこの演習を完了し、この手順をスキップできます。

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します

1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。

1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

1. 左側のサイドバーで、**[(+) 新規]** タスクを選択し、**[クラスター]** を選択します。

1. **[新しいクラスター]** ページで、次の設定を使用して新しいクラスターを作成します。
    - **クラスター名**: "ユーザー名の" クラスター (既定のクラスター名)**
    - **ポリシー**:Unrestricted
    - **クラスター モード**: 単一ノード
    - **アクセス モード**: 単一ユーザー (*自分のユーザー アカウントを選択*)
    - **Databricks Runtime のバージョン**: 13.3 LTS (Spark 3.4.1、Scala 2.12) 以降
    - **Photon Acceleration を使用する**: 選択済み
    - **ノードの種類**: Standard_DS3_v2
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

    > **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## ノートブックを作成してデータを取り込む

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。 **[接続]** ドロップダウン リストで、まだ選択されていない場合はクラスターを選択します。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

2. ノートブックの最初のセルに次のコードを入力します。このコードは、"シェル" コマンドを使用して、GitHub からクラスターで使用されるファイル システムにデータ ファイルをダウンロードします。**

     ```python
    %sh
    rm -r /dbfs/FileStore
    mkdir /dbfs/FileStore
    wget -O /dbfs/FileStore/sample_sales.csv https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv
     ```

3. セルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行を行います。 そして、コードによって実行される Spark ジョブが完了するまで待ちます。
   
## GitHub リポジトリとAzure DevOps プロジェクトを設定する

GitHub リポジトリを Azure DevOps プロジェクトに接続したら、リポジトリに加えられた変更をトリガーする CI パイプラインを設定できます。

1. [GitHub アカウント](https://github.com/)に移動し、プロジェクトの新しいリポジトリを作成します。

2. `git clone` を使用して、ローカル コンピューターにリポジトリを複製します。

3. [CSV ファイル](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales.csv)をローカル リポジトリにダウンロードし、変更内容をコミットします。

4. CSV ファイルの読み取りとデータ変換の実行に使用する [Databricks ノートブック](https://github.com/MicrosoftLearning/mslearn-databricks/raw/main/data/sample_sales_notebook.dbc)をダウンロードします。 変更をコミットします。

5. [Azure DevOps ポータル](https://azure.microsoft.com/en-us/products/devops/)に移動し、新しいプロジェクトを作成します。

6. Azure DevOps プロジェクトで、 **Repos** セクションに移動し、**[Import]** を選択して GitHub リポジトリに接続します。

7. 左側のバーで、**[プロジェクト設定] > [サービス接続]** に移動します。

8. **[サービス接続の作成]** を選択し、次に **[Azure Resource Manager]** を選択します。

9. **[認証方法]** ウィンドウで、 **[ワークロード ID フェデレーション (自動)]** を選択します。 [**次へ**] を選択します。

10. **[スコープ レベル]** で **[サブスクリプション]** を選択します。 Databricks ワークスペースを作成したサブスクリプションとリソース グループを選択します。

11. サービス接続の名前を入力し、**[すべてのパイプラインへのアクセス許可を与える]** オプションをオンにします。 **[保存]** を選択します。

これで、DevOps プロジェクトが Databricks ワークスペースにアクセスできるようになり、パイプラインに接続できるようになります。

## CI/CD パイプラインを構成する

1. 左側のバーで、**Pipelines** に移動し **[パイプラインの作成]** を選択します。

2. ソース コードの場所として **[GitHub]** を選択し、リポジトリを選択してください。

3. **[パイプラインの構成]** ウィンドウで、**[スタート パイプライン]** を選択し、CI パイプラインに次の YAML 構成を使用します。

```yaml
trigger:
- main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: UsePythonVersion@0
  inputs:
    versionSpec: '3.x'
    addToPath: true

- script: |
    pip install databricks-cli
  displayName: 'Install Databricks CLI'

- script: |
    databricks fs cp dbfs:/FileStore/sample_sales.csv .
  displayName: 'Download Sample Data from DBFS'

- script: |
    python -m unittest discover -s tests
  displayName: 'Run Unit Tests'
```

4. **[保存して実行]** を選択します。

この YAML ファイルは、リポジトリの `main` ブランチへの変更によってトリガーされる CI パイプラインを設定します。 パイプラインは Python 環境を設定し、Databricks CLI をインストールし、Databricks ワークスペースからサンプル データをダウンロードして、Python 単体テストを実行します。 これは CI ワークフローの一般的なセットアップです。

## CI/CD パイプラインを構成する

1. 左側のバーで、**Pipelines > Releases** に移動し **[リリースの作成]** を選択します。

2. 成果物ソースとしてビルド パイプラインを選択します。

3. ステージを追加し、Azure Databricks にデプロイするタスクを構成します。

```yaml
stages:
- stage: Deploy
  jobs:
  - job: DeployToDatabricks
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UsePythonVersion@0
      inputs:
        versionSpec: '3.x'
        addToPath: true

    - script: |
        pip install databricks-cli
      displayName: 'Install Databricks CLI'

    - script: |
        databricks workspace import_dir /path/to/notebooks /Workspace/Notebooks
      displayName: 'Deploy Notebooks to Databricks'
```

このパイプラインを実行する前に、`/path/to/notebooks` をリポジトリ内のノートブックがあるディレクトリへのパスに置き換え、`/Workspace/Notebooks` を、Databricks ワークスペースでノートブックを保存したいファイル パスに置き換えます。

4. **[保存して実行]** を選択します。

## パイプラインを実行する

1. ローカル レポジトリで、`sample_sales.csv` ファイルの末尾に次の行を追加します。

     ```sql
    2024-01-01,ProductG,1,500
     ```

2. 変更をコミットし、GitHub リポジトリにプッシュする。

3. リポジトリ内の変更によって CI パイプラインがトリガーされます。 パイプラインの実行が正常に完了することを確認します。

4. リリース パイプラインで新しいリリースを作成し、ノートブックを Databricks にデプロイします。 ノートブックが Databricks ワークスペースで正常にデプロイされ、実行されていることを確認します。

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。







