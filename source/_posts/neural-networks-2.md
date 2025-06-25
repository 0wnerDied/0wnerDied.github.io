---
title: 从零开始的神经网络学习(二)：实用技巧与进阶
categories:
  - Machine learning
tags:
  - NN
abbrlink: 38196
date: 2025-06-25 01:35:00
mathjax: true
---

在[上一篇文章](https://0wnerdied.github.io/Machine-learning/36604.html)中，我们从零开始构建了一个简单的神经网络，并理解了前向传播、反向传播和梯度下降等核心概念。然而，要让神经网络在现实世界的问题中高效工作，我们还需要掌握更多的工具和技巧。

这篇文章将作为第二部分，专注于第一部分中未能详尽涵盖的几个关键领域：

- 我将用上一篇文章中的代码来训练一个网络，解决一个经典问题，并观察损失函数的变化。
- 除了 ReLU，我还将介绍 Sigmoid 和 Tanh 等其他常用激活函数，并讨论如何为输出层选择合适的激活函数。
- 探讨权重初始化的重要性，并介绍批量归一化 (Batch Normalization) 和 Dropout 等强大的技术。
- 最后，分享一些关于调试神经网络和选择超参数的实用技巧。

## 训练一个 XOR 网络

理论需要实践来检验。让我们使用上一篇文章中定义的 `NeuralNetwork` 类来解决经典的 XOR 问题。XOR 是一个非线性问题，单个神经元无法解决，因此很适合作为我们神经网络的测试案例。

XOR 的真值表如下：

| 输入 A | 输入 B | 输出 |
| :----: | :----: | :--: |
|   0    |   0    |  0   |
|   0    |   1    |  1   |
|   1    |   0    |  1   |
|   1    |   1    |  0   |

### 准备数据和网络

```python
# existing codes...
            avg_loss = total_loss / len(X)
            # 只在每1/10进度时输出一次
            if (epoch + 1) % max(1, epochs // 10) == 0 or epoch == 0 or epoch == epochs - 1:
                print(f"Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.6f}")
# existing codes...

# 1. 定义 XOR 数据集
X_train = np.array([[0, 0], [0, 1], [1, 0], [1, 1]])
y_train = np.array([[0], [1], [1], [0]])

# 2. 创建神经网络实例
# 2个输入节点，一个有2个节点的隐藏层，1个输出节点
nn = NeuralNetwork([2, 2, 1])
```

### 训练并观察损失

我们将使用较小的学习率和足够多的迭代次数来训练网络。

```python
# 3. 训练网络
# XOR 是一个非线性问题，需要足够的迭代次数和合适的学习率
print("开始训练XOR网络...")
nn.train(X_train, y_train, epochs=10000, learning_rate=0.005, batch_size=4)
print("训练完成。")

# 开始训练XOR网络...
# Epoch 1/10000, Loss: 0.499970
# Epoch 1000/10000, Loss: 0.249839
# Epoch 2000/10000, Loss: 0.249042
# Epoch 3000/10000, Loss: 0.244619
# Epoch 4000/10000, Loss: 0.224387
# Epoch 5000/10000, Loss: 0.177830
# Epoch 6000/10000, Loss: 0.113382
# Epoch 7000/10000, Loss: 0.023213
# Epoch 8000/10000, Loss: 0.001769
# Epoch 9000/10000, Loss: 0.000104
# Epoch 10000/10000, Loss: 0.000006
# 训练完成。
```
正如我们所见，损失 (Loss) 随着训练的进行而稳步下降，这表明我们的网络确实在学习如何解决 XOR 问题。

### 查看结果

```python
# 4. 进行预测并展示结果
print("\n对输入进行预测:")
for x_input in X_train:
    prediction = nn.predict(x_input.reshape(1, -1))
    print(f"输入: {x_input}, 预测输出: {prediction[0][0]:.4f}")

# 对输入进行预测:
# 输入: [0 0], 预测输出: 0.0027
# 输入: [0 1], 预测输出: 0.9981
# 输入: [1 0], 预测输出: 0.9981
# 输入: [1 1], 预测输出: 0.0028
```
~~预测值非常接近真实值，证明这个简单的神经网络框架是有效的。~~
其实这个结果是我精挑细选，训练了很多很多次才得到的成功预测结果，训练其实是有随机性的，上面的代码在我本地测试中成功率非常低，训练十次都不一定能有一次成功，极大概率会训练失败，可以说这是一个失败的网络实现。这时就要调整参数或者更换合适的激活函数。

## 其它的激活函数

前面我们只介绍了 ReLU。虽然它非常流行且有效，但了解其他激活函数以及如何为输出层做选择也同样重要。

### Sigmoid
Sigmoid 函数将任意实数压缩到 (0, 1) 区间内，所以它很适合用来表示概率。

- **公式**:
$$ \sigma(z) = \frac{1}{1 + e^{-z}} $$

- **Python 实现**:
  ```python
  import numpy as np


  def sigmoid(z):
      return 1 / (1 + np.exp(-z))


  print(sigmoid(np.array([-2, 0, 2])))
  # [0.11920292 0.5        0.88079708]
  ```

- **优点**: 输出在 (0, 1) 之间，平滑且易于求导。

- **缺点**:

  - **梯度消失**: 当输入非常大或非常小时，函数的导数趋近于0，导致梯度在反向传播时消失，使网络难以训练。
  - **输出不以0为中心**: 输出总是正数，这可能导致后续层权重更新时朝同一个方向移动，降低收敛速度。

### Tanh (双曲正切)
Tanh 函数会将输入压缩到 (-1, 1) 区间。

- **公式**:
$$ tanh(z) = \frac{e^z - e^{-z}}{e^z + e^{-z}} $$

- **Python 实现**:
  ```python
  import numpy as np


  def tanh(z):
      return np.tanh(z)


  print(tanh(np.array([-2, 0, 2])))
  # [-0.96402758  0.          0.96402758]
  ```

- **优点**:

  - **以0为中心**: 输出在 -1 和 1 之间，解决了 Sigmoid 的一个主要缺点。
  - 通常比 Sigmoid 收敛更快。

- **缺点**: 仍然存在梯度消失的问题。

### 如何为输出层选择激活函数？

输出层的激活函数选择至关重要，因为它决定了网络输出的格式。

- **二元分类 (Binary Classification)**: 当你预测两个类别之一时（例如，是猫/不是猫），使用 **Sigmoid** 函数。它输出一个0到1之间的值，可以解释为属于正类的概率。
- **多元分类 (Multi-class Classification)**: 当你在多个类别中选择一个时（例如，数字识别0-9），使用 **Softmax** 函数。它能将一组数字转换成概率分布，所有输出的总和为1。

- **Softmax 实现**:
  ```python
  import numpy as np


  def softmax(z):
      exp_z = np.exp(z - np.max(z, axis=1, keepdims=True))
      return exp_z / np.sum(exp_z, axis=1, keepdims=True)


  print(softmax(np.array([[2.0, 1.0, 0.1]])))
  # [[0.65900114 0.24243297 0.09856589]]
  ```

- **回归 (Regression)**: 当预测一个连续值时（例如，房价），输出层**不使用任何激活函数**。这样，网络就可以输出任意范围的数值。

## 优化与正则化技巧

为了构建更强大、更稳定的神经网络，我们需要一些高级的优化和正则化技术。

### 权重初始化的重要性
我们在第一部分中用简单的 `np.random.randn() * 0.01` 来初始化权重。这虽然行得通，但不是最好的解决方案。糟糕的权重初始化可能导致梯度消失或梯度爆炸，即梯度在反向传播过程中变得过小或过大。

现代的初始化方法，如 **Xavier (Glorot) 初始化** 和 **He 初始化**，通过智能地根据上一层的神经元数量来调整初始权重的方差，从而确保信号在网络中更稳定地传播，显著加快训练速度并提高性能。

-   **Xavier 初始化**: 通常与 Sigmoid 或 Tanh 激活函数配合使用。
-   **He 初始化**: 专为 ReLU 及其变体设计，是现代深度网络中的首选。

- **Xavier/He 初始化示例**:
  ```python
  import numpy as np

  # Xavier 初始化 (适合Sigmoid/Tanh)
  fan_in, fan_out = 64, 32
  xavier = np.random.randn(fan_in, fan_out) * np.sqrt(1.0 / fan_in)
  # He 初始化 (适合ReLU)
  he = np.random.randn(fan_in, fan_out) * np.sqrt(2.0 / fan_in)
  ```

### 批量归一化 (Batch Normalization)
在每个小批量数据通过网络时，对每一层的输入进行归一化（调整为均值为0，方差为1），然后再进行缩放和平移。

**好处**:

- **加速训练**: 允许使用更高的学习率。
- **稳定训练**: 减少了对权重初始化的敏感度。
- **轻微的正则化效果**: 由于是在小批量上计算均值和方差，引入的噪声可以起到类似 Dropout 的效果。

- **BatchNorm 示例**:
```python
  import numpy as np


  def batch_norm(x, gamma, beta, eps=1e-5):
      mu = np.mean(x, axis=0)
      var = np.var(x, axis=0)
      x_norm = (x - mu) / np.sqrt(var + eps)
      return gamma * x_norm + beta


  # x: (batch_size, features)
  # gamma/beta: 可学习参数，初始为1和0
```

### Dropout
Dropout 是一种简单而有效的正则化技术，用于防止网络过拟合。在训练过程中的每一步，它会以一定的概率 `p` 随机地“丢弃”网络中的一部分神经元。

这意味着网络不能依赖于任何一个特定的神经元，迫使它学习到更健壮、更冗余的特征表示。在测试时，所有神经元都会被使用，但它们的输出会按比例 `(1-p)` 缩小，以平衡训练时的丢弃行为。

- **Dropout 示例**:
  ```python
  import numpy as np


  def dropout(x, p):
      mask = (np.random.rand(*x.shape) > p).astype(float)
      return x * mask / (1 - p)


  # 训练时: x = dropout(x, p=0.5)
  # 推理时: 不用dropout
  ```

## 损失函数的选择

在二元分类任务中，输出层通常用 sigmoid 激活，最合适的损失函数是**二元交叉熵（Binary Cross Entropy, BCE）**，而不是均方误差（MSE）。

- **BCE 公式**，其中 $y$ 是真实标签（0或1），$p$ 是 sigmoid 输出概率：
$$
L = -[y \cdot \ln p + (1-y) \cdot \ln(1-p)]
$$

- **BCE 导数**：
$$
\frac{\partial L}{\partial p} = -\frac{y}{p} + \frac{1-y}{1-p}
$$

- **区别**：

  - MSE 在 sigmoid 饱和区间梯度更容易消失，收敛慢。
  - BCE 更适合概率输出，收敛快。

## 建议

### 调试神经网络的技巧
当神经网络不工作时，调试让人~~心旷神怡~~。下面是一些实用的检查步骤：

1. 先用一个非常小的网络（例如一个隐藏层，少量神经元）来**过拟合**一小部分训练数据（例如，仅10-20个样本）。如果连这一步都做不到，说明模型结构或代码实现有根本性问题。

- **过拟合小数据集**:
  ```python
  # X_small, y_small = 10个样本
  nn = NeuralNetwork([2, 2, 1])
  nn.train(X_small, y_small, epochs=5000, learning_rate=0.01)
  # 观察loss是否能降到极低
  ```

2. 确保输入数据 `X` 和标签 `y` 是正确配对的。可以对数据进行可视化，检查是否存在异常值或错误。确保数据已经正确归一化。

3. 一个太高的学习率是导致损失爆炸或不收敛的最常见原因。尝试将学习率降低一个数量级（例如从 `0.01` 到 `0.001`）。

- **学习率调参**:
  ```python
  for lr in [0.1, 0.01, 0.001]:
      nn.train(X, y, epochs=1000, learning_rate=lr)
      # 观察loss曲线
  ```

4. 确保选择了正确的损失函数，并且在反向传播时正确计算了对应的导数。

### 超参数的选择

超参数是在训练开始前设置的参数，例如学习率、层数等。

- **学习率 (Learning Rate)**: 学习率是最重要的超参数。通常从 `0.1`, `0.01`, `0.001` 等值开始尝试。可以使用**学习率衰减 (Learning Rate Decay)**，即在训练过程中逐渐降低学习率。
- **网络架构 (层数和神经元数量)**: 从一个隐藏层开始。如果网络无法很好地拟合训练数据，再逐步增加层的深度和/或宽度。通常，增加深度比增加宽度更有效。
- **批量大小 (Batch Size)**: 以前通常选择2的幂，如 32, 64, 128，但现在的说法是，随便选什么数都行。比如，你可以选 520 训练一个神经网络，送给你的对象（笑
    - **小批量**: 训练速度快，引入的噪声可能有助于泛化。
    - **大批量**: 梯度估计更准确，但可能陷入局部最小值，且需要更多内存。
- **优化器 (Optimizer)**: 我们只讨论了基本的梯度下降。现代优化器如 **Adam**, **RMSprop** 通常能提供更快的收敛速度和更好的性能，它们会自动调整学习率。在平常实践中，Adam 是一个非常好的默认选择。

## 优化 XOR 网络

相信看完上面的内容后，我们对神经网络有了更多的了解。现在对 XOR 网络的代码添加以下优化：

- Neuron 支持 tanh、sigmoid、relu 三种激活函数。
- 隐藏层用 tanh，输出层用 sigmoid。
- 权重初始化方式根据激活函数自动选择（tanh/sigmoid 用 Xavier，relu 用 He）。
- 训练和预测流程自动适配。

下面是优化后的 XOR 神经网络训练完整代码：

```python
import numpy as np


class Neuron:
    """
    单个神经元。支持多种激活函数。
    """

    def __init__(self, num_inputs, activation="relu"):
        self.activation_name = activation
        # 根据激活函数选择初始化方式
        if activation == "relu":
            self.weights = np.random.randn(num_inputs, 1) * np.sqrt(2.0 / num_inputs)
        elif activation in ("tanh", "sigmoid"):
            self.weights = np.random.randn(num_inputs, 1) * np.sqrt(1.0 / num_inputs)
        else:
            raise ValueError(f"不支持的激活函数: {activation}")
        self.bias = np.zeros((1, 1))
        self.last_input = None
        self.last_z = None
        self.last_a = None

    def relu(self, z):
        """ReLU 激活函数"""
        return np.maximum(0, z)

    def relu_derivative(self, z):
        """ReLU 激活函数的导数"""
        return np.where(z > 0, 1, 0)

    def tanh(self, z):
        """Tanh 激活函数"""
        return np.tanh(z)

    def tanh_derivative(self, z):
        """Tanh 激活函数的导数"""
        return 1 - np.tanh(z) ** 2

    def sigmoid(self, z):
        """Sigmoid 激活函数"""
        return 1 / (1 + np.exp(-z))

    def sigmoid_derivative(self, z):
        """Sigmoid 激活函数的导数"""
        s = self.sigmoid(z)
        return s * (1 - s)

    def forward(self, activations):
        """
        执行前向传播：z = a * w + b, a_out = 激活函数(z)
        """
        self.last_input = activations
        # 计算加权和 z
        z = activations @ self.weights + self.bias
        self.last_z = z
        # 应用激活函数
        if self.activation_name == "relu":
            a = self.relu(z)
        elif self.activation_name == "tanh":
            a = self.tanh(z)
        elif self.activation_name == "sigmoid":
            a = self.sigmoid(z)
        else:
            raise ValueError(f"不支持的激活函数: {self.activation_name}")
        self.last_a = a
        return a

    def backward(self, dC_da, learning_rate):
        """
        执行反向传播，计算并应用梯度。
        dC_da: 损失函数对本神经元输出 a 的梯度 (从下一层传来)
        """
        # 1. 计算 da/dz (激活函数对z的梯度)
        if self.activation_name == "relu":
            da_dz = self.relu_derivative(self.last_z)
        elif self.activation_name == "tanh":
            da_dz = self.tanh_derivative(self.last_z)
        elif self.activation_name == "sigmoid":
            da_dz = self.sigmoid_derivative(self.last_z)
        else:
            raise ValueError(f"不支持的激活函数: {self.activation_name}")

        # 2. 计算 dC/dz (损失对z的梯度) = dC/da * da/dz
        dC_dz = dC_da * da_dz

        # 3. 计算 dC/dw (损失对权重的梯度) = dC/dz * dz/dw
        #    dz/dw = last_input, 所以 dC/dw = last_input.T * dC/dz
        dC_dw = self.last_input.T @ dC_dz

        # 4. 计算 dC/db (损失对偏置的梯度) = dC/dz * dz/db
        #    dz/db = 1, 所以 dC/db = dC/dz
        dC_db = np.sum(dC_dz, axis=0, keepdims=True)

        # 5. 计算 dC/da_prev (损失对前一层激活值的梯度)，用于传给前一层
        #    dC/da_prev = dC/dz * dz/da_prev
        #    dz/da_prev = weights, 所以 dC/da_prev = dC/dz * weights.T
        dC_da_prev = dC_dz @ self.weights.T

        # 6. 根据梯度更新权重和偏置
        self.weights -= learning_rate * dC_dw
        self.bias -= learning_rate * dC_db

        # 返回 dC/da_prev，传递给前一层继续反向传播
        return dC_da_prev


class Layer:
    """
    一层神经元。支持指定激活函数。
    """

    def __init__(self, num_neurons, num_inputs_per_neuron, activation="relu"):
        self.neurons = [
            Neuron(num_inputs_per_neuron, activation) for _ in range(num_neurons)
        ]

    def forward(self, activations):
        """对层中所有神经元执行前向传播"""
        # hstack 用于水平堆叠输出，形成一个 (batch_size, num_neurons) 的矩阵
        return np.hstack([neuron.forward(activations) for neuron in self.neurons])

    def backward(self, dC_da, learning_rate):
        """对层中所有神经元执行反向传播"""
        # dC_da 的形状是 (batch_size, num_neurons)
        # 我们需要为每个神经元传入对应的梯度 dC_da[:, [i]]
        # 然后将所有神经元返回的 dC/da_prev 相加，得到传给前一层的总梯度
        return np.sum(
            [
                neuron.backward(dC_da[:, [i]], learning_rate)
                for i, neuron in enumerate(self.neurons)
            ],
            axis=0,
        )


class NeuralNetwork:
    """
    完整的神经网络模型。
    """

    def __init__(self, layer_sizes, activations=None):
        # activations: 每层的激活函数（不含输入层），如 ["tanh", "sigmoid"]
        if activations is None:
            # 默认隐藏层tanh，输出层sigmoid
            activations = ["tanh"] * (len(layer_sizes) - 2) + ["sigmoid"]
        assert len(activations) == len(layer_sizes) - 1
        self.layers = []
        for i in range(len(layer_sizes) - 1):
            self.layers.append(
                Layer(layer_sizes[i + 1], layer_sizes[i], activations[i])
            )
        self.output_activation = activations[-1]

    def forward(self, activations):
        """对所有层执行前向传播"""
        for layer in self.layers:
            activations = layer.forward(activations)
        return activations

    def mse_loss(self, y_true, y_pred):
        """均方误差损失函数"""
        return np.mean((y_pred - y_true) ** 2)

    def derivative_mse_loss(self, y_true, y_pred):
        """均方误差损失函数的导数"""
        return 2 * (y_pred - y_true) / y_true.shape[0]

    def bce_loss(self, y_true, y_pred, eps=1e-8):
        """二元交叉熵损失函数"""
        y_pred = np.clip(y_pred, eps, 1 - eps)
        return -np.mean(y_true * np.log(y_pred) + (1 - y_true) * np.log(1 - y_pred))

    def derivative_bce_loss(self, y_true, y_pred, eps=1e-8):
        """二元交叉熵损失函数的导数"""
        y_pred = np.clip(y_pred, eps, 1 - eps)
        return (y_pred - y_true) / (y_pred * (1 - y_pred) * y_true.shape[0])

    def train(self, X, y, epochs, learning_rate, batch_size=32):
        """训练神经网络"""
        for epoch in range(epochs):
            total_loss = 0
            for i in range(0, len(X), batch_size):
                X_batch = X[i : i + batch_size]
                y_batch = y[i : i + batch_size]

                outputs = self.forward(X_batch)

                # 自动选择损失函数
                if self.output_activation == "sigmoid":
                    loss = self.bce_loss(y_batch, outputs)
                    output_gradient = self.derivative_bce_loss(y_batch, outputs)
                else:
                    loss = self.mse_loss(y_batch, outputs)
                    output_gradient = self.derivative_mse_loss(y_batch, outputs)

                total_loss += loss * len(X_batch)

                for layer in reversed(self.layers):
                    output_gradient = layer.backward(output_gradient, learning_rate)

            avg_loss = total_loss / len(X)
            if (
                (epoch + 1) % max(1, epochs // 10) == 0
                or epoch == 0
                or epoch == epochs - 1
            ):
                print(f"Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.6f}")

    def predict(self, X):
        out = self.forward(X)
        if self.output_activation == "sigmoid":
            return out
        return out


# 1. 定义 XOR 数据集
X_train = np.array([[0, 0], [0, 1], [1, 0], [1, 1]])
y_train = np.array([[0], [1], [1], [0]])

# 2. 创建神经网络实例
# 2个输入节点，一个有2个节点的隐藏层，1个输出节点，激活函数分别为tanh和sigmoid
nn = NeuralNetwork([2, 2, 1], activations=["tanh", "sigmoid"])

# 3. 训练网络
# XOR 是一个非线性问题，需要足够的迭代次数和合适的学习率
print("开始训练XOR网络...")
nn.train(X_train, y_train, epochs=200000, learning_rate=0.1, batch_size=4)
print("训练完成。")

# 4. 进行预测并展示结果
print("\n对输入进行预测:")
for x_input in X_train:
    prediction = nn.predict(x_input.reshape(1, -1))
    print(f"输入: {x_input}, 预测输出: {prediction[0][0]:.4f}")

# 开始训练XOR网络...
# Epoch 1/200000, Loss: 0.710580
# Epoch 20000/200000, Loss: 0.001633
# Epoch 40000/200000, Loss: 0.000789
# Epoch 60000/200000, Loss: 0.000520
# Epoch 80000/200000, Loss: 0.000387
# Epoch 100000/200000, Loss: 0.000308
# Epoch 120000/200000, Loss: 0.000256
# Epoch 140000/200000, Loss: 0.000219
# Epoch 160000/200000, Loss: 0.000191
# Epoch 180000/200000, Loss: 0.000170
# Epoch 200000/200000, Loss: 0.000153
# 训练完成。

# 对输入进行预测:
# 输入: [0 0], 预测输出: 0.0002
# 输入: [0 1], 预测输出: 0.9999
# 输入: [1 0], 预测输出: 0.9999
# 输入: [1 1], 预测输出: 0.0002
```

可以明显看到 XOR 网络的预测更加精准了，并且我在本地连续训练了很多次，几乎不再有训练失败的情况发生。显然，我们的神经网络实现可以说成功了。

本篇中的 Layer/Neuron 结构采用 for-loop + np.hstack，便于理解原理。实际程序中建议采用全矩阵化实现以提升效率。
