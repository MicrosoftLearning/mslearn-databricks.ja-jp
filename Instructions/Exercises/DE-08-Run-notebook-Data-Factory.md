---
lab:
  title: Azure Data Factory を使用して Azure Databricks ノートブックを自動化する
---

# Azure Data Factory を使用して Azure Databricks ノートブックを自動化する

Azure Databricks でノートブックを使用して、データ ファイルの処理やテーブルへのデータの読み込みなどのデータ エンジニアリング タスクを実行できます。 これらのタスクをデータ エンジニアリング パイプラインの一部として調整する必要がある場合は、Azure Data Factory を使用できます。

この演習の所要時間は約 **40** 分です。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。***PowerShell*** 環境を選択します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: *Bash* 環境を使用するクラウド シェルを以前に作成した場合は、それを ***PowerShell*** に切り替えます。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#10530;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

4. PowerShell のペインで、次のコマンドを入力して、リポジトリを複製します。

    ```
    rm -r mslearn-databricks -f
    git clone https://github.com/MicrosoftLearning/mslearn-databricks
    ```

5. リポジトリをクローンした後、次のコマンドを入力して **setup.ps1** スクリプトを実行します。これにより、使用可能なリージョンに Azure Databricks ワークスペースがプロビジョニングされます。

    ```
    ./mslearn-databricks/setup.ps1
    ```

6. メッセージが表示された場合は、使用するサブスクリプションを選択します (これは、複数の Azure サブスクリプションへのアクセス権を持っている場合にのみ行います)。
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、「[Azure Data Factory とは何ですか](https://docs.microsoft.com/azure/data-factory/introduction)」を確認してください。

## Azure Data Factory リソースを作成する

Azure Databricks ワークスペースに加えて、サブスクリプションに Azure Data Factory リソースをプロビジョニングする必要があります。

1. Azure portal でクラウド シェル ペインを閉じて、セットアップ スクリプトによって作成された ***msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します。
1. ツール バーで **[+ 作成]** を選択し、`Data Factory` を検索します。 その後、次の設定を使用して新しい **Data Factory** リソースを作成します。
    - **[サブスクリプション]**: *該当するサブスクリプション*
    - **リソース グループ**: msl-*xxxxxxx* (または、既存の Azure Databricks ワークスペースを含むリソース グループ)
    - **[名前]**: *一意の名前。**adf-xxxxxxx** など*
    - **[リージョン]**: *Azure Databricks ワークスペースと同じリージョン (このリージョンが一覧にない場合は他の使用可能なリージョン)*
    - **[バージョン]**: V2
1. 新しいリソースが作成されたら、リソース グループに Azure Databricks ワークスペースと Azure Data Factory リソースの両方が含まれていることを確認します。

## ノートブックを作成する

Azure Databricks ワークスペースにノートブックを作成して、さまざまなプログラミング言語で記述されたコードを実行できます。 この演習では、ファイルからデータを取り込んで Databricks File System (DBFS) のフォルダーに保存する簡単なノートブックを作成します。

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します
1. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。
1. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

    > **ヒント**: Databricks ワークスペース ポータルを使用すると、さまざまなヒントと通知が表示される場合があります。 これらは無視し、指示に従ってこの演習のタスクを完了してください。

1. 次に、Azure Databricks ワークスペース ポータルを表示し、左側のサイド バーに、実行できるさまざまなタスクのアイコンが含まれていることに注目します。
1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
1. 既定のノートブック名 (**無題のノートブック *[日付]***) を「`Process Data`」に変更します。
1. ノートブックの最初のセルには、このノートブックによるデータ保存先フォルダーの変数を設定するために、次のコードを入力します (ただし、実行しないでください)。

    ```python
   # Use dbutils.widget define a "folder" variable with a default value
   dbutils.widgets.text("folder", "data")
   
   # Now get the parameter value (if no value was passed, the default set above will be used)
   folder = dbutils.widgets.get("folder")
    ```

1. 既存のコード セルの下で、 **[+]** アイコンを使用して新しいコード セルを追加します。 その後、新しいセルに、データをダウンロードしてフォルダーに保存するために、次のコードを入力します (ただし、実行しないでください)。

    ```python
   import urllib3
   
   # Download product data from GitHub
   response = urllib3.PoolManager().request('GET', 'https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/products.csv')
   data = response.data.decode("utf-8")
   
   # Save the product data to the specified folder
   path = "dbfs:/{0}/products.csv".format(folder)
   dbutils.fs.put(path, data, True)
    ```

1. 左側のサイドバーで **[ワークスペース]** を選択し、"**データの処理**" ノートブックが表示されていることを確認します。 Azure Data Factory を使用して、パイプラインの一部としてノートブックを実行します。

    > **注**:ノートブックには、必要とするほぼすべてのデータ処理ロジックを含めることができます。 この簡単な例は、主要な原則を示すために設計されています。

## Azure Databricks と Azure Data Factory の統合を有効にする

Azure Data Factory パイプラインから Azure Databricks を使用するには、Azure Databricks ワークスペースへのアクセスを可能にするリンク サービスを Azure Data Factory 内に作成する必要があります。

### アクセス トークンを生成する

1. Azure Databricks ポータルの右上のメニュー バーでユーザー名を選んで、ドロップダウンから **[ユーザー設定]** を選びます。
1. **[ユーザー設定]** ページで、 **[開発者]** を選びます。 次に、 **[アクセス トークン]** の横にある **[管理]** を選びます。
1. **[新しいトークンの生成]** を選び、*Data Factory* というコメントと空の有効期間 (トークンは期限切れになりません) を指定して、新しいトークンを生成します。 **トークンが表示されたら、[完了] を選ぶ<u>前に</u>それをコピーする** ことに注意してください。
1. コピーしたトークンをテキスト ファイルに貼り付けておくと、この演習の後半で役立ちます。

## パイプラインを使用して Azure Databricks ノートブックを実行する

リンク サービスを作成し終わったので、それをパイプラインで使用して、前に表示したノートブックを実行できます。

### パイプラインを作成する

1. Azure Data Factory Studio のナビゲーション ウィンドウで、 **[作成]** を選択します。
2. **[作成]** ページの **[ファクトリのリソース]** ペインで、 **[+]** アイコンを使用して**パイプライン**を追加します。
3. 新しいパイプラインの **[プロパティ]** ペインで、名前を `Process Data with Databricks` に変更します。 ツール バーの右端にある **[プロパティ]** ボタン ( **&#128463;<sub>*</sub>** のような外観) を使用して、 **[プロパティ]** ペインを非表示にします。
4. **[アクティビティ]** ペインで **[Databricks]** を展開し、パイプライン デザイナー画面に **[ノートブック]** アクティビティをドラッグします。
5. 新しい **Notebook1** アクティビティが選択された状態で、下部のペインで次のプロパティを設定します。
    - **全般**:
        - **名前**: `Process Data`
    - **Azure Databricks**:
        - **Databricks のリンク サービス**: "先ほど作成した **AzureDatabricks** リンク サービスを選択します"**
    - **設定**:
        - **ノートブック パス**: "**Users/<ユーザー名>** フォルダーを参照し、**Process Data** ノートブックを選択します"**
        - **基本パラメーター**: *値 `product_data` を含む `folder` という名前の新しいパラメーターを追加します*
6. パイプライン デザイナー画面の上にある **[検証]** ボタンを使用して、パイプラインを検証します。 次に、 **[すべて発行]** ボタンを使用して発行 (保存) します。

### Azure Data Factory でリンク サービスを作成する

1. Azure portal に戻り、**msl-*xxxxxxx*** リソース グループで、Azure Data Factory リソース **adf*xxxxxxx*** を選択します。
2. **[概要]** ページで、 **[スタジオの起動]** を選択して Azure Data Factory Studio を開きます。 メッセージが表示されたらサインインします。
3. Azure Data Factory Studio で、 **[>>]** アイコンを使用して左側のナビゲーション ウィンドウを展開します。 次に、 **[管理]** ページを選択します。
4. **[管理]** ページの **[リンク サービス]** タブで、 **[+ 新規]** を選択して、新しいリンク サービスを追加します。
5. **[新しいリンク サービス]** ペインで、上部にある **[コンピューティング]** タブを選択します。 次に、 **[Azure Databricks]** を選択します。
6. 続けて、次の設定を使用してリンク サービスを作成します。
    - **名前**: `AzureDatabricks`
    - **説明**: `Azure Databricks workspace`
    - **統合ランタイム経由で接続する**: AutoResolveIntegrationRuntime
    - **アカウントの選択方法**: Azure サブスクリプションから
    - **Azure サブスクリプション**: "サブスクリプションを選択します"**
    - **Databricks ワークスペース**: "**databricksxxxxxxx** ワークスペースを選択します"**
    - **クラスターの選択**: 新しいジョブ クラスター
    - **Databrick ワークスペース URL**: "Databricks ワークスペース URL に自動的に設定されます"**
    - **認証の種類**: アクセス トークン
    - **アクセス トークン**: "アクセス トークンを貼り付けます"**
    - **クラスターのバージョン**: 13.3 LTS (Spark 3.4.1, Scala 2.12)
    - **クラスターのノード タイプ**: Standard_D4ds_v5
    - **Python バージョン**: 3
    - **ワーカー オプション**: 固定
    - **ワーカー数**: 1

### パイプラインを実行する

1. パイプライン デザイナー画面の上にある **[トリガーの追加]** を選択し、 **[今すぐトリガー]** を選択します。
2. **[パイプライン実行]** ペインで **[OK]** を選択してパイプラインを実行します。
3. 左側のナビゲーション ウィンドウで、 **[監視]** を選択し、 **[パイプラインの実行]** タブの **Process Data with Databricks** パイプラインを観察します。Spark クラスターを動的に作成してノートブックを実行するため、実行に時間がかかる場合があります。 **[パイプラインの実行]** ページの **[&#8635; 最新の情報に更新]** ボタンを使用して、状態を更新できます。

    > **注**: パイプラインが失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンで、ジョブ クラスターを作成するためのサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./setup.ps1 eastus`

4. 実行が成功したら、その名前を選択して実行の詳細を表示します。 次に、 **[Process Data with Databricks]** ページの **[アクティビティの実行]** セクションで、**Process Data** アクティビティを選択し、その [出力] アイコンを使用してアクティビティからの出力 JSON を表示します。これは次のようになります。******

    ```json
    {
        "runPageUrl": "https://adb-..../run/...",
        "effectiveIntegrationRuntime": "AutoResolveIntegrationRuntime (East US)",
        "executionDuration": 61,
        "durationInQueue": {
            "integrationRuntimeQueue": 0
        },
        "billingReference": {
            "activityType": "ExternalActivity",
            "billableDuration": [
                {
                    "meterType": "AzureIR",
                    "duration": 0.03333333333333333,
                    "unit": "Hours"
                }
            ]
        }
    }
    ```

## クリーンアップ

Azure Databricks を調べ終わったら、不要な Azure コストがかからないように、また、サブスクリプションの容量を解放するために、作成したリソースを削除することができます。
