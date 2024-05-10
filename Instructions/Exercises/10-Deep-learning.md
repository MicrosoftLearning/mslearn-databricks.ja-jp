---
lab:
  title: ディープ ラーニング モデルをトレーニングする
---

# ディープ ラーニング モデルをトレーニングする

この演習では、**PyTorch** ライブラリを使用して、Azure Databricks でディープ ラーニング モデルをトレーニングします。 次に、**Horovod** ライブラリを使用して、クラスター内の複数のワーカー ノードにディープ ラーニング トレーニングを配布します。

この演習の所要時間は約 **45** 分です。

## 開始する前に

管理レベルのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure Databricks ワークスペースをプロビジョニングする

> **ヒント**: 既に Azure Databricks ワークスペースがある場合は、この手順をスキップして、既存のワークスペースを使用できます。

この演習には、新しい Azure Databricks ワークスペースをプロビジョニングするスクリプトが含まれています。 このスクリプトは、この演習で必要なコンピューティング コアに対する十分なクォータが Azure サブスクリプションにあるリージョンに、*Premium* レベルの Azure Databricks ワークスペース リソースを作成しようとします。また、使用するユーザー アカウントのサブスクリプションに、Azure Databricks ワークスペース リソースを作成するための十分なアクセス許可があることを前提としています。 十分なクォータやアクセス許可がないためにスクリプトが失敗した場合は、Azure portal で、Azure Databricks ワークスペースを対話形式で作成してみてください。

1. Web ブラウザーで、`https://portal.azure.com` の [Azure portal](https://portal.azure.com) にサインインします。
2. ページ上部の検索バーの右側にある **[\>_]** ボタンを使用して、Azure portal に新しい Cloud Shell を作成します。メッセージが表示されたら、***PowerShell*** 環境を選んで、ストレージを作成します。 次に示すように、Azure portal の下部にあるペインに、Cloud Shell のコマンド ライン インターフェイスが表示されます。

    ![Azure portal と Cloud Shell のペイン](./images/cloud-shell.png)

    > **注**: 前に *Bash* 環境を使ってクラウド シェルを作成している場合は、そのクラウド シェル ペインの左上にあるドロップダウン メニューを使って、***PowerShell*** に変更します。

3. ペインの上部にある区分線をドラッグして Cloud Shell のサイズを変更したり、ペインの右上にある **&#8212;** 、 **&#9723;** 、**X** アイコンを使用して、ペインを最小化または最大化したり、閉じたりすることができます。 Azure Cloud Shell の使い方について詳しくは、[Azure Cloud Shell のドキュメント](https://docs.microsoft.com/azure/cloud-shell/overview)をご覧ください。

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
7. スクリプトの完了まで待ちます。通常、約 5 分かかりますが、さらに時間がかかる場合もあります。 待っている間に、Azure Databricks ドキュメントの「[分散トレーニング](https://learn.microsoft.com/azure/databricks/machine-learning/train-model/distributed-training/)」の記事を確認してください。

## クラスターの作成

Azure Databricks は、Apache Spark "クラスター" を使用して複数のノードでデータを並列に処理する分散処理プラットフォームです。** 各クラスターは、作業を調整するドライバー ノードと、処理タスクを実行するワーカー ノードで構成されています。 この演習では、ラボ環境で使用されるコンピューティング リソース (リソースが制約される場合がある) を最小限に抑えるために、*単一ノード* クラスターを作成します。 運用環境では、通常、複数のワーカー ノードを含むクラスターを作成します。

> **ヒント**: Azure Databricks ワークスペースに 13.3 LTS **<u>ML</u>** 以降のランタイム バージョンを備えたクラスターが既にある場合は、この手順をスキップし、そのクラスターを使用してこの演習を完了できます。

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
    - **Databricks Runtime のバージョン**: "以下に該当する最新の非ベータ版ランタイム (標準ランタイム バージョン**ではない***) の **<u>ML</u>** エディションを選択します。"
        - "*GPU を使用**しない***"
        - *Scala > **2.11** を含める*
        - "**3.4** 以上の Spark を含む"**
    - **Photon Acceleration を使用する**: <u>オフ</u>にする
    - **ノードの種類**: Standard_DS3_v2
    - **非アクティブ状態が ** *20* ** 分間続いた後終了する**

1. クラスターが作成されるまで待ちます。 これには 1、2 分かかることがあります。

> **注**: クラスターの起動に失敗した場合、Azure Databricks ワークスペースがプロビジョニングされているリージョンでサブスクリプションのクォータが不足していることがあります。 詳細については、「[CPU コアの制限によってクラスターを作成できない](https://docs.microsoft.com/azure/databricks/kb/clusters/azure-core-limit)」を参照してください。 その場合は、ワークスペースを削除し、別のリージョンに新しいワークスペースを作成してみてください。 次のように、セットアップ スクリプトのパラメーターとしてリージョンを指定できます: `./mslearn-databricks/setup.ps1 eastus`

## ノートブックを作成する

Spark MLLib ライブラリを使って機械学習モデルをトレーニングするコードを実行するので、最初の手順ではワークスペースに新しいノートブックを作成します。

1. サイド バーで **[(+) 新規]** タスクを使用して、**Notebook** を作成します。
1. 既定のノートブック名 (**無題のノートブック *[日付]***) を "**ディープ ラーニング**" に変更し、**[接続]** ドロップダウン リストでクラスターを選択します (まだ選択されていない場合)。 クラスターが実行されていない場合は、起動に 1 分ほどかかる場合があります。

## データの取り込みと準備

この演習のシナリオは、南極でのペンギンの観察に基づいており、機械学習モデルをトレーニングして、観察されたペンギンの位置と体の測定値に基づいて種類を予測することを目標としています。

> **[引用]**: この演習で使用するペンギンのデータセットは、[Dr. Kristen Gorman](https://www.uaf.edu/cfos/people/faculty/detail/kristen-gorman.php) と、[Long Term Ecological Research Network](https://lternet.edu/) のメンバーである [Palmer Station, Antarctica LTER](https://pal.lternet.edu/) によって収集されて使用できるようにされているデータのサブセットです。

1. ノートブックの最初のセル内に次のコードを入力します。これは "シェル" コマンドを使用して、GitHub のペンギン データをクラスターで使用されるファイル システムの中にダウンロードします。**

    ```bash
    %sh
    rm -r /dbfs/deepml_lab
    mkdir /dbfs/deepml_lab
    wget -O /dbfs/deepml_lab/penguins.csv https://raw.githubusercontent.com/MicrosoftLearning/mslearn-databricks/main/data/penguins.csv
    ```

1. そのセルの左側にある **[&#9656; セルの実行]** メニュー オプションを使用して実行します。 その後、コードによって実行される Spark ジョブが完了するまで待ちます。
1. 次に、機械学習用のデータを準備します。 既存のコード セルの下で、 **[+]** アイコンを使用して新しいコード セルを追加します。 次に、新しいセルに次のコードを入力して実行します。
    - 不完全な行を削除する
    - (文字列の) 島名を整数としてエンコードする
    - 適切なデータ型を適用する
    - 数値データを同様のスケールに正規化する
    - データを 2 つのデータセットに分割する (1 つはトレーニング用、もう 1 つはテスト用)。

    ```python
   from pyspark.sql.types import *
   from pyspark.sql.functions import *
   from sklearn.model_selection import train_test_split
   
   # Load the data, removing any incomplete rows
   df = spark.read.format("csv").option("header", "true").load("/deepml_lab/penguins.csv").dropna()
   
   # Encode the Island with a simple integer index
   # Scale FlipperLength and BodyMass so they're on a similar scale to the bill measurements
   islands = df.select(collect_set("Island").alias('Islands')).first()['Islands']
   island_indexes = [(islands[i], i) for i in range(0, len(islands))]
   df_indexes = spark.createDataFrame(island_indexes).toDF('Island', 'IslandIdx')
   data = df.join(df_indexes, ['Island'], 'left').select(col("IslandIdx"),
                      col("CulmenLength").astype("float"),
                      col("CulmenDepth").astype("float"),
                      (col("FlipperLength").astype("float")/10).alias("FlipperScaled"),
                       (col("BodyMass").astype("float")/100).alias("MassScaled"),
                      col("Species").astype("int")
                       )
   
   # Oversample the dataframe to triple its size
   # (Deep learning techniques like LOTS of data)
   for i in range(1,3):
       data = data.union(data)
   
   # Split the data into training and testing datasets   
   features = ['IslandIdx','CulmenLength','CulmenDepth','FlipperScaled','MassScaled']
   label = 'Species'
      
   # Split data 70%-30% into training set and test set
   x_train, x_test, y_train, y_test = train_test_split(data.toPandas()[features].values,
                                                       data.toPandas()[label].values,
                                                       test_size=0.30,
                                                       random_state=0)
   
   print ('Training Set: %d rows, Test Set: %d rows \n' % (len(x_train), len(x_test)))
    ```

## PyTorch ライブラリをインストールしてインポートする

PyTorch は、ディープ ニューラル ネットワーク (DNN) などの機械学習モデルを作成するためのフレームワークです。 PyTorch を使用してペンギン分類器を作成する予定なので、使用する PyTorch ライブラリをインポートする必要があります。 PyTorch は、ML Databricks ランタイムを使用して Azure Databricks クラスターに既にインストールされています (PyTorch の具体的なインストールは、*cuda* での高パフォーマンス処理に使用できるグラフィックス処理ユニット (GPU) がクラスターにあるかどうかによって異なります)。

1. 新しいコード セルを追加し、次のコードを実行して PyTorch を使用する準備をします。

    ```python
   import torch
   import torch.nn as nn
   import torch.utils.data as td
   import torch.nn.functional as F
   
   # Set random seed for reproducability
   torch.manual_seed(0)
   
   print("Libraries imported - ready to use PyTorch", torch.__version__)
    ```

## データ ローダーを作成する

PyTorch では、トレーニング データと検証データが、"*データ ローダー*" を使用して一括で読み込まれます。 データは既に numpy 配列に読み込まれていますが、それを PyTorch データセットにラップし (データはそこでPyTorch *tensor* オブジェクトに変換されます)、そのデータセットからバッチを読み取るローダーを作成する必要があります。

1. セルを追加し、次のコードを実行してデータ ローダーを準備します。

    ```python
   # Create a dataset and loader for the training data and labels
   train_x = torch.Tensor(x_train).float()
   train_y = torch.Tensor(y_train).long()
   train_ds = td.TensorDataset(train_x,train_y)
   train_loader = td.DataLoader(train_ds, batch_size=20,
       shuffle=False, num_workers=1)

   # Create a dataset and loader for the test data and labels
   test_x = torch.Tensor(x_test).float()
   test_y = torch.Tensor(y_test).long()
   test_ds = td.TensorDataset(test_x,test_y)
   test_loader = td.DataLoader(test_ds, batch_size=20,
                                shuffle=False, num_workers=1)
   print('Ready to load data')
    ```

## ニューラル ネットワークを定義する

これで、ニューラル ネットワークを定義する準備ができました。 この場合は、完全に接続された次の 3 つのレイヤーで構成されるネットワークを作成します。

- 入力レイヤー。各特徴の入力値 (この場合は島インデックスと 4 つのペンギン測定値) を受け取り、10 個の出力を生成しました。
- 非表示レイヤー。入力レイヤーから 10 個の入力を受け取り、10 個の出力を次のレイヤーに送信します。
- 出力レイヤー。考えられる 3 種のペンギンそれぞれについて確率のベクトルを生成します。

データがネットワークに渡され通過することで、そのネットワークがトレーニングされると、**forward** 関数は、(結果が正の数値に制約されるように) *RELU* アクティブ化関数を最初の 2 つのレイヤーに適用します。また、*log_softmax* 関数が使用される最終的な出力レイヤーを返すことで、考えられる 3 つのクラスそれぞれの確率スコアを表す値を返します。

1. 次のコードを実行してニューラル ネットワークを定義します。

    ```python
   # Number of hidden layer nodes
   hl = 10
   
   # Define the neural network
   class PenguinNet(nn.Module):
       def __init__(self):
           super(PenguinNet, self).__init__()
           self.fc1 = nn.Linear(len(features), hl)
           self.fc2 = nn.Linear(hl, hl)
           self.fc3 = nn.Linear(hl, 3)
   
       def forward(self, x):
           fc1_output = torch.relu(self.fc1(x))
           fc2_output = torch.relu(self.fc2(fc1_output))
           y = F.log_softmax(self.fc3(fc2_output).float(), dim=1)
           return y
   
   # Create a model instance from the network
   model = PenguinNet()
   print(model)
    ```

## ニューラル ネットワーク モデルをトレーニングしてテストする関数を作成する

モデルをトレーニングするには、トレーニング値をネットワーク経由で繰り返し前方に送り込み、損失関数を使用して損失を計算し、オプティマイザーを使用して重みとバイアス値の調整を逆伝播し、残しておいたテスト データを使用してモデルを検証する必要があります。

1. これを行うには、次のコードを使用して、モデルをトレーニングして最適化する関数と、モデルをテストする関数を作成します。

    ```python
   def train(model, data_loader, optimizer):
       device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
       model.to(device)
       # Set the model to training mode
       model.train()
       train_loss = 0
       
       for batch, tensor in enumerate(data_loader):
           data, target = tensor
           #feedforward
           optimizer.zero_grad()
           out = model(data)
           loss = loss_criteria(out, target)
           train_loss += loss.item()
   
           # backpropagate adjustments to the weights
           loss.backward()
           optimizer.step()
   
       #Return average loss
       avg_loss = train_loss / (batch+1)
       print('Training set: Average loss: {:.6f}'.format(avg_loss))
       return avg_loss
              
               
   def test(model, data_loader):
       device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
       model.to(device)
       # Switch the model to evaluation mode (so we don't backpropagate)
       model.eval()
       test_loss = 0
       correct = 0
   
       with torch.no_grad():
           batch_count = 0
           for batch, tensor in enumerate(data_loader):
               batch_count += 1
               data, target = tensor
               # Get the predictions
               out = model(data)
   
               # calculate the loss
               test_loss += loss_criteria(out, target).item()
   
               # Calculate the accuracy
               _, predicted = torch.max(out.data, 1)
               correct += torch.sum(target==predicted).item()
               
       # Calculate the average loss and total accuracy for this epoch
       avg_loss = test_loss/batch_count
       print('Validation set: Average loss: {:.6f}, Accuracy: {}/{} ({:.0f}%)\n'.format(
           avg_loss, correct, len(data_loader.dataset),
           100. * correct / len(data_loader.dataset)))
       
       # return average loss for the epoch
       return avg_loss
    ```

## モデルをトレーニングする

これで、**トレーニング**関数と**テスト**関数を使用して、ニューラル ネットワーク モデルをトレーニングできるようになりました。 複数の "*エポック*" に対してニューラル ネットワークを繰り返しトレーニングし、各エポックの損失と精度の統計情報をログします。

1. 次のコードを使用してモデルをトレーニングします。

    ```python
   # Specify the loss criteria (we'll use CrossEntropyLoss for multi-class classification)
   loss_criteria = nn.CrossEntropyLoss()
   
   # Use an optimizer to adjust weights and reduce loss
   learning_rate = 0.001
   optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
   optimizer.zero_grad()
   
   # We'll track metrics for each epoch in these arrays
   epoch_nums = []
   training_loss = []
   validation_loss = []
   
   # Train over 100 epochs
   epochs = 100
   for epoch in range(1, epochs + 1):
   
       # print the epoch number
       print('Epoch: {}'.format(epoch))
       
       # Feed training data into the model
       train_loss = train(model, train_loader, optimizer)
       
       # Feed the test data into the model to check its performance
       test_loss = test(model, test_loader)
       
       # Log the metrics for this epoch
       epoch_nums.append(epoch)
       training_loss.append(train_loss)
       validation_loss.append(test_loss)
    ```

    トレーニング プロセスの実行中に何が起こっているかを理解してみましょう。

    - 各 "*エポック*" で、完全なトレーニング データ セットが前方に渡されネットワークを通過します。 観察ごとに 5 つの特徴があり、入力レイヤーには 5 つの対応するノードがあります。そのため、各観察の特徴は、5 つの値のベクトルとしてそのレイヤーに渡されます。 ただし、効率化のために、特徴ベクトルはバッチにグループ化されるので、実際には複数の特徴ベクトルのマトリックスが毎回フィードされます。
    - 特徴量の値のマトリックスは、初期化された重みとバイアス値を使用して、加重和を実行する関数によって処理されます。 その後、この関数の結果は、入力レイヤーのアクティブ化関数によって、次のレイヤーのノードに渡される値を制約するように処理されます。
    - 加重和とアクティブ化関数は、各レイヤーで繰り返されます。 関数は、個々のスカラー値ではなく、ベクトルとマトリックスで動作することに注意してください。 つまり、前方パスは、基本的に入れ子になった一連の線形代数関数です。 だから、データ サイエンティストはグラフィカル処理ユニット (GPU) を搭載したコンピューターを好んで使用します。GPU はマトリックスとベクトルの計算に合わせて最適化されているのです。
    - ネットワークの最終レイヤーでは、出力ベクトルには、考えらえるクラス (この場合は、クラス 0、1、および 2) それぞれについて計算された値が含まれます。 このベクトルは、実際のクラスに基づいて予測値からの距離を決定する "*損失関数*" によって処理されます。たとえば、ジェンツーペンギン (クラス 1) 観察の出力が \[0.3、0.4、0.3\] だとします。 正しい予測は \[0.0、1.0、0.0\] になるため、予測値と実際の値の差異 (各予測値が本来の値からどのくらい離れているか) は \[0.3、0.6、0.3\] です。 この差異はバッチごとに集計され、エポックのトレーニング データによって発生した全体的なエラー レベル ("*損失*") を計算するために、実行中の集計として維持されます。
    - 検証データは各エポックの最後にネットワークに渡され通過し、その損失と精度 (出力ベクトルの中の最も高い確率値に基づく正しい予測の割合) も計算されます。 これを行うと、トレーニングされていないデータを使用して、各エポックの後にモデルのパフォーマンスを比較し、新しいデータに対して適切に一般化されるか、トレーニング データに "*オーバーフィット*" されるかを判断することができるため便利です。
    - すべてのデータが前方に渡されネットワークを通過すると、"*トレーニング*" データの損失関数の出力 (ただし、"*検証*" データ<u>ではない</u>) がオプティマイザーに渡されます。 オプティマイザーが損失を処理する方法の正確な詳細は、使用されている特定の最適化アルゴリズムによって異なりますが、基本的には、入力レイヤーから損失関数までのネットワーク全体を、1 つの大きな入れ子になった ("*複合*") 関数と考えることができます。 オプティマイザーは微分学をいくつか適用して、ネットワークで使用された重みとバイアス値ごとに関数の "*偏導関数*" を計算します。 入れ子になった関数に対してこれを効率的に行うことができるのは、"*チェーン ルール*" と呼ばれるものがあるからです。これを使用すると、合成関数の導関数を、その内部関数と外部関数の導関数から求めることができます。 ここでは数学的な細かなことを気にする必要はありませんが (これはオプティマイザーで自動的に行われます)、最終的な結果は偏導関数であり、これにより重みとバイアス値それぞれに関する損失関数の傾き ("*勾配*") がわかります。つまり、損失を最小限に抑えるために、重みとバイアス値を増やすか減らすかを決定できます。
    - オプティマイザーは、重みとバイアスをどちらの方向に調整するか、そして "*学習率*" を使用してどの程度調整するかを決定します。その後、"*逆伝搬法*" と呼ばれるプロセスで後方に向かって動作してネットワークを通過し、各レイヤーの重みとバイアスに新しい値を割り当てます。
    - これで次のエポックは、トレーニング、検証、逆伝播プロセス全体を、前のエポックから修正された重みとバイアスで開始して繰り返すようになり、うまくいけば損失レベルが低くなります。
    - このようなプロセスが、100 エポックに対して継続して行われます。

## トレーニングと検証の損失を確認する

トレーニングが完了したら、モデルのトレーニングと検証中に記録した損失メトリックを調べることができます。 次の 2 つが本当に実現しているかを確認します。

- 損失がエポックごとに減っている。モデルが適切な重みとバイアスを学習し、正しいラベルを予測していることを示している。
- トレーニングの損失と検証の損失が同じようなトレンドに従っている。モデルがトレーニング データにオーバーフィットしていないことを示している。

1. 次のコードを使用して損失をプロットします。

    ```python
   %matplotlib inline
   from matplotlib import pyplot as plt
   
   plt.plot(epoch_nums, training_loss)
   plt.plot(epoch_nums, validation_loss)
   plt.xlabel('epoch')
   plt.ylabel('loss')
   plt.legend(['training', 'validation'], loc='upper right')
   plt.show()
    ```

## トレーニング済みの重みとバイアスを表示する

トレーニング済みのモデルは、トレーニング中にオプティマイザーによって決められた最終的な重みとバイアスで構成されます。 ネットワーク モデルに基づいて、各レイヤーに対して次の値が想定されます。

- レイヤー 1 (*fc1*): 10 個の出力ノードに対して 5 個の入力値なので、10 x 5 の重みと 10 個のバイアス値があるはずです。
- レイヤー 2 (*fc2*): 10 個の出力ノードに対して 10 個の入力値なので、10 x 10 の重みと 10 個のバイアス値があるはずです。
- レイヤー 3 (*fc3*): 3 個の出力ノードに対して 10 個の入力値なので、3 x 10 の重みと 3 個のバイアス値があるはずです。

1. 次のコードを使用して、トレーニング済みのモデルのレイヤーを表示します。

    ```python
   for param_tensor in model.state_dict():
       print(param_tensor, "\n", model.state_dict()[param_tensor].numpy())
    ```

## トレーニング済みのモデルを保存して使用する

トレーニング済みのモデルがあります。そのトレーニング済みの重みは、後で使用できるように保存できます。

1. 次のコードを使用して、モデルを保存します。

    ```python
   # Save the model weights
   model_file = '/dbfs/penguin_classifier.pt'
   torch.save(model.state_dict(), model_file)
   del model
   print('model saved as', model_file)
    ```

1. 次のコードを使用して、モデルの重みを読み込み、新しいペンギン観察に対して種を予測します。

    ```python
   # New penguin features
   x_new = [[1, 50.4,15.3,20,50]]
   print ('New sample: {}'.format(x_new))
   
   # Create a new model class and load weights
   model = PenguinNet()
   model.load_state_dict(torch.load(model_file))
   
   # Set model to evaluation mode
   model.eval()
   
   # Get a prediction for the new data sample
   x = torch.Tensor(x_new).float()
   _, predicted = torch.max(model(x).data, 1)
   
   print('Prediction:',predicted.item())
    ```

## Horovod を使用してトレーニングを配布する

前のモデル トレーニングは、クラスターの単一ノードで実行されました。 実際のところ、ディープ ラーニング モデルのトレーニングは一般的に、1 台のコンピューターで複数の CPU (または GPU) にわたってスケーリングすることをお勧めします。ただし、大量のトレーニング データを、ディープ ラーニング モデルの複数のレイヤーに渡して通過させる必要がある場合は、トレーニング作業を複数のクラスター ノードに分散させることで、ある程度効率化を図れることがあります。

Horovod は、ディープ ラーニング トレーニングを、Spark クラスター内の複数のノード (Azure Databricks ワークスペースにプロビジョニングされているノードなど) に配布するときに使用できるオープン ソース ライブラリです。

### トレーニング関数を作成する

Horovod を使用するには、トレーニング設定を構成するコードをカプセル化し、新しい関数で**トレーニング**関数を呼び出します。これを **HorovodRunner** クラスを使用して実行し、複数のノードに実行を分散します。 トレーニング ラッパー関数では、さまざまな Horovod クラスを使用して分散データ ローダーを定義することで、各ノードがデータセット全体のサブセットで動作し、モデルの重みとオプティマイザーの初期状態をすべてのノードにブロードキャストし、使用されているノードの数を特定し、コードが実行されているノードを判断できるようにします。

1. 次のコードを実行して、Horovod を使用してモデルをトレーニングする関数を作成します。

    ```python
   import horovod.torch as hvd
   from sparkdl import HorovodRunner
   
   def train_hvd(model):
       from torch.utils.data.distributed import DistributedSampler
       
       hvd.init()
       
       device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
       if device.type == 'cuda':
           # Pin GPU to local rank
           torch.cuda.set_device(hvd.local_rank())
       
       # Configure the sampler so that each worker gets a distinct sample of the input dataset
       train_sampler = DistributedSampler(train_ds, num_replicas=hvd.size(), rank=hvd.rank())
       # Use train_sampler to load a different sample of data on each worker
       train_loader = torch.utils.data.DataLoader(train_ds, batch_size=20, sampler=train_sampler)
       
       # The effective batch size in synchronous distributed training is scaled by the number of workers
       # Increase learning_rate to compensate for the increased batch size
       learning_rate = 0.001 * hvd.size()
       optimizer = torch.optim.Adam(model.parameters(), lr=learning_rate)
       
       # Wrap the local optimizer with hvd.DistributedOptimizer so that Horovod handles the distributed optimization
       optimizer = hvd.DistributedOptimizer(optimizer, named_parameters=model.named_parameters())
   
       # Broadcast initial parameters so all workers start with the same parameters
       hvd.broadcast_parameters(model.state_dict(), root_rank=0)
       hvd.broadcast_optimizer_state(optimizer, root_rank=0)
   
       optimizer.zero_grad()
   
       # Train over 50 epochs
       epochs = 100
       for epoch in range(1, epochs + 1):
           print('Epoch: {}'.format(epoch))
           # Feed training data into the model to optimize the weights
           train_loss = train(model, train_loader, optimizer)
   
       # Save the model weights
       if hvd.rank() == 0:
           model_file = '/dbfs/penguin_classifier_hvd.pt'
           torch.save(model.state_dict(), model_file)
           print('model saved as', model_file)
    ```

1. 次のコードを使用して、**HorovodRunner** オブジェクトから自分の関数を呼び出します。

    ```python
   # Reset random seed for PyTorch
   torch.manual_seed(0)
   
   # Create a new model
   new_model = PenguinNet()
   
   # We'll use CrossEntropyLoss to optimize a multiclass classifier
   loss_criteria = nn.CrossEntropyLoss()
   
   # Run the distributed training function on 2 nodes
   hr = HorovodRunner(np=2, driver_log_verbosity='all') 
   hr.run(train_hvd, model=new_model)
   
   # Load the trained weights and test the model
   test_model = PenguinNet()
   test_model.load_state_dict(torch.load('/dbfs/penguin_classifier_hvd.pt'))
   test_loss = test(test_model, test_loader)
    ```

すべての出力を表示するには、スクロールしなければならない場合があります。これにより Horovod からの情報メッセージに続いて、ノードからのログ出力が表示されるはずです (**driver_log_verbosity** パラメーターが **all** に設定されているため)。 ノード出力には、各エポックの後の損失が表示されます。 最後に、**テスト**関数を使用して、トレーニング済みのモデルをテストします。

> **ヒント**: 各エポックの後の損失が減らない場合は、セルをもう一度実行してみてください。

## クリーンアップ

Azure Databricks ポータルの **[コンピューティング]** ページでクラスターを選択し、**[&#9632; 終了]** を選択してクラスターをシャットダウンします。

Azure Databricks を調べ終わったら、作成したリソースを削除できます。これにより、不要な Azure コストが生じないようになり、サブスクリプションの容量も解放されます。
