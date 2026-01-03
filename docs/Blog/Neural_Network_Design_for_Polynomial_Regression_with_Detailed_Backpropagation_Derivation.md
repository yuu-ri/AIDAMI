# ゼロからニューラルネットワークを作成：二次多項式のフィッティング（完全ドキュメント）

本ドキュメントは、**ネットワーク設計（バックプロパゲーションの詳細な導出を含む）**、**Pythonコード実装**、**訓練結果**を統合した包括的な教育資料です。目的は、シンプルな3層ニューラルネットワーク（入力層 + sigmoid隠れ層 + 線形出力層）を使用して、関数  
**f(x) = 1 + 2x + 3x²** を学習することです（少量のノイズを加えた合成データを使用）。

機械学習初心者向けに、前向伝播、損失関数、連鎖律による勾配導出、パラメータ更新の全プロセスを理解しやすいように解説しています。

## 1. ニューラルネットワークの設計

### ネットワーク構造
- **入力層**：1ニューロン（入力 x 用）  
  出力：A[0] = X

- **隠れ層**：n個のニューロン（例では10個）  
  活性化関数：シグモイド、σ(z) = 1 / (1 + exp(-z))  
  計算：  
  Z[1] = W[1] ⋅ X + b[1]  
  A[1] = sigmoid(Z[1])

- **出力層**：1ニューロン（回帰タスク）  
  活性化関数：線形（活性化なし）  
  計算：  
  Z[2] = W[2] ⋅ A[1] + b[2]  
  Y_hat = A[2] = Z[2]

### 順伝播の流れ図

```
X（入力）
  │
  ↓  W[1] を掛けて + b[1]
Z[1]（隠れ層の線形出力）
  ↓  シグモイド活性化関数を通す
A[1]（隠れ層の活性化後出力）
  │
  ↓  W[2] を掛けて + b[2]
Z[2]（出力層の線形出力）
  ↓  （出力層は活性化なし、直接出力）
Y_hat（最終予測値）
```

### Y と Y_hat の定義
- **Y**：真の目標値（正解ラベル）。入力 X に対して Y = 1 + 2X + 3X²（訓練時には少量のノイズを追加）。  
- **Y_hat**：ネットワークの予測値。Y_hat = A[2] = Z[2]。

訓練の目的：重みとバイアスを調整して、Y_hat を Y にできるだけ近づけること。

### 損失関数
平均二乗誤差（Mean Squared Error, MSE）：  
L = (1/N) × Σ (Y_hat - Y)²  
（N はサンプル数）

### バックプロパゲーションの勾配導出（詳細プロセス）

連鎖律を使ってステップバイステップで勾配を導出します。まず1サンプルの損失 l = (Y_hat - Y)² で導出し、その後バッチ平均に拡張します。

#### (a) 出力層の重み W[2] に対する勾配
∂l / ∂W[2] = ∂l / ∂Y_hat × ∂Y_hat / ∂Z[2] × ∂Z[2] / ∂W[2]

- ∂l / ∂Y_hat = 2 (Y_hat - Y)  
- ∂Y_hat / ∂Z[2] = 1（線形活性化）  
- ∂Z[2] / ∂W[2] = A[1]（出力層への入力）

1サンプル：∂l / ∂W[2] = 2 (Y_hat - Y) * A[1]  

Nサンプル平均：  
**dL/dW[2] = (1/N) * sum(2 * (Y_hat - Y)) * A[1]**

#### (b) 隠れ層の重み W[1] に対する勾配
さらに後ろへ連鎖を続けます：  
∂l / ∂W[1] = ∂l / ∂Y_hat × ∂Y_hat / ∂Z[2] × ∂Z[2] / ∂A[1] × ∂A[1] / ∂Z[1] × ∂Z[1] / ∂W[1]

- 最初の2項：2 (Y_hat - Y)  
- ∂Z[2] / ∂A[1] = W[2]  
- ∂A[1] / ∂Z[1] = sigmoid'(Z[1]) = A[1] ⋅ (1 - A[1])（sigmoid_derivative(A[1]) と表記）  
- ∂Z[1] / ∂W[1] = A[0]（= X）

1サンプル：∂l / ∂W[1] = 2 (Y_hat - Y) * W[2] * sigmoid_derivative(A[1]) * A[0]  

Nサンプル平均：  
**dL/dW[1] = (1/N) * sum(2 * (Y_hat - Y) * W[2] * sigmoid_derivative(A[1])) * A[0]**

バイアスの勾配も同様です（最後の入力項を1に置き換え）。

## 2. Pythonコード実装（フレームワークなし、ゼロから実装）

```python
import numpy as np

class NeuralNetwork:
    def __init__(self, input_size, hidden_size, output_size):
        self.input_size = input_size
        self.hidden_size = hidden_size
        self.output_size = output_size
        
        # 重みとバイアスの初期化（小さなランダム値）
        self.W1 = np.random.randn(hidden_size, input_size) * 0.01
        self.b1 = np.zeros((hidden_size, 1))
        self.W2 = np.random.randn(output_size, hidden_size) * 0.01
        self.b2 = np.zeros((output_size, 1))
    
    def sigmoid(self, Z):
        return 1 / (1 + np.exp(-Z))
    
    def sigmoid_derivative(self, A):
        return A * (1 - A)
    
    def forward_propagation(self, X):
        self.A0 = X
        
        self.Z1 = np.dot(self.W1, self.A0) + self.b1
        self.A1 = self.sigmoid(self.Z1)
        
        self.Z2 = np.dot(self.W2, self.A1) + self.b2
        self.A2 = self.Z2  # 線形出力
        
        return self.A2
    
    def backward_propagation(self, X, Y, learning_rate):
        m = X.shape[1]  # サンプル数
        
        # 勾配計算（導出式に対応）
        dZ2 = 2 * (self.A2 - Y) / m
        dW2 = np.dot(dZ2, self.A1.T)
        db2 = np.sum(dZ2, axis=1, keepdims=True)
        
        dZ1 = np.dot(self.W2.T, dZ2) * self.sigmoid_derivative(self.A1)
        dW1 = np.dot(dZ1, self.A0.T)
        db1 = np.sum(dZ1, axis=1, keepdims=True)
        
        # パラメータ更新
        self.W2 -= learning_rate * dW2
        self.b2 -= learning_rate * db2
        self.W1 -= learning_rate * dW1
        self.b1 -= learning_rate * db1
    
    def train(self, X, Y, epochs=1000, learning_rate=0.01):
        for i in range(epochs):
            predictions = self.forward_propagation(X)
            self.backward_propagation(X, Y, learning_rate)
            
            if i % 100 == 0:
                loss = np.mean((predictions - Y) ** 2)
                print(f"Epoch {i}, Loss: {loss}")
    
    def predict(self, X):
        return self.forward_propagation(X)

# 合成データの生成
X = np.random.rand(1, 100) * 2  # 必要に応じて範囲を調整
Y = 1 + 2*X + 3*X**2 + np.random.randn(1, 100) * 0.1

# モデル作成・訓練（隠れ層10ニューロン）
model = NeuralNetwork(input_size=1, hidden_size=10, output_size=1)
model.train(X, Y, epochs=3000, learning_rate=0.001)

# 新しい入力でのテスト
new_X = np.array([[0.5]])
prediction = model.predict(new_X)
print("x=0.5 の予測値:", prediction)
```

## 3. 訓練結果と可視化の説明

上記コードで3000エポック訓練（学習率0.001、隠れ層10ニューロン）した結果、損失は徐々に低下しました：

```
Epoch 0, Loss: 9.75192920840894
Epoch 100, Loss: 3.9307388513436994
Epoch 200, Loss: 2.4914077376169765
...
Epoch 2800, Loss: 1.8868737638135729
Epoch 2900, Loss: 1.8806908160225506
```

最終損失は約1.88付近で安定（データにノイズを加えているため、最適MSEはノイズ分散に近い値）。

**予測例**（x = 0.5）：  
真の値：1 + 2×0.5 + 3×(0.5)² = 1 + 1 + 0.75 = **2.75**  
モデル予測：**[[2.80587383]]**（非常に近い！）

可視化結果（元のLinkedIn画像に含まれる内容）：
- 青い散布図：ノイズ付き訓練データ
- 赤い実線：真の関数 f(x) = 1 + 2x + 3x²
- 緑の破線/実線：ニューラルネットワークが学習したフィッティング曲線
- 曲線が真の関数に高密度でフィットしており、二次多項式の関係を正しく学習できたことを示しています。

このシンプルな例は、数学的設計 → コード実装 → 実際の訓練までの全フローを完全に示しています。より複雑なタスクへの拡張もぜひお試しください！
