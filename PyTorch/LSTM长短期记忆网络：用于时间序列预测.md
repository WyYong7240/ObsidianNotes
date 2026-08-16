---
tags:
  - LSTM
  - 长短期记忆网络
  - 时间序列预测
  - 单步预测
  - 多步预测
---
# LSTM单步预测模型

> 环境与依赖
>
> python==3.9.24
>
> Flask==3.1.2
>
> flask_cors==6.0.1
>
> joblib==1.5.2
>
> matplotlib==3.9.4
>
> numpy==2.0.2
>
> pandas==2.3.3
>
> scikit_learn==1.6.1
>
> torch==2.7.1+cu118
>
> torchaudio==2.7.1+cu118
>
> torchvision==0.22.1+cu118

## 构造数据集：从CSV文件到数据集迭代器

1. 处理CSV文件

   本例中处理的CSV文件

   ~~~csv
   time,latency
   2025-04-19 15:46:46,0.500
   2025-04-19 15:46:56,0.344
   2025-04-19 15:47:06,0.362
   2025-04-19 15:47:16,0.383
   2025-04-19 15:47:26,0.377
   ~~~

   处理代码

   ~~~python
   def data_prepare(path):
       # 读取数据
       # 从CSV读取数据，并将time列解析为datetime类型
       df = pd.read_csv(path, parse_dates=['time'])
       # 按时间排序，否则打乱依赖关系
       df = df.sort_values('time')
   
       print("Latency min:", df['latency'].min())
       print("Latency max:", df['latency'].max())
       print("Latency mean:", df['latency'].mean())
       print("Latency std:", df['latency'].std())
   
       # 只取latency列,并reshape为2维，因为MinMaxScaler要求输入为2D:(N,1)
       values = df['latency'].values.reshape(-1, 1)
   
       # 使用MinMaxScaler将latency数据归一化到0-1区间;这里使用了fit_transform,如果有多个文件，建议只对第一个fit，后面用transform,形状不变
       # scaled = minMaxScaler.fit_transform(values)
       # scaled = robustScaler.fit_transform(values)
       scaled = standardScaler.fit_transform(values)
   ~~~

   这里，`values = df['latency'].values.reshape(-1, 1)`是只将latency列的数据取出来用，**由于Scaler要求输入是二维的，而latency列的数据取出来是N行1列，形状属于(N),因此需要reshape为二维，变为(N,1)；**

   reshape的参数含义：**-1是占位符，表示第一维度的大小自动计算，1表示第二维度的大小必须是1；因此reshape完毕之后形状变为(N,1),数组形式是二维数组，但是每一列只有一个元素，一个元素作为一行**

   这里scaler采用了standardScaler，`from sklearn.preprocessing import StandardScaler`,作用是将数值归一化，防止数值过大影响神经网络训练，**经过此步骤，数据形状不会变化**

2. 利用CSV文件生成时序样本

   示例代码

   ~~~python
   def create_sequences(data, seq_len=30):
       xs, ys = [], []
       # 30个一组
       for i in range(len(data) - seq_len):
           # x形状为(seq_len, 1), y形状为(1)
           x = data[i:i+seq_len]   # 取i到i+seq_len-1的数据作为输入
           y = data[i+seq_len]     # 取第i+seq_len个数组作为标签
           xs.append(x)
           ys.append(y)
       # xs形状(seq_num, seq_len, 1), y形状为(seq_num, 1)
       return np.array(xs), np.array(ys)
   ~~~

   这里由于是单步预测，因此使用过去30个时间步的数据，来预测未来第一个时间步的延迟，因此seq_len是30；

   这里x的形状是(seq_len,1)，y的形状是(1),数组表示如下

   ~~~python
   # x
   [
       [1],
       [2],
       ...
       [30]
   ]
   # y
   [31]
   ~~~

   因此，xs形状(seq_num, seq_len, 1), y形状为(seq_num, 1)

   ~~~python
   # xs
   [
       [ [1],[2],...,[30] ],
       [ [2],[3],...,[31] ],
       ....
   ]
   # ys
   [
       [31],
       [32],
       ....
   ]
   ~~~

3. 利用时序样本构造数据集迭代器

   示例代码

   ~~~python
   # 自定义pytorch数据集
   class TimeSeriesDataSet(Dataset):
       def __init__(self, X, y):
           self.X = torch.tensor(X, dtype=torch.float32)
           self.y = torch.tensor(y, dtype=torch.float32)
   
       def __len__(self):
           return len(self.X)
   
       def __getitem__(self, idx):
           return self.X[idx], self.y[idx]
   
   def train_test_partition(X, y):
       # 划分训练集/测试集
       # 按80%训练，20%测试划分数据，时间序列不能随机打乱，所以是前80%训练
       train_size = int(0.8 * len(X))
       train_dataset = TimeSeriesDataSet(X[:train_size], y[:train_size])
       test_dataset = TimeSeriesDataSet(X[train_size:], y[train_size:])
   
       # 训练集每个epoch打乱样本顺序，有助于泛化
       # x_batch形状为(batch_size, seq_len, 1), y_batch形状为(batch_size, 1)
       train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)
       # 测试集保持时间顺序以便于后续绘图对比
       test_loader = DataLoader(test_dataset, batch_size=32, shuffle=False)
       return train_loader, test_loader
   ~~~

   首先定义一个TimeSeriesDateSet类，继承自DataSet，该类中，定义两个张量成员变量x、y

   将前面生成的时序数据样本按照80%训练集20%测试集比例划分，然后再将划分好的数据集，构建为数据集迭代器

   其中训练集数据迭代器开启打乱顺序，有助于泛化，测试集不打乱；设置batch_size为32，自此迭代器每次迭代得到的数据形状为

   **x_batch形状为(batch_size, seq_len, 1), y_batch形状为(batch_size, 1)**

## 定义LSTM单步预测模型

* LSTM概述

  在pytorch中，LSTM的期望输入是一个3D张量，其形状为`(batch_size, seq_len, input_size)`，也就是序列样本、时间步长、特征数量

  当我们需要一次预测多个指标或者其他数据时，input_size就不是1了
  
  | 维度             | 含义                                         | 在你的代码中                                                 |
  | ---------------- | -------------------------------------------- | ------------------------------------------------------------ |
  | **`batch_size`** | 一个批次中包含多少个**独立的序列样本**。     | 由 `DataLoader` 的 `batch_size=32` 决定，所以通常是 `32`。   |
  | **`seq_len`**    | 每个序列有多少个**时间步长（time steps）**。 | 由 `SQL_LEN` 定义，比如 `30`。表示用过去 30 个时间点的数据来预测。 |
  | **`input_size`** | 每个时间步上有多少个**特征（features）**。   | 由 `LSTMModel` 的 `input_size` 参数定义，你设为 `1`，表示每个时间步只有 `latency` 这一个特征。 |

  `nn.LSTM` 的 `forward` 方法返回两个东西：`out, (h_n, c_n) = self.lstm(x, (h0, c0))`
  
  我们只关心 `out`，它是 LSTM 在每个时间步的**隐藏状态输出**。`out` 的形状是：`(batch_size, seq_len, hidden_size)`

~~~python
# 定义LSTM模型
class LSTMModel(nn.Module):
    # 继承nn.Module，默认输入特征维度latency只有1，隐藏层大小64，LSTM层数2
    def __init__(self, input_size=1, hidden_size=128, num_layers=3, output_size=1):
        super(LSTMModel, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers

        # 创建一个LSTM层，输入形状为(batch，seq_len，features);dropout=0.2在除了最后一层外的所有层之间添加dropout，防止过拟合
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True, dropout=0.3)
        # 全连接层，即输出层，将LSTM最后一个时间步的输出映射到预测值，通常是1维
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x):
        # 手动初始化LSTM的初始隐藏状态h0和细胞状态c0
        # 形状(num_layers, batch_size, hidden_size)
        # 使用.to(x.device)确保它们与输入张量在同一个设备
        h0 = torch.zeros(self.num_layers, x.size(0), self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers, x.size(0), self.hidden_size).to(x.device)

        # 前向传播通过LSTM，得到所有时间步的输出out，形状为(batch_size, seq_len, hidden_size)
        out, _ =self.lstm(x, (h0, c0))
        # 只取最后一个时间步的输出，因为这是单步预测，然后送入全连接层输出最终预测
        # out[:,-1,:]形状为(batch_size, hidden_size)
        # 最终out形状为(batch_size, output_size)
        out = self.fc(out[:, -1, :])
        return out
~~~

* 模型初始化函数

  设定输入的特征维度只有1，隐藏层大小128，LSTM层数3，输出维度1

  > * `hidden_size` 和 `num_layers` 对模型精度的影响
  >
  >   1. **hidden_size（隐藏层维度）**
  >
  >      **定义**：每个LSTM单元中隐藏状态的向量长度。
  >
  >      **作用**：决定了模型的“记忆容量”或“表达能力”。
  >
  >      与精度的关系
  >
  >      过小（如 16、32）：
  >
  >      - 模型容量不足，无法捕捉复杂的时间依赖模式，容易**欠拟合**。
  >      - 特别是在输入序列较长或动态变化复杂时，表现较差。
  >
  >      适中（如 64~256）：
  >
  >      - 多数时间序列任务中表现良好，平衡了表达能力和计算开销。
  >      - 你的设置 `hidden_size=128` 是一个**合理起点**。
  >
  >      过大（如 512 以上）：
  >
  >      - 增强了表达能力，但也显著增加参数量，可能导致：
  >        - **过拟合**（尤其数据量少时）
  >        - 训练变慢、梯度不稳定
  >        - 更难收敛
  >
  >   2. **num_layers（LSTM层数）**
  >
  >      **定义**：堆叠的LSTM层数，形成“深度LSTM”。
  >
  >      **作用**：更深的网络可以学习更抽象、非线性的特征组合。
  >
  >      与精度的关系
  >
  >      单层（1）
  >
  >      - 简单有效，适合大多数时间序列任务。
  >      - 不容易过拟合，训练稳定。
  >
  >      2~3层
  >
  >      - 能捕捉更复杂的时序结构，可能提升精度。
  >      - 你的设置 `num_layers=3` 是**较深但可行的选择**。
  >
  >      4层及以上
  >
  >      - 很容易导致：
  >        - 梯度消失/爆炸（尽管LSTM设计上缓解了这个问题）
  >        - 过拟合
  >        - 训练困难，需要残差连接或更好的初始化

  **为什么输出维度是1呢，因为我们是取过去30个时间步的数据来预测未来1个时间步的数据，**

  `out[:, -1, :]`的含义是，取out第一维的全部，第二维的最后一个数据也就是最后一个时间步，第三位的全部

  为什么取第二维的最后一个呢，也就是最后一个时间步呢，因为最后一个时间步的隐藏层是见过了总结了过去30个时间步的数据的，前面的时间步没有见过所有的前30个时间步的数据；相当于总结了30个时间步的经验之后再去推到未来第一个时间步的数据

  out[:, -1, :]的形状变为(batch_size, hidden_size);

  然后再经过全连接层，变为(batch_size, output_size)

## 模型训练与验证

1. 模型训练

   ~~~python
       # 训练模型
       # 自动选择设备，有cuda用cuda，没有用cpu
       device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
       # 实例化模型并移动到指定设备
       model = LSTMModel().to(device)
   
       # 定义损失函数均方误差和优化器Adam
       criterion = nn.MSELoss()
       optimizer = torch.optim.Adam(model.parameters(), lr=0.0001)
   
       for epoch in range(EPOCHS):
           # 启动Dropout/BatchNorm等训练行为
           model.train()
           total_loss = 0
           for X_batch, y_batch in train_loader:
               # 将每一批数据移动到设备上
               X_batch, y_batch = X_batch.to(device), y_batch.to(device)
   
               optimizer.zero_grad()   # 清除梯度
               y_pred = model(X_batch) # 前向传播
               loss = criterion(y_pred, y_batch)   # 计算损失
               loss.backward()     # 反向传播
               optimizer.step()    # 更新参数
   
               total_loss += loss.item()
           print(f"Epoch {epoch + 1}/{EPOCHS}, Train Loss: {total_loss / len(train_loader):.6f}")
   
       torch.save(model.state_dict(), "model/lstm_single_step.pth")
       torch.save(standardScaler, "model/scaler_single_step.pkl")
   ~~~

   > 1. 关于学习率
   >
   >    学习率是优化器（如 Adam）的关键超参数。影响机制：
   >
   >    **太小**
   >
   >    （如 1e-5）：
   >
   >    - 收敛极慢，训练耗时长。
   >    - 可能陷入局部最优或鞍点。
   >
   >    **适中**
   >
   >    （如 1e-3 ~ 5e-4）：
   >
   >    - 你的模型默认可设为 `1e-3`（Adam 的常见初始值）。
   >    - 收敛快且稳定，推荐作为起点。
   >
   >    **太大**
   >
   >    （如 1e-2 或更大）：
   >
   >    - 损失震荡剧烈，甚至发散（loss 不下降或 NaN）。
   >    - 跳过最优解，无法收敛

   最后两行是用于保存该参数训练下的LSTM模型参数和Scaler，用于后面重新加载模型用于预测

2. 模型验证

   ~~~python
       #测试与预测
       # 关闭Dropout，启动评估模式
       model.eval()
       preds, trues = [], []
   
       # 禁用梯度计算，节省内存
       with torch.no_grad():
           for X_batch, y_batch in test_loader:
               X_batch = X_batch.to(device)
               y_pred = model(X_batch)
               preds.append(y_pred.cpu().numpy())  # 将GPU张量转回Numpy
               trues.append(y_batch.numpy())
   
           # 将多个batch的预测结果拼接成完整数组
           preds = np.concatenate(preds)
           trues = np.concatenate(trues)
   
       # 反归一化;使用之前拟合的scaler将归一化的预测值还原为原始尺度，便于解释和绘图
       # preds_inv = minMaxScaler.inverse_transform(preds)
       # trues_inv = minMaxScaler.inverse_transform(trues)
       # preds_inv = robustScaler.inverse_transform(preds)
       # trues_inv = robustScaler.inverse_transform(trues)
       preds_inv = standardScaler.inverse_transform(preds)
       trues_inv = standardScaler.inverse_transform(trues)
       return preds_inv, trues_inv
   ~~~

   为什么要将y_pred.cpu呢，因为X_batch被放到了GPU上，因此生成的y_pred也在GPU上，为了返回数据，采用y_pred.cpu将数据放回CPU上

   此时y_pred的形状是(batch_size, 1)

   经过concatenate之后，preds的形状变为(N, 1),

   inverse_transform的作用是将数据缩放为原本大小，例如ms，便于后面画图和识别

3. 画图

   ~~~python
   def consequence_show(preds_inv, trues_inv):
       # 结果可视化
       plt.figure(figsize=(10, 6))
       plt.plot(trues_inv, label="True")
       plt.plot(preds_inv, label="Predicated")
       plt.title("LSTM Latency Predication")
       plt.xlabel("Time Step")
       plt.ylabel("Latency")
       plt.legend()
       plt.show()
   ~~~

   plt.plot是画数据，默认输入是2维的



## 整体代码

~~~python
import pandas as pd
import numpy as np
from sklearn.preprocessing import MinMaxScaler
from sklearn.preprocessing import RobustScaler
from sklearn.preprocessing import StandardScaler
import torch
from torch import nn
from torch.utils.data import Dataset, DataLoader
import matplotlib.pyplot as plt

# 归一化
minMaxScaler = MinMaxScaler()     # 对数据进行归一化，缩放到0-1区间，防止数值过大影响神经网络训练稳定性
robustScaler = RobustScaler()
standardScaler = StandardScaler()
SQL_LEN = 30    # 定义时间窗口长度，即每次输入LSTM的连续时间步为30，例如用前30秒的延迟值预测第31秒的延迟
EPOCHS = 50     # 训练轮数

# 构造时序样本
def create_sequences(data, seq_len=30):
    xs, ys = [], []
    # 30个一组
    for i in range(len(data) - seq_len):
        # x形状为(seq_len, 1), y形状为(1)
        x = data[i:i+seq_len]   # 取i到i+seq_len-1的数据作为输入
        y = data[i+seq_len]     # 取第i+seq_len个数组作为标签
        xs.append(x)
        ys.append(y)
    # xs形状(seq_num, seq_len, 1), y形状为(seq_num, 1)
    return np.array(xs), np.array(ys)

def data_prepare(path):
    # 读取数据
    # 从CSV读取数据，并将time列解析为datetime类型
    df = pd.read_csv(path, parse_dates=['time'])
    # 按时间排序，否则打乱依赖关系
    df = df.sort_values('time')


    print("Latency min:", df['latency'].min())
    print("Latency max:", df['latency'].max())
    print("Latency mean:", df['latency'].mean())
    print("Latency std:", df['latency'].std())

    # 只取latency列,并reshape为2维，因为MinMaxScaler要求输入为2D:(N,1)
    values = df['latency'].values.reshape(-1, 1)

    # 使用MinMaxScaler将latency数据归一化到0-1区间;这里使用了fit_transform,如果有多个文件，建议只对第一个fit，后面用transform,形状不变
    # scaled = minMaxScaler.fit_transform(values)
    # scaled = robustScaler.fit_transform(values)
    scaled = standardScaler.fit_transform(values)

    # 生成时序样本
    X, y = create_sequences(scaled, SQL_LEN)
    return X, y

# 自定义pytorch数据集
class TimeSeriesDataSet(Dataset):
    def __init__(self, X, y):
        self.X = torch.tensor(X, dtype=torch.float32)
        self.y = torch.tensor(y, dtype=torch.float32)

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        return self.X[idx], self.y[idx]

def train_test_partition(X, y):
    # 划分训练集/测试集
    # 按80%训练，20%测试划分数据，时间序列不能随机打乱，所以是前80%训练
    train_size = int(0.8 * len(X))
    train_dataset = TimeSeriesDataSet(X[:train_size], y[:train_size])
    test_dataset = TimeSeriesDataSet(X[train_size:], y[train_size:])

    # 训练集每个epoch打乱样本顺序，有助于泛化
    # x_batch形状为(batch_size, seq_len, 1), y_batch形状为(batch_size, 1)
    train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)
    # 测试集保持时间顺序以便于后续绘图对比
    test_loader = DataLoader(test_dataset, batch_size=32, shuffle=False)
    return train_loader, test_loader

# 定义LSTM模型
class LSTMModel(nn.Module):
    # 继承nn.Module，默认输入特征维度latency只有1，隐藏层大小64，LSTM层数2
    def __init__(self, input_size=1, hidden_size=128, num_layers=3, output_size=1):
        super(LSTMModel, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers

        # 创建一个LSTM层，输入形状为(batch，seq_len，features);dropout=0.2在除了最后一层外的所有层之间添加dropout，防止过拟合
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True, dropout=0.3)
        # 全连接层，即输出层，将LSTM最后一个时间步的输出映射到预测值，通常是1维
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x):
        # 手动初始化LSTM的初始隐藏状态h0和细胞状态c0
        # 形状(num_layers, batch_size, hidden_size)
        # 使用.to(x.device)确保它们与输入张量在同一个设备
        h0 = torch.zeros(self.num_layers, x.size(0), self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers, x.size(0), self.hidden_size).to(x.device)

        # 前向传播通过LSTM，得到所有时间步的输出out，形状为(batch_size, seq_len, hidden_size)
        out, _ =self.lstm(x, (h0, c0))
        # 只取最后一个时间步的输出，因为这是单步预测，然后送入全连接层输出最终预测
        # out[:,-1,:]形状为(batch_size, hidden_size)
        # 最终out形状为(batch_size, output_size)
        out = self.fc(out[:, -1, :])
        return out

def train_and_test_model(train_loader, test_loader):
    # 训练模型
    # 自动选择设备，有cuda用cuda，没有用cpu
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    # 实例化模型并移动到指定设备
    model = LSTMModel().to(device)

    # 定义损失函数均方误差和优化器Adam
    criterion = nn.MSELoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=0.0001)

    for epoch in range(EPOCHS):
        # 启动Dropout/BatchNorm等训练行为
        model.train()
        total_loss = 0
        for X_batch, y_batch in train_loader:
            # 将每一批数据移动到设备上
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)

            optimizer.zero_grad()   # 清除梯度
            y_pred = model(X_batch) # 前向传播
            loss = criterion(y_pred, y_batch)   # 计算损失
            loss.backward()     # 反向传播
            optimizer.step()    # 更新参数

            total_loss += loss.item()
        print(f"Epoch {epoch + 1}/{EPOCHS}, Train Loss: {total_loss / len(train_loader):.6f}")

    torch.save(model.state_dict(), "model/lstm_single_step.pth")
    torch.save(standardScaler, "model/scaler_single_step.pkl")

    #测试与预测
    # 关闭Dropout，启动评估模式
    model.eval()
    preds, trues = [], []

    # 禁用梯度计算，节省内存
    with torch.no_grad():
        for X_batch, y_batch in test_loader:
            X_batch = X_batch.to(device)
            y_pred = model(X_batch)
            preds.append(y_pred.cpu().numpy())  # 将GPU张量转回Numpy
            trues.append(y_batch.numpy())

        # 将多个batch的预测结果拼接成完整数组
        preds = np.concatenate(preds)
        trues = np.concatenate(trues)

    # 反归一化;使用之前拟合的scaler将归一化的预测值还原为原始尺度，便于解释和绘图
    # preds_inv = minMaxScaler.inverse_transform(preds)
    # trues_inv = minMaxScaler.inverse_transform(trues)
    # preds_inv = robustScaler.inverse_transform(preds)
    # trues_inv = robustScaler.inverse_transform(trues)
    preds_inv = standardScaler.inverse_transform(preds)
    trues_inv = standardScaler.inverse_transform(trues)
    return preds_inv, trues_inv

def consequence_show(preds_inv, trues_inv):
    # 结果可视化
    plt.figure(figsize=(10, 6))
    plt.plot(trues_inv, label="True")
    plt.plot(preds_inv, label="Predicated")
    plt.title("LSTM Latency Predication")
    plt.xlabel("Time Step")
    plt.ylabel("Latency")
    plt.legend()
    plt.show()

def main():
    X, y = data_prepare("latency_csv/master2node1.csv")
    train_loader, test_loader = train_test_partition(X, y)
    preds_inv, trues_inv = train_and_test_model(train_loader, test_loader)
    consequence_show(preds_inv, trues_inv)


if __name__ == '__main__':
    # preprocess_latency_txt("latency/master2node1.txt", "latency_csv/master2node1.csv")
    main()
    # print("PyTorch version:", torch.__version__)
    # print("CUDA available:", torch.cuda.is_available())
    # print("CUDA version:", torch.version.cuda)
    # print("Current device:", torch.cuda.current_device())
    # print("Device name:", torch.cuda.get_device_name(0) if torch.cuda.is_available() else "No GPU")
~~~

# LSTM多步预测模型

> 这个就是将用过去30个时间步的数据预测未来1个时间步的数据改为预测未来30个时间步的数据
>
> 也是取用模型最后一个时间步的数据来一次性预测未来30个时间步的数据

## 构造数据集：从CSV文件到数据集迭代器

* 处理CSV文件

  ~~~python
  def data_prepare_and_partition(path):
      # 读取数据
      # 从CSV读取数据，并将time列解析为datetime类型
      df = pd.read_csv(path, parse_dates=['time'])
      # 按时间排序，否则打乱依赖关系
      df = df.sort_values('time')
  
      # 判断是要训练集数据还是测试集数据,如果是训练集，取前80%，测试集取后20%
      train_size = int(0.8 * len(df))
      train_df = df[:train_size]
      test_df = df[train_size:]
  
      print("Train:")
      print("Latency min:", train_df['latency'].min())
      print("Latency max:", train_df['latency'].max())
      print("Latency mean:",train_df['latency'].mean())
      print("Latency std:", train_df['latency'].std())
      print("Test:")
      print("Latency min:", test_df['latency'].min())
      print("Latency max:", test_df['latency'].max())
      print("Latency mean:",test_df['latency'].mean())
      print("Latency std:", test_df['latency'].std())
  
      # 只取latency列,并reshape为2维，因为MinMaxScaler要求输入为2D
      # 这里，将latency重塑为一个二维数组，如果原始有N行，那么values的形状是(N,1)，即N行1列
      # 参数含义：-1是占位符，表示自动计算这个维度的大小； 1表示第二维度列数必须是1
      train_values = train_df['latency'].values.reshape(-1, 1)
      test_values = test_df['latency'].values.reshape(-1, 1)
  
      # 经过这一步，形状不变
      # 使用MinMaxScaler将latency数据归一化到0-1区间;这里使用了fit_transform,如果有多个文件，建议只对第一个fit，后面用transform
      train_scaled = scaler.fit_transform(train_values)
      test_scaled = scaler.fit_transform(test_values)
  
      return train_scaled, test_scaled
  ~~~

  这里为什么在处理CSV文件的地方提前将数据集划分为训练集和测试集呢，因为按照原本单步预测的方式，要到构建数据集迭代器才开始划分
  这就使得，**训练集和测试集都是一样使用过去30个时间步的数据来直接预测未来30个时间步的数据，按照原本的时序样本构造方式，测试集的样本形式是**

  ~~~python
  # xs
  [
      [ [1],[2],...,[30] ],
      [ [2],[3],...,[31] ],
      ....
  ]
  # ys
  [
      [ [31],[32],...,[60] ],
      [ [32],[33],...,[61] ],
      ....
  ]
  ~~~

  **可以发现，对应的标签y是有重复预测的部分，因为现在是多步预测，与单步预测不一致，所以y标签就不能采用步长为1的滑动方式了，需要采用步长为预测时间步数的滑动**

  具体其实是为了后面针对测试集的预测结果来画图方便，可以将预测结果拼接成一整个具有时序性的时间序列数据，直接绘图

  为了方便，我们预测时间步数采用的也是30

* 构建时序样本

  ~~~python
  # 构造训练集时序样本
  def create_train_multi_step_sequences(data, seq_len=30, pred_len=30):
      xs, ys = [], []
      # 30个一组
      for i in range(len(data) - seq_len - pred_len):
          # 这里，每个x都是一个形状为(seq_len,1)的二维数组;y也同理
          x = data[i:i+seq_len]   # 取i到i+seq_len-1的数据作为输入
          y = data[i+seq_len:i+seq_len+pred_len]     # 取第i+seq_len个数组作为标签
          xs.append(x)
          ys.append(y)
          # 因此，xs的形状是( len(data)-seq_len-pred_len, seq_len, 1)
          # ys的形状是( len(data)-seq_len-pred_len, pred_len, 1)
          # 该3D形状是用于序列模型，例如LSTM、GRU的标准的3D形状，分别代表(样本数，时间步长，特征数)
      return np.array(xs), np.array(ys)
  
  # 构造测试集时序样本，使得没有重复预测点
  def create_test_multi_step_sequences(data, seq_len=30, pred_len=30):
      xs, ys = [], []
      # 30个一组
      n_blocks = (len(data) - pred_len) // seq_len
      for i in range(n_blocks):
          # 这里，每个x都是一个形状为(seq_len,1)的二维数组;y也同理
          x = data[i*seq_len:(i+1)*seq_len]
          y = data[(i+1)*seq_len:(i+1)*seq_len+pred_len]
          xs.append(x)
          ys.append(y)
          # 因此，xs的形状是( len(data)-seq_len-pred_len, seq_len, 1)
          # ys的形状是( len(data)-seq_len-pred_len, pred_len, 1)
          # 该3D形状是用于序列模型，例如LSTM、GRU的标准的3D形状，分别代表(样本数，时间步长，特征数)
      return np.array(xs), np.array(ys)
  
  ~~~

  由于原本滑动步长为1的方式可以增加我们训练集的样本数，所以我们训练集还是采用滑动步长为1的方式来构造

  而测试集则采用滑动步长为pred_len的方式来构造

  > 其实后面通过训练实验发现，好像还是训练集采用与测试集相同的构造方式的预测精度更加理想
  >
  > 可能是数据关联一致性的原因

* 构建数据集迭代器

  ~~~python
  # 自定义pytorch数据集
  class TimeSeriesDataSet(Dataset):
      def __init__(self, X, y):
          self.X = torch.tensor(X, dtype=torch.float32)
          self.y = torch.tensor(y, dtype=torch.float32)
  
      def __len__(self):
          return len(self.X)
  
      def __getitem__(self, idx):
          return self.X[idx], self.y[idx]
  
  def build_train_test_loader(train_scaled, test_scaled):
      # 生成时序样本
      train_X, train_y = create_test_multi_step_sequences(train_scaled, SEQ_LEN, PRED_LEN)
      test_X, test_y = create_test_multi_step_sequences(test_scaled, SEQ_LEN, PRED_LEN)
  
      # 构造数据集,训练集测试集的划分已经划分完了
      train_dataset = TimeSeriesDataSet(train_X, train_y)
      test_dataset = TimeSeriesDataSet(test_X, test_y)
  
      # 训练集每个epoch打乱样本顺序，有助于泛化
      train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)
      # 测试集保持时间顺序以便于后续绘图对比
      test_loader = DataLoader(test_dataset, batch_size=32, shuffle=False)
      # 该数据迭代器，返回的数据形状是:
      # X_batch:(batch_size, seq_len, features_len)即(32, 30, 1)
      # y_batch:(batch_size, pred_len, features_len)即(32, 30, 1)
      return train_loader, test_loader
  
  ~~~

  这里定义的TimeSeriesDataSet类与单步预测的定义一致
  
  构造数据集迭代器与前面类似，只不过数据集划分不在这里了

## 定义LSTM多步预测模型

~~~python
# 定义LSTM模型
# LSTM模型期望输入是一个3D张量，形状为(batch_size, seq_len, input_size)也就是序列样本、时间步长、特征数量
class LSTMModel(nn.Module):
    # 继承nn.Module，默认输入特征维度latency只有1，隐藏层大小64，LSTM层数2
    def __init__(self, input_size=1, hidden_size=128, num_layers=2, output_size=30, dropout=0.4):
        super(LSTMModel, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers

        # 创建一个LSTM层，输入形状为(batch，seq_len，features);dropout=0.2在除了最后一层外的所有层之间添加dropout，防止过拟合
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True, dropout=dropout)

        # 另外添加一个全连接隐藏层+ReLu
        # self.fc_hidden = nn.Linear(hidden_size, hidden_size)
        # self.relu = nn.ReLU()
        # self.fc_dropout = nn.Dropout(dropout)

        # 全连接层，即输出层，将LSTM最后一个时间步的输出映射到预测值，通常是1维
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x):
        batch_size = x.size(0)
        # 手动初始化LSTM的初始隐藏状态h0和细胞状态c0
        # 形状(num_layers, batch_size, hidden_size)
        # 使用.to(x.device)确保它们与输入张量在同一个设备
        h0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)

        # 前向传播通过LSTM，得到所有时间步的输出out，形状(bath_size, seq_len, hidden_size)
        lstm_out, _ =self.lstm(x, (h0, c0))
        # 只取最后一个时间步的输出，因为这是单步预测，然后送入全连接层输出最终预测; 形状(batch_size, hidden_size)
        last_hidden = lstm_out[:, -1, :]  # 表示':, -1, :'表示，第一维度取全部，第二维度取最后一个，第三维度取全部

        # 取最后一个时间步后，经过全连接隐藏层再到Relu，再到dropout，最后输出
        # fc_hidden_out = self.fc_hidden(last_hidden)
        # relu_out = self.relu(fc_hidden_out)
        # fc_dropout_out = self.fc_dropout(relu_out)

        # 全连接层输出未来pred_len个预测值,形状是(batch_size, output_size)
        predictions = self.fc(last_hidden)
        # 增加一个维度，变为3D,形状为(batch_size, output_size, 1)
        # unsqueeze(-1)表示在最后一个维度后面插入一个大小为1的新维度
        predictions = predictions.unsqueeze(-1)
        return predictions
~~~

* 由于这次是多步预测了，所以将模型的输出大小`output_size`定义为30

  在代码中可以看到，在注释部分，还自己添加了全连接隐藏层、Relu激活函数层、dropout层，这其实是应了论文中的要求，但是经过一次实验，结果似乎并不如原本的理想（也可能是次数少了，建议有空继续实验）

  所以最后将该部分代码注释了

  在`forward`中，有对应新添加层的改动，但是最后也一并注释掉了

* 最后的unsqueeze(-1)

  **这是为了让LSTM的输出形状与样本y_batch的形状一致；由于last_hidden取了最后一个时间步，使得形状变为(batch_size, hidden_size)**

  **再经过全连接层使得形状变为(batch_size, output_size),而实际上y_batch的形状是(batch_size, pred_len ,1),并且，这里output_size与pred_len相等，因此采用unsqueeze(-1)将形状变为**

  **(batch_size, output_size, 1)**

## 模型训练与验证

* 模型训练

  ~~~python
  def train_and_test_model(train_loader, test_loader, model_name):
      # 训练模型
      # 自动选择设备，有cuda用cuda，没有用cpu
      device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
      # 实例化模型并移动到指定设备
      model = LSTMModel(hidden_size=HIDDEN_SIZE, num_layers=NUM_LAYERS, dropout= DROPOUT).to(device)
  
      # 定义损失函数均方误差和优化器Adam
      criterion = nn.MSELoss()
      optimizer = torch.optim.Adam(model.parameters(), lr=LR)
  
      for epoch in range(EPOCHS):
          # 启动Dropout/BatchNorm等训练行为
          model.train()
          total_loss = 0
          for X_batch, y_batch in train_loader:
              # 将每一批数据移动到设备上
              X_batch, y_batch = X_batch.to(device), y_batch.to(device)
  
              optimizer.zero_grad()   # 清除梯度
              y_pred = model(X_batch) # 前向传播
  
              loss = criterion(y_pred, y_batch)   # 计算损失
              loss.backward()     # 反向传播
              optimizer.step()    # 更新参数
  
              total_loss += loss.item()
          print(f"Epoch {epoch + 1}/{EPOCHS}, Train Loss: {total_loss / len(train_loader):.6f}")
  
      torch.save(model.state_dict(), f"model/lstm_{model_name}.pth")
      # torch.save(standardScaler, "model/standardScaler.pkl")
      joblib.dump(scaler, f"model/scaler_{model_name}.pkl")
  
  ~~~

  模型训练部分与单步预测并无不同，除了最后保存Scaler的地方，原本的方式说是保存Scaler不太合适，还是使用joblib来保存Scaler比较合适

* 模型验证

  ~~~python
      #测试与预测
      # 关闭Dropout，启动评估模式
      model.eval()
      preds, trues = [], []
  
      # 禁用梯度计算，节省内存
      with torch.no_grad():
          for X_batch, y_batch in test_loader:
              X_batch, y_batch = X_batch.to(device), y_batch.to(device)
              y_pred = model(X_batch)
  
              loss = criterion(y_pred, y_batch)
              preds.append(y_pred.cpu().numpy())  # 将GPU张量转回Numpy
              trues.append(y_batch.cpu().numpy())
  
          # 将多个batch的预测结果拼接成完整数组
          preds = np.concatenate(preds, axis=0)
          trues = np.concatenate(trues, axis=0)
  
      # 反归一化;使用之前拟合的scaler将归一化的预测值还原为原始尺度，便于解释和绘图
      preds_2d = preds.reshape(-1, 1)
      trues_2d = trues.reshape(-1, 1)
      preds_inv = scaler.inverse_transform(preds_2d)
      trues_inv = scaler.inverse_transform(trues_2d)
      return preds_inv, trues_inv
  ~~~

  在模型验证部分与单步预测有较多不同

  1. 其实我感觉，y_batch不用.to(device)，只是拿他来做对比，不用转移到GPU上，但是为了一致性，还是放上去了

  2. 在concatenate部分，preds和trues的形状都是(seq_num, seq_len, 1)的形状，但是由于测试集我们采用的是步长为pred_len的滑动，是无重复的，因此preds的数据形式应该是

     ~~~python
     [
         [ [31],[32],...,[60] ],
         [ [61],[62],...,[90] ],
         ....
     ]
     ~~~

  3. 将preds reshape为二维，参数含义就是保持最后一维为1，前面自动计算，这也就实现了将每个数据连接起来，变为

     ~~~python
         [ [31],[32],...,[60],[61],[61],...,[90],... ]
     ~~~

     这样方便绘图

* 画图

  ~~~python
  def consequence_show(preds_inv, trues_inv, model_name):
      # 结果可视化
      plt.figure(figsize=(10, 6))
      plt.plot(trues_inv, label="True (30 future steps")
      plt.plot(preds_inv, label="Predicated (30 future steps)")
      plt.title("LSTM Latency Predication Multi Step 30")
      plt.xlabel("Time Step")
      plt.ylabel("Latency")
      plt.figtext(0.2, 0.02, f"model_name={model_name}, hidden_size={HIDDEN_SIZE}, num_layers={NUM_LAYERS}, dropout={DROPOUT}, Lr={LR}, strategic=2*Test")
      plt.legend()
      plt.show()
  
  ~~~

  这里多加了一行注释figtext，用于标识每次不同参数的测试精度结果

## 整体代码

~~~python
import joblib
import pandas as pd
import numpy as np
from sklearn.preprocessing import MinMaxScaler
from sklearn.preprocessing import RobustScaler
from sklearn.preprocessing import StandardScaler
import torch
from torch import nn
from torch.utils.data import Dataset, DataLoader
import matplotlib.pyplot as plt
from global_config import NODES

# 归一化
# scaler = MinMaxScaler()     # 对数据进行归一化，缩放到0-1区间，防止数值过大影响神经网络训练稳定性
# scaler = RobustScaler()
scaler = StandardScaler()
SEQ_LEN = 30    # 定义时间窗口长度，即每次输入LSTM的连续时间步为30，例如用前30秒的延迟值预测第31秒的延迟
PRED_LEN = 30
EPOCHS = 100
HIDDEN_SIZE = 128
NUM_LAYERS = 2
DROPOUT = 0.3
LR = 0.0003

# 构造训练集时序样本
def create_train_multi_step_sequences(data, seq_len=30, pred_len=30):
    xs, ys = [], []
    # 30个一组
    for i in range(len(data) - seq_len - pred_len):
        # 这里，每个x都是一个形状为(seq_len,1)的二维数组;y也同理
        x = data[i:i+seq_len]   # 取i到i+seq_len-1的数据作为输入
        y = data[i+seq_len:i+seq_len+pred_len]     # 取第i+seq_len个数组作为标签
        xs.append(x)
        ys.append(y)
        # 因此，xs的形状是( len(data)-seq_len-pred_len, seq_len, 1)
        # ys的形状是( len(data)-seq_len-pred_len, pred_len, 1)
        # 该3D形状是用于序列模型，例如LSTM、GRU的标准的3D形状，分别代表(样本数，时间步长，特征数)
    return np.array(xs), np.array(ys)

# 构造测试集时序样本，使得没有重复预测点
def create_test_multi_step_sequences(data, seq_len=30, pred_len=30):
    xs, ys = [], []
    # 30个一组
    n_blocks = (len(data) - pred_len) // seq_len
    for i in range(n_blocks):
        # 这里，每个x都是一个形状为(seq_len,1)的二维数组;y也同理
        x = data[i*seq_len:(i+1)*seq_len]
        y = data[(i+1)*seq_len:(i+1)*seq_len+pred_len]
        xs.append(x)
        ys.append(y)
        # 因此，xs的形状是( len(data)-seq_len-pred_len, seq_len, 1)
        # ys的形状是( len(data)-seq_len-pred_len, pred_len, 1)
        # 该3D形状是用于序列模型，例如LSTM、GRU的标准的3D形状，分别代表(样本数，时间步长，特征数)
    return np.array(xs), np.array(ys)

def data_prepare_and_partition(path):
    # 读取数据
    # 从CSV读取数据，并将time列解析为datetime类型
    df = pd.read_csv(path, parse_dates=['time'])
    # 按时间排序，否则打乱依赖关系
    df = df.sort_values('time')

    # 判断是要训练集数据还是测试集数据,如果是训练集，取前80%，测试集取后20%
    train_size = int(0.8 * len(df))
    train_df = df[:train_size]
    test_df = df[train_size:]

    print("Train:")
    print("Latency min:", train_df['latency'].min())
    print("Latency max:", train_df['latency'].max())
    print("Latency mean:",train_df['latency'].mean())
    print("Latency std:", train_df['latency'].std())
    print("Test:")
    print("Latency min:", test_df['latency'].min())
    print("Latency max:", test_df['latency'].max())
    print("Latency mean:",test_df['latency'].mean())
    print("Latency std:", test_df['latency'].std())

    # 只取latency列,并reshape为2维，因为MinMaxScaler要求输入为2D
    # 这里，将latency重塑为一个二维数组，如果原始有N行，那么values的形状是(N,1)，即N行1列
    # 参数含义：-1是占位符，表示自动计算这个维度的大小； 1表示第二维度列数必须是1
    train_values = train_df['latency'].values.reshape(-1, 1)
    test_values = test_df['latency'].values.reshape(-1, 1)

    # 经过这一步，形状不变
    # 使用MinMaxScaler将latency数据归一化到0-1区间;这里使用了fit_transform,如果有多个文件，建议只对第一个fit，后面用transform
    train_scaled = scaler.fit_transform(train_values)
    test_scaled = scaler.fit_transform(test_values)

    return train_scaled, test_scaled

# 自定义pytorch数据集
class TimeSeriesDataSet(Dataset):
    def __init__(self, X, y):
        self.X = torch.tensor(X, dtype=torch.float32)
        self.y = torch.tensor(y, dtype=torch.float32)

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        return self.X[idx], self.y[idx]

def build_train_test_loader(train_scaled, test_scaled):
    # 生成时序样本
    train_X, train_y = create_test_multi_step_sequences(train_scaled, SEQ_LEN, PRED_LEN)
    test_X, test_y = create_test_multi_step_sequences(test_scaled, SEQ_LEN, PRED_LEN)

    # 构造数据集,训练集测试集的划分已经划分完了
    train_dataset = TimeSeriesDataSet(train_X, train_y)
    test_dataset = TimeSeriesDataSet(test_X, test_y)

    # 训练集每个epoch打乱样本顺序，有助于泛化
    train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)
    # 测试集保持时间顺序以便于后续绘图对比
    test_loader = DataLoader(test_dataset, batch_size=32, shuffle=False)
    # 该数据迭代器，返回的数据形状是:
    # X_batch:(batch_size, seq_len, features_len)即(32, 30, 1)
    # y_batch:(batch_size, pred_len, features_len)即(32, 30, 1)
    return train_loader, test_loader

# 定义LSTM模型
# LSTM模型期望输入是一个3D张量，形状为(batch_size, seq_len, input_size)也就是序列样本、时间步长、特征数量
class LSTMModel(nn.Module):
    # 继承nn.Module，默认输入特征维度latency只有1，隐藏层大小64，LSTM层数2
    def __init__(self, input_size=1, hidden_size=128, num_layers=2, output_size=30, dropout=0.4):
        super(LSTMModel, self).__init__()
        self.hidden_size = hidden_size
        self.num_layers = num_layers

        # 创建一个LSTM层，输入形状为(batch，seq_len，features);dropout=0.2在除了最后一层外的所有层之间添加dropout，防止过拟合
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True, dropout=dropout)

        # 另外添加一个全连接隐藏层+ReLu
        # self.fc_hidden = nn.Linear(hidden_size, hidden_size)
        # self.relu = nn.ReLU()
        # self.fc_dropout = nn.Dropout(dropout)

        # 全连接层，即输出层，将LSTM最后一个时间步的输出映射到预测值，通常是1维
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x):
        batch_size = x.size(0)
        # 手动初始化LSTM的初始隐藏状态h0和细胞状态c0
        # 形状(num_layers, batch_size, hidden_size)
        # 使用.to(x.device)确保它们与输入张量在同一个设备
        h0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)
        c0 = torch.zeros(self.num_layers, batch_size, self.hidden_size).to(x.device)

        # 前向传播通过LSTM，得到所有时间步的输出out，形状(bath_size, seq_len, hidden_size)
        lstm_out, _ =self.lstm(x, (h0, c0))
        # 只取最后一个时间步的输出，因为这是单步预测，然后送入全连接层输出最终预测; 形状(batch_size, hidden_size)
        last_hidden = lstm_out[:, -1, :]  # 表示':, -1, :'表示，第一维度取全部，第二维度取最后一个，第三维度取全部

        # 取最后一个时间步后，经过全连接隐藏层再到Relu，再到dropout，最后输出
        # fc_hidden_out = self.fc_hidden(last_hidden)
        # relu_out = self.relu(fc_hidden_out)
        # fc_dropout_out = self.fc_dropout(relu_out)

        # 全连接层输出未来pred_len个预测值,形状是(batch_size, output_size)
        predictions = self.fc(last_hidden)
        # 增加一个维度，变为3D,形状为(batch_size, output_size, 1)
        # unsqueeze(-1)表示在最后一个维度后面插入一个大小为1的新维度
        predictions = predictions.unsqueeze(-1)
        return predictions

def train_and_test_model(train_loader, test_loader, model_name):
    # 训练模型
    # 自动选择设备，有cuda用cuda，没有用cpu
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    # 实例化模型并移动到指定设备
    model = LSTMModel(hidden_size=HIDDEN_SIZE, num_layers=NUM_LAYERS, dropout= DROPOUT).to(device)

    # 定义损失函数均方误差和优化器Adam
    criterion = nn.MSELoss()
    optimizer = torch.optim.Adam(model.parameters(), lr=LR)

    for epoch in range(EPOCHS):
        # 启动Dropout/BatchNorm等训练行为
        model.train()
        total_loss = 0
        for X_batch, y_batch in train_loader:
            # 将每一批数据移动到设备上
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)

            optimizer.zero_grad()   # 清除梯度
            y_pred = model(X_batch) # 前向传播

            loss = criterion(y_pred, y_batch)   # 计算损失
            loss.backward()     # 反向传播
            optimizer.step()    # 更新参数

            total_loss += loss.item()
        print(f"Epoch {epoch + 1}/{EPOCHS}, Train Loss: {total_loss / len(train_loader):.6f}")

    torch.save(model.state_dict(), f"model/lstm_{model_name}.pth")
    # torch.save(standardScaler, "model/standardScaler.pkl")
    joblib.dump(scaler, f"model/scaler_{model_name}.pkl")

    #测试与预测
    # 关闭Dropout，启动评估模式
    model.eval()
    preds, trues = [], []

    # 禁用梯度计算，节省内存
    with torch.no_grad():
        for X_batch, y_batch in test_loader:
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)
            y_pred = model(X_batch)

            loss = criterion(y_pred, y_batch)
            preds.append(y_pred.cpu().numpy())  # 将GPU张量转回Numpy
            trues.append(y_batch.cpu().numpy())

        # 将多个batch的预测结果拼接成完整数组
        preds = np.concatenate(preds, axis=0)
        trues = np.concatenate(trues, axis=0)

    # 反归一化;使用之前拟合的scaler将归一化的预测值还原为原始尺度，便于解释和绘图
    preds_2d = preds.reshape(-1, 1)
    trues_2d = trues.reshape(-1, 1)
    preds_inv = scaler.inverse_transform(preds_2d)
    trues_inv = scaler.inverse_transform(trues_2d)
    return preds_inv, trues_inv

def consequence_show(preds_inv, trues_inv, model_name):
    # 结果可视化
    plt.figure(figsize=(10, 6))
    plt.plot(trues_inv, label="True (30 future steps")
    plt.plot(preds_inv, label="Predicated (30 future steps)")
    plt.title("LSTM Latency Predication Multi Step 30")
    plt.xlabel("Time Step")
    plt.ylabel("Latency")
    plt.figtext(0.2, 0.02, f"model_name={model_name}, hidden_size={HIDDEN_SIZE}, num_layers={NUM_LAYERS}, dropout={DROPOUT}, Lr={LR}, strategic=2*Test")
    plt.legend()
    plt.show()

def train_get_model_scaler(node_from, node_to):
    model_name = f"{node_from}2{node_to}"
    csv_path = f"latency_csv/{model_name}.csv"
    train_scaled, test_scaled = data_prepare_and_partition(csv_path)
    train_loader, test_loader = build_train_test_loader(train_scaled, test_scaled)
    preds_inv, trues_inv = train_and_test_model(train_loader, test_loader, model_name)
    consequence_show(preds_inv, trues_inv, model_name)
    print(f"Model And Scaler:{model_name} train complete! Save as file!")


if __name__ == '__main__':
    node_num = len(NODES)
    for i in range(node_num):
        nodeFrom = NODES[i]
        for nodeTo in NODES[i+1:node_num]:
            train_get_model_scaler(nodeFrom, nodeTo)
    # print("PyTorch version:", torch.__version__)
    # print("CUDA available:", torch.cuda.is_available())
    # print("CUDA version:", torch.version.cuda)
    # print("Current device:", torch.cuda.current_device())
    # print("Device name:", torch.cuda.get_device_name(0) if torch.cuda.is_available() else "No GPU")

~~~

