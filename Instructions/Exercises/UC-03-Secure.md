---
lab:
  title: Unity Catalog でのデータのセキュリティ保護
---

# Unity Catalog でのデータのセキュリティ保護

データ セキュリティは、組織がデータ レイクハウスで機密情報を管理するうえで重要な懸念事項です。 データ チームがさまざまな役割や部門で共同作業を行い、適切なユーザーの適切なデータへのアクセスを確保しながら、機密情報を不正アクセスから保護することは、ますます複雑になります。

このハンズオン ラボでは、基本的なアクセス制御を超える 2 つの強力な Unity Catalog セキュリティ機能を紹介します。

1. **行フィルター処理と列マスク**:ユーザーのアクセス許可に基づいて特定の行を非表示にするか、列の値をマスクして、テーブル レベルで機密データを保護する方法について学習します
2. **動的ビュー**:グループ メンバーシップとアクセス レベルに基づいて、ユーザーが表示できるデータを自動的に調整するインテリジェントなビューを作成します

このラボは完了するまで、約 **45** 分かかります。

> **注**: Azure Databricks ユーザー インターフェイスは継続的な改善の対象となります。 この演習の手順が記述されてから、ユーザー インターフェイスが変更されている場合があります。

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
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待機している間に、Microsoft Learn の [Unity Catalog でのセキュリティとアクセスの制御の実装](https://learn.microsoft.com/training/modules/implement-security-unity-catalog)に関する学習モジュールをご確認ください。

## カタログを作成する

1. メタストアにリンクされているワークスペースにログインします。
2. 左側のメニューから **[カタログ]** を選択します。
3. **[クイック アクセス]** の下にある **[カタログ]** を選択します。
4. **[カタログの作成]** を選択します。
5. **[新しいカタログの作成]** ダイアログで、**[カタログ名]** を入力し、作成するカタログの **[タイプ]** を選択します。**標準**カタログ。
6. マネージド**保存場所**を指定します。

## ノートブックを作成する

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
   
1. 既定のノートブック名 (**無題のノートブック *[日付]***) を「`Secure data in Unity Catalog`」に変更し、**[接続]** ドロップダウン リストで **[サーバーレス クラスター]** を選択します (まだ選択されていない場合)。  **[サーバーレス]** は既定で有効になっていることに注意してください。

1. このコースの作業環境を構成するために、ご自分のノートブックの新しいセルに次のコードをコピーして実行します。 また、`USE` ステートメントを使用して、既定のカタログが特定のカタログに、スキーマが次に示すスキーマ名に設定されます。

```SQL
USE CATALOG `<your catalog>`;
USE SCHEMA `default`;
```

1. 次のコードを実行し、現在のカタログが一意のカタログ名に設定されていること、および現在のスキーマが **default** であることを確認します。

```sql
SELECT current_catalog(), current_schema()
```

## 列マスクと行フィルター処理を使用して列と行を保護する

### テーブル Customers を作成する

1. 次のコードを実行して、**default**スキーマに **customers** テーブルを作成します。

```sql
CREATE OR REPLACE TABLE customers AS
SELECT *
FROM samples.tpch.customer;
```

2. クエリを実行して、**default** スキーマの **customers** テーブルから *10* 行を表示します。 テーブルには、**c_name**、**c_phone**、**c_mktsegment** などの情報が含まれていることに注目してください。
   
```sql
SELECT *
FROM customers  
LIMIT 10;
```

### 列マスクを実行する関数を作成する

より詳細なヘルプについては、[行フィルターと列マスクを使用して機密性の高いテーブル データをフィルター処理する方法](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/filters-and-masks/)に関するドキュメントを参照してください。

1. **phone_mask** という名前の関数を作成し、ユーザーが ('admin') グループのメンバーでない場合は、`is_account_group_member` 関数を使用して **customers** テーブルの **c_phone ** 列をリダクトします。 ユーザーがメンバーでない場合は、**phone_mask** 関数で、文字列 *REDACTED PHONE NUMBER* が返されます。

```sql
CREATE OR REPLACE FUNCTION phone_mask(c_phone STRING)
  RETURN CASE WHEN is_account_group_member('metastore_admins') 
    THEN c_phone 
    ELSE 'REDACTED PHONE NUMBER' 
  END;
```

2. `ALTER TABLE` ステートメントを使用して、列マスク関数 **phone_mask** を **customers** テーブルに適用します。

```sql
ALTER TABLE customers 
  ALTER COLUMN c_phone 
  SET MASK phone_mask;
```

3. 次のクエリを実行して、列マスクが適用された **customers** テーブルを表示します。 **c_phone** 列に、値 *REDACTED PHONE NUMBER* が表示されていることを確認します。

```sql
SELECT *
FROM customers
LIMIT 10;
```

### 行フィルター処理を実行する関数を作成する

より詳細なヘルプについては、[行フィルターと列マスクを使用して機密性の高いテーブル データをフィルター処理する方法](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/filters-and-masks/)に関するドキュメントを参照してください。

1. 次のコードを実行して、**customers** テーブル内の行の合計数をカウントします。 テーブルに 750,000 行のデータが含まれていることを確認します。

```sql
SELECT count(*) AS TotalRows
FROM customers;
```

2. **nation_filter ** という名前の関数を作成し、ユーザーが ('admin') グループのメンバーでない場合に、`is_account_group_member` 関数を使用して **customers** テーブル内の **c_nationkey** をフィルター処理します。 この関数は、**c_nationkey** が *21* と等しい行のみを返します。

    より詳細なヘルプについては、[if 関数](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/functions/if) に関するドキュメントを参照してください。

```sql
CREATE OR REPLACE FUNCTION nation_filter(c_nationkey INT)
  RETURN IF(is_account_group_member('admin'), true, c_nationkey = 21);
```

3. `ALTER TABLE` ステートメントを使用して、関数行フィルター関数 `nation_filter` を **customers** テーブルに適用します。

```sql
ALTER TABLE customers 
SET ROW FILTER nation_filter ON (c_nationkey);
```

4. 管理者ではないユーザーの行をフィルター処理で除外しているため、次のクエリを実行して **customers** テーブル内の行数をカウントします。 *29,859* 行のみを表示できることを確認します ("ここで、c_nationkey = 21 です")。** 

```sql
SELECT count(*) AS TotalRows
FROM customers;
```

5. 次のクエリを実行して、**customers** テーブルを表示します。 

    最終的なテーブルを確認します。
    - **c_phone** 列をリダクトし、
    - "管理者" ではないユーザーの **c_nationkey** 列に基づいて行をフィルター処理します。**

```sql
SELECT *
FROM customers;
```

## 動的ビューを使用して列と行を保護する

### テーブル Customers_new を作成する

1. 次のコードを実行して、**default** スキーマに **customers_new** テーブルを作成します。

```sql
CREATE OR REPLACE TABLE customers_new AS
SELECT *
FROM samples.tpch.customer;
```

2. クエリを実行して、**default** スキーマの **customers_new** テーブルから *10* 行を表示します。 テーブルには、**c_name**、**c_phone**、**c_mktsegment** などの情報が含まれていることに注目してください。

```sql
SELECT *
FROM customers_new
LIMIT 10;
```

### 動的ビューを作成する

次の変換を使用して、**customers_new** テーブル データの処理されたビューを表示する **vw_customers** という名前のビューを作成してみましょう。

- **customers_new** テーブルからすべての列を選択します。

- `is_account_group_member('admins')` に属している場合を除いて、**c_phone** 列のすべての値を *REDACTED PHONE NUMBER* にリダクトします
    - ヒント：`SELECT` 句で `CASE WHEN` ステートメントを使用します。

- `is_account_group_member('admins')` に属している場合を除いて、**c_nationkey** が *21* と等しい行を制限します。
    - ヒント：`WHERE` 句で `CASE WHEN` ステートメントを使用します。

```sql
-- Create a movies_gold view by redacting the "votes" column and restricting the movies with a rating below 6.0

CREATE OR REPLACE VIEW vw_customers AS
SELECT 
  c_custkey, 
  c_name, 
  c_address, 
  c_nationkey,
  CASE 
    WHEN is_account_group_member('admins') THEN c_phone
    ELSE 'REDACTED PHONE NUMBER'
  END as c_phone,
  c_acctbal, 
  c_mktsegment, 
  c_comment
FROM customers_new
WHERE
  CASE WHEN
    is_account_group_member('admins') THEN TRUE
    ELSE c_nationkey = 21
  END;
```

3. **vw_customers** ビューにデータを表示します。 **c_phone** 列がリダクトされていることを確認します。 管理者でない限り、**c_nationkey** が *21* と等しいことを確認します。

```sql
SELECT * 
FROM vw_customers;
```

6. **vw_customers** ビューの行数をカウントします。 ビューに *29,859* 行が含まれていることを確認します。

```sql
SELECT count(*)
FROM vw_customers;
```

### ビューへのアクセス権許可を発行する

1. **vw_customers** ビューを表示するため "アカウント ユーザー" に対する許可を発行しましょう。

**注:** また、カタログとスキーマへのアクセス権をユーザーに提供する必要もあります。 この共有トレーニング環境では、カタログへのアクセス権を他のユーザーに付与することはできません。

```sql
GRANT SELECT ON VIEW vw_customers TO `account users`
```

2. `SHOW` ステートメントを使用して、**vw_customers** ビューに影響を与えるすべての特権 (継承、拒否、付与) を表示します。 **プリンシパル**列に*アカウント ユーザー*が含まれていることを確認します。

    ヘルプについては、[SHOW GRANTS](https://learn.microsoft.com/en-us/azure/databricks/sql/language-manual/security-show-grant) に関するドキュメントを参照してください。

```sql
SHOW GRANTS ON vw_customers
```