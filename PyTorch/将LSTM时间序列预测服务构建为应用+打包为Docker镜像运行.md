---
tags:
  - 服务应用构建
  - 模型应用
  - CUDA与Docker
  - 模型Docker应用
---

# 将模型预测服务构建为应用

* 其实不仅将其构建为了Docker镜像， 还构建了一个以Deployment和Service为基础资源的Operator，详见https://github.com/WyYong7240/LSTMServerOperator

## 保存与加载模型

* 保存模型

  其实就是这两句

  ~~~python
      torch.save(model.state_dict(), f"model/lstm_{model_name}.pth")
      # torch.save(standardScaler, "model/standardScaler.pkl")
      joblib.dump(scaler, f"model/scaler_{model_name}.pkl")
  ~~~

  具体参见[[LSTM长短期记忆网络：用于时间序列预测#LSTM多步预测模型#]]

* 加载模型(load_model.py)

  ~~~python
  import torch
  import joblib
  from lstm_multi_step import LSTMModel, HIDDEN_SIZE, NUM_LAYERS, DROPOUT
  
  def load_trained_model(model_path, scaler_path):
      # 实例化模型：参数需要与训练时一致
      device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
      # 实例化模型并移动到指定设备
      model = LSTMModel(hidden_size=HIDDEN_SIZE, num_layers=NUM_LAYERS, dropout=DROPOUT)
  
      # 加载权重
      model.load_state_dict(torch.load(model_path, map_location=device))
      # 模型移动到对应设备
      model.to(device)
      model.eval()
  
      # 加载scaler
      scaler = joblib.load(scaler_path)
      print(f"模型和Scaler加载成功，当前设备{device}")
      return model, scaler
  ~~~
  
  上述的导入`lstm_multi_step`就是文件[[LSTM长短期记忆网络：用于时间序列预测#LSTM多步预测模型#整体代码]]

## 构建为Flask应用

### 获取请求发送的过去的30个时间步的数据

~~~python
def get_input_tensor(scaler_name):
    data = request.get_json()
    if not data or "data" not in data:
        return jsonify({"error": "Missing 'data' field"}), 400
    # 提取输入的数据
    values = np.array(data["data"], dtype=float).reshape(-1, 1)
    if len(values) != 30:
        return jsonify({"error": "Data length is not 30"}), 400
    # 缩放
    scaled_data = SCALERS[scaler_name].transform(values).reshape(1, SEQ_LEN, 1)
    input_tensor = torch.tensor(scaled_data, dtype=torch.float32).to(DEVICE)
    return input_tensor
~~~

1. 首先检查发送的请求中是否有data字段，该字段存放了过去30个时间步的延迟数据，是用于预测未来30个时间步延迟的依据

   如果没有，返回错误，并且错误码为400

2. 利用json解析，获取30个时间步的数据，并且检查是否真的有30个时间步的数据

3. 最后将数据利用对应的Scaler进行归一化，并将形状reshape为(1, SEQ_LEN, 1)；再将数据初始化为tensor

### 构建应用的不同服务访问端点

以预测master节点到node1节点的延迟为例

~~~python
@app.route('/predict/master2node1', methods=['POST'])
def predict_master2node1():
    try:
        # 获取数据
        input_tensor = get_input_tensor("master2node1")
        # 推理
        with torch.no_grad():
            pred_scaled = MODELS["master2node1"](input_tensor)
            pred_scaled = pred_scaled.cpu().numpy()
        # 反归一化
        pred_inv = SCALERS["master2node1"].inverse_transform(pred_scaled.reshape(-1, 1))
        return jsonify({"prediction": pred_inv.tolist()})
    except Exception as e:
        return jsonify({"error":str(e)}), 500
~~~

1. 先获取要输入的tensor数据，利用main中初始化的对应的模型model，进行预测

2. 与模型的验证部分类似，在获取到预测数据之后，需要将预测值先reshape为二维，在反归一化到正常尺度的预测值

   如果发生异常，就返回异常值

### 模型初始化与服务监听

~~~python
MODEL_SCALER_PATH = "model/"
DEVICE = torch.device("cuda" if torch.cuda.is_available() else "cpu")
MODELS = dict()
SCALERS = dict()

if __name__ == '__main__':
    node_num = len(NODES)
    for i in range(node_num):
        nodeFrom = NODES[i]
        for nodeTo in NODES[i+1:node_num]:
            model_name = f"{nodeFrom}2{nodeTo}"
            model_path = MODEL_SCALER_PATH + f"lstm_{model_name}.pth"
            scaler_path = MODEL_SCALER_PATH + f"scaler_{model_name}.pkl"
            model, scaler = load_trained_model(model_path, scaler_path)
            MODELS[model_name] = model
            SCALERS[model_name] = scaler
            print(f"{model_name}的Model和Scaler加载成功！")
    app.run(host='0.0.0.0', port=8080, debug=False)
~~~

1. MODEL_SCALER_PATH是改项目中，存放训练好的模型和Scaler的路径

2. DEVICE，由于不同的环境使用不同的设备，因此使用全局变量来存放当前环境下使用的设备名

3. MODELS，针对不同节点之间延迟预测使用不同的模型，因此使用字典存放不同节点之间的延迟预测模型

4. SCALERS，与MODELS同理

5. **在启动服务监听时，记得将host改为0.0.0.0，如果是127.0.0.1那么当打包为容器从容器外部访问时就会访问不到了**

6. 服务访问方式-使用命令行

   ~~~sh
   curl -X POST http://127.0.0.1:8080/predict/master2node1 -H "Content-Type: application/json" -d "{\"data\": [0.1, 0.2, 0.3, 0.4, 0.5,0.6, 0.7, 0.8, 0.9, 1.0,1.1, 1.2, 1.3, 1.4, 1.5,1.6, 1.7, 1.8, 1.9, 2.0,2.1, 2.2, 2.3, 2.4, 2.5,2.6, 2.7, 2.8, 2.9, 3.0]}"
   ~~~

   

   

### 整体代码(predict_app_flask.py)

> 参考global_config.py
>
> ~~~python
> NODES = ["master","node1","node2"]
> ~~~
>

~~~python
import numpy as np
import torch
from flask import Flask, request, jsonify
from flask_cors import CORS
from load_model import load_trained_model
from lstm_multi_step import SEQ_LEN
from global_config import NODES

app = Flask(__name__)
CORS(app)

MODEL_SCALER_PATH = "model/"
DEVICE = torch.device("cuda" if torch.cuda.is_available() else "cpu")
MODELS = dict()
SCALERS = dict()

def get_input_tensor(scaler_name):
    data = request.get_json()
    if not data or "data" not in data:
        return jsonify({"error": "Missing 'data' field"}), 400
    # 提取输入的数据
    values = np.array(data["data"], dtype=float).reshape(-1, 1)
    if len(values) != 30:
        return jsonify({"error": "Data length is not 30"}), 400
    # 缩放
    scaled_data = SCALERS[scaler_name].transform(values).reshape(1, SEQ_LEN, 1)
    input_tensor = torch.tensor(scaled_data, dtype=torch.float32).to(DEVICE)
    return input_tensor

@app.route('/predict/master2node1', methods=['POST'])
def predict_master2node1():
    try:
        # 获取数据
        input_tensor = get_input_tensor("master2node1")
        # 推理
        with torch.no_grad():
            pred_scaled = MODELS["master2node1"](input_tensor)
            pred_scaled = pred_scaled.cpu().numpy()
        # 反归一化
        pred_inv = SCALERS["master2node1"].inverse_transform(pred_scaled.reshape(-1, 1))
        return jsonify({"prediction": pred_inv.tolist()})
    except Exception as e:
        return jsonify({"error":str(e)}), 500

@app.route('/predict/master2node2', methods=['POST'])
def predict_master2node2():
    try:
        # 获取数据
        input_tensor = get_input_tensor("master2node2")
        # 推理
        with torch.no_grad():
            pred_scaled = MODELS["master2node2"](input_tensor)
            pred_scaled = pred_scaled.cpu().numpy()
        # 反归一化
        pred_inv = SCALERS["master2node2"].inverse_transform(pred_scaled.reshape(-1, 1))
        return jsonify({"prediction": pred_inv.tolist()})
    except Exception as e:
        return jsonify({"error":str(e)}), 500

@app.route('/predict/node12node2', methods=['POST'])
def predict_node22node1():
    try:
        # 获取数据
        input_tensor = get_input_tensor("node12node2")
        # 推理
        with torch.no_grad():
            pred_scaled = MODELS["node12node2"](input_tensor)
            pred_scaled = pred_scaled.cpu().numpy()
        # 反归一化
        pred_inv = SCALERS["node12node2"].inverse_transform(pred_scaled.reshape(-1, 1))
        return jsonify({"prediction": pred_inv.tolist()})
    except Exception as e:
        return jsonify({"error":str(e)}), 500

@app.route('/health', methods=['GET'])
def health():
    return jsonify({"status": "ok"})

@app.route('/shutdown', methods=['POST'])
def shutdown():
    func = request.environ.get('werkzeug.server.shutdown')
    if func is None:
        return jsonify({"error": "Not running with the Werkzeug Server"}), 500
    func()
    return jsonify({"message": "Server shutting down..."})

if __name__ == '__main__':
    node_num = len(NODES)
    for i in range(node_num):
        nodeFrom = NODES[i]
        for nodeTo in NODES[i+1:node_num]:
            model_name = f"{nodeFrom}2{nodeTo}"
            model_path = MODEL_SCALER_PATH + f"lstm_{model_name}.pth"
            scaler_path = MODEL_SCALER_PATH + f"scaler_{model_name}.pkl"
            model, scaler = load_trained_model(model_path, scaler_path)
            MODELS[model_name] = model
            SCALERS[model_name] = scaler
            print(f"{model_name}的Model和Scaler加载成功！")
    app.run(host='0.0.0.0', port=8080, debug=False)

~~~

# 将模型预测应用打包为Docker镜像

## 生成requirements.txt并编写Dockerfile

### 生成python项目的requirements.txt

1. 方式1，使用`pip freeze > requirements.txt`

   但是这会列出当前环境中所有已经安装的包及其精确的版本号，包含了一些项目不需要的包

2. 方式2，使用pipreqs

   先安装该工具

   ~~~sh
   pip install pipreqs
   ~~~

   在项目根目录下运行如下命令

   ~~~sh
   pipreqs ./ --encoding=utf8 --force
   ~~~

   导出的只包含项目实际需要的依赖包，建议使用此方法

3. `requirements.txt`

   ~~~txt
   Flask==3.1.2
   flask_cors==6.0.1
   joblib==1.5.2
   matplotlib==3.9.4
   numpy==2.0.2
   pandas==2.3.3
   scikit_learn==1.6.1
   torch==2.4.1    # 容器使用的torch依赖，CPU版本
   # torch==2.7.1+cu118 是在模型开发环境中使用的torch依赖
   --extra-index-url https://download.pytorch.org/whl/cpu
   ~~~

   

### 将requirements.txt文件中有关torch的依赖项替换

* 项目中依赖的torch包实际上是`torch==2.7.1+cu118`这是GPU版本，但是打包为容器镜像之后，肯能就无法使用GPU了，改用CPU版本`torch==2.7.1`

  但是后来发现CPU版本似乎并没有2.7.1版本，经过查询，得到`torch==2.4.1`

  但是后来构建镜像时，发现在`pip install -r requirements.txt`时，依然下载了许多与CUDA有关的内容

  ~~~txt
  => [4/4] RUN pip install --no-cache-dir -r requirements.txt                                                                                                                                                             143.6s 
   => => #      ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 393.1/393.1 MB 24.7 MB/s eta 0:00:00                                                                                                                                       
   => => # Collecting nvidia-cusolver-cu12==11.7.1.2                                                                                                                                                                              
   => => #   Downloading nvidia_cusolver_cu12-11.7.1.2-py3-none-manylinux2014_x86_64.manylinux_2_17_x86_64.whl (158.2 MB)                                                                                                         
   => => #      ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 158.2/158.2 MB 26.4 MB/s eta 0:00:00                                                                                                                                       
   => => # Collecting nvidia-cufft-cu12==11.3.0.4                                                                                                                                                                                 
   => => #   Downloading nvidia_cufft_cu12-11.3.0.4-py3-none-manylinux2014_x86_64.manylinux_2_17_x86_64.whl (200.2 MB)                                               
  ~~~

  并且构建完成的镜像也是非常大的
  <img src="https://typora2icture.oss-cn-beijing.aliyuncs.com/img2/202511051833630.png" alt="image.png" style="zoom:67%;" />

  这是因为pytorch默认安装带CUDA的完整版本，体积非常大，在CPU-Only的环境中用不着，因此将依赖改为
  
  ~~~txt
  torch==2.4.1    # 容器使用的torch依赖，CPU版本
  --extra-index-url https://download.pytorch.org/whl/cpu
  ~~~
  
  **构建出的镜像只有1.3GB左右**
  
  但是同时需要注意，torch版本与其他依赖包之间的兼容关系
  
  

### 编写Dockerfile

~~~dockerfile
# 1️⃣ 基础镜像
FROM python:3.9-slim

# 2️⃣ 工作目录
WORKDIR /app

# 3️⃣ 拷贝项目文件
COPY . /app

# 4️⃣ 安装依赖
RUN pip install --no-cache-dir -r requirements.txt

# 5️⃣ 暴露端口
EXPOSE 8080

# 6️⃣ 启动命令
CMD ["python", "predict_app_flask.py"]
~~~

1. 如果后面想要使用模型参数的热更新，需要替换容器项目目录`app/model`下的文件，需要用到容器挂载目录相关，具体参考[[构建容器镜像#容器挂载解析]]
1. 启动命令，注意将`predict_app_flask.py`替换为Flask应用实际所在文件的文件名
