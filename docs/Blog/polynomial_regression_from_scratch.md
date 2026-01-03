### 純粋な機械学習を大学生向けにわかりやすく説明  
〜ニューラルネットワークを使わない古典的な機械学習の例〜

このコードは、ニューラルネットワークを一切使わず、とてもシンプルな古典的な機械学習（多項式回帰 + 勾配降下法）の例です。

#### 1. 何をやろうとしているか？（問題設定）

私たちは、次のような「本当の法則（真の関数）」があると仮定します：

```
y = 2 − 3x + x²
```

この法則は私たちには隠されています。代わりに、たくさんの x と、それに対応する y（少しノイズが入ったデータ）だけが与えられます。

目標は、データだけを見て、この法則にできるだけ近い数式を自分で発見することです。

#### 2. データの生成

まず、人工的に訓練用のデータを生成します。

```python
def generate_data(true_coeffs, x, noise_std=1):
    y_true = np.zeros_like(x)
    for i, coeff in enumerate(true_coeffs):
        y_true += coeff * np.power(x, i)
    y_true += np.random.normal(0, noise_std, size=len(x))
    return y_true
```

- true_coeffs = [2, -3, 1] が本当の係数（2 − 3x + x²）
- x は -100 から 100 までの 1000 点
- 最後にノイズを加えて、現実らしいデータにしています

#### 3. モデル（予測関数）の定義

学習するモデルは、次のような多項式です：

```python
def power_series_model(coeffs, x):
    y_pred = np.zeros_like(x)
    for i, coeff in enumerate(coeffs):
        y_pred += coeff * np.power(x, i)
    return y_pred
```

最初は coeffs（係数）をランダムに初期化します：

```python
def initialize_parameters(degree):
    return np.random.rand(degree + 1)  # 次数2なら3個の係数（定数項 + x + x²）
```

#### 4. 誤差（損失関数）の計算

予測がどれだけ間違っているかを測るために、平均二乗誤差（MSE）を使います：

```python
def mse_loss(y_pred, y_true):
    return np.mean((y_pred - y_true)**2)
```

#### 5. 勾配の計算（一番大事！）

「係数を少し変えると誤差がどう変わるか」を計算します。これが勾配降下法の心臓部です。

```python
def mse_gradient(coeffs, x, y_pred, y_true):
    grad_coeffs = np.zeros_like(coeffs)
    for i, coeff in enumerate(coeffs):
        grad_coeffs[i] = np.mean(2 * (y_pred - y_true) * np.power(x, i))
    return grad_coeffs
```

#### 6. 学習ループ（訓練のメイン部分）

ここで繰り返し学習を行います。

```python
def train(degree):
    learning_rate = 0.005
    num_epochs = 10000
    threshold = 1e-8
    
    # データ生成
    np.random.seed(42)
    true_coeffs = [2, -3, 1]
    x = np.linspace(-100, 100, 1000)
    y_true = generate_data(true_coeffs, x)
    
    # 特徴量の正規化（平均0、標準偏差1にする）
    x_mean = np.mean(x)
    x_std = np.std(x)
    x_normalized = (x - x_mean) / x_std
    
    # パラメータ初期化
    coeffs = initialize_parameters(degree)
    old_loss = float('inf')
    
    # 学習ループ
    for epoch in range(num_epochs):
        # 順伝播：予測
        y_pred = power_series_model(coeffs, x_normalized)
        
        # 損失計算
        loss = mse_loss(y_pred, y_true)
        
        # 逆伝播：勾配計算
        grad_coeffs = mse_gradient(coeffs, x_normalized, y_pred, y_true)
        
        # パラメータ更新
        coeffs -= learning_rate * grad_coeffs
        
        # 収束判定
        if epoch > 0 and abs(old_loss - loss) < threshold:
            print(f"Converged. Loss: {loss}")
            break
        old_loss = loss
        
        if (epoch + 1) % 1000 == 0:
            print(f'Epoch {epoch+1}/{num_epochs}, Loss: {loss}')
    
    print(f'True Coefficients: {true_coeffs}')
    print(f'x_mean, x_std: {x_mean}, {x_std}')
    print(f'Learned Coefficients: {coeffs}')
    
    return x_mean, x_std, coeffs
```

#### 7. 予測関数

学習したモデルで新しい x に対して予測します（正規化を忘れずに！）

```python
def predct(x_new, mean, std, learned_coeffs):
    x_new_normalized = (x_new - mean) / std
    y_pred_normalized = power_series_model(learned_coeffs, x_new_normalized)
    return y_pred_normalized
```

#### 8. 結果の解釈

実行すると、次のような出力が得られます：

```
True Coefficients: [2, -3, 1]
Learned Coefficients: [ 2.00969657e+00 -1.73343644e+02  3.34001689e+03]
```

係数が全然違うように見えますが、これは x を正規化（平均≈0、標準偏差≈57.8）したためです。正規化した x に対して学習された係数は大きくなりますが、予測時には正規化を戻すので、実際の予測値はほぼ正しい値になります。

例：
```python
predct([1,2,3], x_mean, x_std, learned_coeffs)
→ [0.01030055 0.01091065 2.01152686]
```

本当の値は [0, 0, 2] なので、非常に近いことがわかります。

#### まとめ

このコードは、ニューラルネットワークを使わず、**純粋な数学（多項式 + 微分 + 勾配降下法）だけで機械学習を実現**しています。  
大学生が機械学習を学ぶ最初のステップとして、これ以上わかりやすい例はありません。  
線形代数と微分の基礎があれば、すべて自分で書き直して実験できますよ！

---

### 【全コード（完全版）】

以下に、説明で使ったすべてのコードを1つの実行可能なスクリプトとしてまとめました。  
必要なライブラリは `numpy` のみです。

```python
import argparse
import numpy as np

# データ生成
def generate_data(true_coeffs, x, noise_std=1):
    y_true = np.zeros_like(x)
    for i, coeff in enumerate(true_coeffs):
        y_true += coeff * np.power(x, i)
    y_true += np.random.normal(0, noise_std, size=len(x))
    return y_true

# モデル（多項式）
def power_series_model(coeffs, x):
    y_pred = np.zeros_like(x)
    for i, coeff in enumerate(coeffs):
        y_pred += coeff * np.power(x, i)
    return y_pred

# 損失関数（平均二乗誤差）
def mse_loss(y_pred, y_true):
    return np.mean((y_pred - y_true)**2)

# 勾配計算
def mse_gradient(coeffs, x, y_pred, y_true):
    grad_coeffs = np.zeros_like(coeffs)
    for i, coeff in enumerate(coeffs):
        grad_coeffs[i] = np.mean(2 * (y_pred - y_true) * np.power(x, i))
    return grad_coeffs

# パラメータ初期化
def initialize_parameters(degree):
    return np.random.rand(degree + 1)

# 訓練関数
def train(degree):
    learning_rate = 0.005
    num_epochs = 10000
    threshold = 1e-8
    
    np.random.seed(42)
    true_coeffs = [2, -3, 1]  # y = 2 - 3x + x²
    x = np.linspace(-100, 100, 1000)
    y_true = generate_data(true_coeffs, x)
    
    # 正規化
    x_mean = np.mean(x)
    x_std = np.std(x)
    x_normalized = (x - x_mean) / x_std
    
    coeffs = initialize_parameters(degree)
    old_loss = float('inf')
    
    for epoch in range(num_epochs):
        y_pred = power_series_model(coeffs, x_normalized)
        loss = mse_loss(y_pred, y_true)
        grad_coeffs = mse_gradient(coeffs, x_normalized, y_pred, y_true)
        coeffs -= learning_rate * grad_coeffs
        
        if epoch > 0 and abs(old_loss - loss) < threshold:
            print(f"Converged. Loss: {loss}")
            break
        old_loss = loss
        
        if (epoch + 1) % 1000 == 0:
            print(f'Epoch {epoch+1}/{num_epochs}, Loss: {loss}')
    
    print(f'True Coefficients: {true_coeffs}')
    print(f'x_mean, x_std: {x_mean}, {x_std}')
    print(f'Learned Coefficients: {coeffs}')
    
    return x_mean, x_std, coeffs

# 予測関数
def predct(x_new, mean, std, learned_coeffs):
    x_new_normalized = (x_new - mean) / std
    return power_series_model(learned_coeffs, x_new_normalized)

# 係数の文字列解析（コマンドライン用）
def parse_coefficients(coefficients_str):
    return [float(c) for c in coefficients_str.split()]

# メイン処理
if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Power series machine learning example")
    subparsers = parser.add_subparsers(dest='command')

    parser_train = subparsers.add_parser('train', help='Train the model')
    parser_train.add_argument("-d", "--degree", type=int, help="Polynomial degree")
    parser_train.add_argument("-m", "--multiple", action="store_true", help="Train multiple degrees")

    parser_predict = subparsers.add_parser('pred', help='Predict using trained model')
    parser_predict.add_argument("-m", "--mean", type=float, required=True)
    parser_predict.add_argument("-s", "--std", type=float, required=True)
    parser_predict.add_argument("-x", "--x", type=float, nargs='+', required=True)
    parser_predict.add_argument('-c', '--coefficients', type=str, required=True)

    args = parser.parse_args()

    if args.command == 'train':
        degree = int(args.degree) if args.degree else 2
        if args.multiple:
            for i in range(degree + 1):
                train(i)
        else:
            train(degree)
    elif args.command == 'pred':
        coeffs = parse_coefficients(args.coefficients)
        pred = predct(np.array(args.x), args.mean, args.std, np.array(coeffs))
        print("Prediction:", pred.tolist())
    else:
        # デフォルト実行
        x_mean, x_std, coeffs = train(2)
        print("Default prediction for x=[1,2,3]:")
        print(predct(np.array([1,2,3]), x_mean, x_std, coeffs))
```

このスクリプトをそのまま実行すれば、説明と同じ結果が再現できます。  
ぜひ自分で動かして、学習率や次数を変えて実験してみてください！
