---
lab:
  title: テーブルを Unity Catalog にアップグレードする
  description: データ ストレージの要件に基づくさまざまな移行戦略を使って、Hive メタストアから Unity Catalog にテーブルをアップグレードする方法を学びます。 ユニバーサル テーブルの移行には CTAS を使い、データを再構築するためにアップグレード プロセスの間に変換を適用して、Delta テーブルのディープ クローンを練習します。 また、SYNC コマンドと Catalog Explorer の UI を使って、データを移動せずに外部テーブルをアップグレードする方法についても調べます。
  duration: 30 minutes
  level: 300
  islab: true
  primarytopics:
    - Azure Databricks
    - Azure Portal
---

# テーブルを Unity Catalog にアップグレードする

Unity Catalog には、Azure Databricks のデータ資産に対応する集中ガバナンス ソリューションが用意されています。 Unity Catalog は、Hive メタストアによって提供される基本的なテーブル管理を基にして構築された、きめ細かいアクセス制御、データ系列の自動追跡、ワークスペース間のデータ共有の機能を備えています。

この演習では、既存のテーブルを Hive メタストアから Unity Catalog にアップグレードする方法を学びます。 SQL コマンドと Azure Databricks のユーザー インターフェイスを使って、既存のデータ構造を分析し、移行手法を適用し、変換オプションを評価して、データを移動せずにメタデータをアップグレードします。

この演習の所要時間は約 **30** 分です。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

## 開始する前に

メタストアをサポートするには、アカウント管理者の機能とクラウド ストレージ リソースが必要です。 カタログの作成と管理には、メタストア管理機能も必要です。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 

十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、[Azure portal で、Azure Databricks ワークスペースを対話形式で作成](https://learn.microsoft.com/azure/databricks/getting-started/#--create-an-azure-databricks-workspace)してみてください。

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
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの「[Unity Catalog とは](https://learn.microsoft.com/azure/databricks/data-governance/unity-catalog/)」の記事を確認してください。

## Azure Databricks ワークスペースを開く

1. Azure portal で、スクリプトによって作成された **msl-*xxxxxxx*** リソース グループ (または既存の Azure Databricks ワークスペースを含むリソース グループ) に移動します。

2. Azure Databricks Service リソース (セットアップ スクリプトを使って作成した場合は、**databricks-*xxxxxxx*** という名前) を選択します。

3. Azure Databricks ワークスペースの [**概要**] ページで、[**ワークスペースの起動**] ボタンを使用して、新しいブラウザー タブで Azure Databricks ワークスペースを開きます。サインインを求められた場合はサインインします。

## Catalog Explorer の機能を確認する
1. ワークスペースの左側で **[カタログ]** を選びます。
2. ワークスペースと同じ名前のカタログが自動的に作成されていることに注意してください。
3. そのカタログを展開して、既定のスキーマに注目します。 この演習で後ほど、テーブルをその既定のスキーマに移行します。
4. **hive_metastore** カタログも自動的に作成されることに注意してください。 
5. **samples** カタログを開き、**bakehouse** スキーマに注目します。 それを展開して、sales_customers という名前のテーブルに注目します。
6. この演習では、samples.bakehouse.sales_customers テーブルのサンプル データを使って、hive_metastore にテーブルを作成して設定します。 その後、そのテーブルを Unity Catalog にアップグレードするためのオプションを調べます。

## ノートブックを作成する
ノートブックを使って、さまざまなテーブル アップグレード手法がわかる SQL コマンドを実行します。

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
   
2. 既定のノートブック名 (**Untitled Notebook *[date]***) を `Upgrade tables to Unity Catalog` に変更し、**[接続]** ドロップダウン リストで **[サーバーレス]** を選びます。

## hive_metastore オブジェクトを作成する
サンプル データを hive_metastore テーブルに読み込んで、Unity Catalog に移行できるようにします。

1. 新しいセルを追加し、次のコードを実行してスキーマと設定されたテーブルを hive_metastore に作成してから、新しいテーブルに行があることを確認します。

    ```sql
    CREATE SCHEMA IF NOT EXISTS hive_metastore.bakehouse;

    CREATE TABLE hive_metastore.bakehouse.sales_customers AS
    SELECT * FROM samples.bakehouse.sales_customers;

    SELECT * FROM hive_metastore.bakehouse.sales_customers;
    ```

2.  新しいセルで次を実行して、ワークスペースと同じ名前の Unity Catalog に既定のカタログとスキーマを設定します。
    
    **注**: ワークスペースの名前は、Azure Databricks ワークスペースの右上に表示されています。 下の USE CATALOG ステートメントの `<workspace_catalog>` を、ご自分のワークスペースの名前に置き換えます。 ワークスペース名に `-` が含まれる場合は、それを `_` に置き換えます。

    ```sql
    USE CATALOG <workspace_catalog>;
    USE SCHEMA default;
    SELECT current_catalog(), current_schema()
    ```  

## アップグレード方法の概要

テーブルをアップグレードするにはいくつかの方法がありますが、どの方法を選ぶかは主にテーブル データの処理方法によって決まります。 テーブル データをそのままにしたい場合は、結果のアップグレードされたテーブルは外部テーブルになります。 テーブル データを Unity Catalog メタストアに移動したい場合は、結果のテーブルはマネージド テーブルになります。

### テーブル データを Unity Catalog メタストアに移動する

この方法では、テーブル データはそれが存在する場所からコピー先スキーマのマネージド データ ストレージ領域にコピーされます。 結果は、Unity Catalog メタストア内のマネージド Delta テーブルになります。

#### テーブルを複製する

ソース テーブルが Delta の場合は、テーブルを複製するのが最適です。 使い方が簡単で、メタデータがコピーされ、データをコピーするか (ディープ クローン) そのままにするか (シャロー クローン) を選択できます。

1. 新しいセルを追加し、次のコードを実行してソース テーブルの形式を調べます。

    ```sql
    DESCRIBE EXTENDED hive_metastore.bakehouse.sales_customers;
    ```

   *Provider* 行ではソースが Delta テーブルであることが示され、*Location* 行ではテーブルが DBFS に格納されていることが示されていることに注意してください。

2. 新しいセルを追加して次のコードを実行し、ディープ クローン操作を実行します。 これにより、Unity Catalog の既定のカタログにテーブルが作成されます。 新しいセルを追加して次のコードを実行し、hive_metastore を Unity Catalog の新しいテーブルに複製します。 新しいテーブルは、ワークスペースと同じ名前を持つカタログの既定のスキーマに作成されます。  

    ```sql
    CREATE OR REPLACE TABLE sales_customers_clone
      DEEP CLONE hive_metastore.bakehouse.sales_customers;
    ```

3. Catalog Explorer でカタログを表示して、複製されたテーブルを確認します。
   - ワークスペースの左側にある **[カタログ]** アイコンを選びます
   - ワークスペースと同じ名前のカタログを展開します
   - **default** スキーマを展開します
   - **sales_customer** テーブルが **sales_customer_clone** として複製されていることに注意してください

#### CREATE TABLE AS SELECT (CTAS)

CTAS の使用はどのような場合でも適用できる手法であり、`SELECT` ステートメントの出力を基にして新しいテーブルを作成します。 データは常にコピーされますが、メタデータはコピーされません。

1. ノートブックに戻ります。 新しいセルを追加して次のコードを実行し、CTAS を使ってテーブルをコピーします。

    ```sql
    CREATE TABLE sales_customers_ctas AS 
    SELECT * 
    FROM hive_metastore.bakehouse.sales_customers;
    ```

2. 新しいセルを追加して次のコードを実行し、テーブルが作成されたことを確認します。

    ```sql
    SHOW TABLES IN default;
    ```

#### アップグレードの間に変換を適用する

CTAS を使うと、データをコピー中に変換できます。 テーブルを Unity Catalog に移行するときは、テーブルの構造が組織のビジネス要件にまだ対応しているかどうかを検討するのに絶好の機会です。

1. 新しいセルを追加して次のコードを実行し、テーブルの変換されたバージョンを作成します。

    ```sql
    CREATE TABLE sales_customers_transformed AS 
    SELECT
      customerID as ID,
      first_name as first_name,
      last_name as last_name
    FROM hive_metastore.bakehouse.sales_customers;
    ```

2. Catalog Explorer でカタログを表示して、新しいテーブルを確認します。
   - ワークスペースの左側にある **[カタログ]** アイコンを選びます
   - ワークスペースと同じ名前のカタログを展開します
   - **default** スキーマを展開します
   - **[テーブル]** を展開します
   - **sales_customer_transformed** テーブルを選び、ID 列名が customerID から変更されていることを確認します。 テーブルが表示されない場合は、Catalog Explorer の上部にある更新ボタンを選びます。

## 外部テーブルをアップグレードする (例)

**注**: 外部テーブルにアクセスできない可能性があるため、これは環境で実行できることの例です。

外部テーブルをアップグレードする際、データの場所が規制要件によって定められている、データ形式を Delta に変更できない、大規模なデータセットを移動する時間とコストを回避したい、といった一部のユース ケースでは、データをその場に残しておくことが求められる場合があります。

### SYNC コマンドを使用する

`SYNC` SQL コマンドを使うと、データを移動せずに、Hive メタストアの外部テーブルを Unity Catalog の外部テーブルにアップグレードできます。

### Catalog Explorer を使用したテーブルのアップグレード

Catalog Explorer のユーザー インターフェイスを使ってテーブルをアップグレードすることもできます。

1. 左側のカタログ アイコンを選びます
2. **hive_metastore** を展開します
3. Hive メタストアの bakehouse スキーマを展開します
5. **sales_customers** テーブルを選んで、**[アップグレード]** をクリックします
6. アップグレード先のカタログとスキーマを選びます
7. 必要に応じてアップグレード オプションを構成します

この演習では、バックグラウンドで `SYNC` を使用するため、アップグレードを実際に実行する必要はありません。

## クリーンアップ

Unity Catalog テーブルのアップグレードを調べ終わったら、Azure で不要なコストが発生しないよう、作成したリソースを削除してかまいません。

Unity Catalog の確認が終わったら、不要な Azure コストが発生しないように、作成したリソースを削除できます。

1. Azure Databricks ワークスペースのブラウザー タブを閉じ、Azure portal に戻ります。
2. Azure portal の **[ホーム]** ページで、**[リソース グループ]** を選択します。
3. (管理対象リソース グループではなく) Azure Databricks ワークスペースを含むリソース グループを選択します。
4. リソース グループの **[概要]** ページの上部で、**[リソース グループの削除]** を選択します。 
5. リソース グループ名を入力して、削除することを確認し、**[削除]** を選択します。

    数分後に、リソース グループと、それに関連付けられているマネージド ワークスペース リソース グループが削除されます。