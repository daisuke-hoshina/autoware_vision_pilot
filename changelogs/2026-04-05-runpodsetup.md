
# Autoware Vision Pilot RunPod環境構築ログ

このドキュメントでは、RunPodのGPUインスタンス（Ubuntu 22.04 + ROS 2 Humble）上で `autoware_vision_pilot` のROS 2パッケージをセットアップしてビルドし、Foxglove Studioを用いた3Dデバッグが出来るようにするまでの手順を記録しています。

## 1. インスタンスの設定
- **OS:** Ubuntu 22.04
- **ROSのバージョン:** ROS 2 Humble
- **ネットワーク設定:** Foxglove WebSocket Bridge 用に TCP ポート `8765` を公開するように設定。

## 2. システムと ROS 2 の依存パッケージのインストール
aptパッケージリストを更新し、コンパイルに必要なツール、OpenCV、および ROS 2 用の依存パッケージをインストールしました。

```bash
apt-get update
apt-get install -y git wget curl cmake build-essential libopencv-dev
apt-get install -y ros-humble-foxglove-bridge ros-humble-cv-bridge ros-humble-image-transport ros-humble-vision-msgs ros-humble-nav2-common
```

## 3. リポジトリのクローン
ソースコードをクローンし、ROS 2 パッケージのディレクトリに移動しました。

```bash
cd /workspace
git clone https://github.com/daisuke-hoshina/autoware_vision_pilot.git
cd autoware_vision_pilot/VisionPilot/middleware_recipes/ROS2
```

## 4. ONNX Runtime のセットアップ
`models` パッケージのビルドで要求されるため、ONNX Runtime（Linux x64版）をダウンロードして展開しました。

```bash
wget https://github.com/microsoft/onnxruntime/releases/download/v1.17.1/onnxruntime-linux-x64-1.17.1.tgz
tar xzf onnxruntime-linux-x64-1.17.1.tgz
rm onnxruntime-linux-x64-1.17.1.tgz
```

## 5. CUDA と TensorRT のセットアップ（ビルドエラーへの対処）
RunPodの初期イメージ（ROS 2環境）にはCUDAツールキット（`nvcc`等）や TensorRT が含まれておらず、バックエンドモデルのビルドで `Failed to find TensorRT` エラーが発生したため、追加でインストールを行いました。

### CUDA Toolkit のインストール
```bash
apt-get install -y nvidia-cuda-toolkit
```

### NVIDIA CUDA Keyring の追加と TensorRT のインストール
`apt` 経由で TensorRT パッケージを取得できるよう、Ubuntu 22.04向けのNVIDIAリポジトリキーを設定してインストールしました。

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
dpkg -i cuda-keyring_1.1-1_all.deb
apt-get update
apt-get install -y libnvinfer-dev libnvonnxparsers-dev
```

## 6. パッケージのビルド
ONNX Runtime の参照パスを指定し、`sensors`, `models`, `visualization` モジュールをビルドしました。

```bash
cd /workspace/autoware_vision_pilot/VisionPilot/middleware_recipes/ROS2
source /opt/ros/humble/setup.bash

# CUDA 11とGCC 11の互換性対策 (gcc-10をインストール)
apt-get install -y gcc-10 g++-10

# Ubuntu 22.04でのnvlinkによるlibrt.a/libpthread.a(リンカスクリプト)の静的リンクエラー回避
echo "int dummy_rt;" > /tmp/empty_rt.c
gcc -c /tmp/empty_rt.c -o /tmp/empty_rt.o
ar rcs /tmp/librt.a /tmp/empty_rt.o
mv /usr/lib/x86_64-linux-gnu/librt.a /usr/lib/x86_64-linux-gnu/librt.a.ldscript
cp /tmp/librt.a /usr/lib/x86_64-linux-gnu/librt.a

echo "int dummy_pthread;" > /tmp/empty_pthread.c
gcc -c /tmp/empty_pthread.c -o /tmp/empty_pthread.o
ar rcs /tmp/libpthread.a /tmp/empty_pthread.o
mv /usr/lib/x86_64-linux-gnu/libpthread.a /usr/lib/x86_64-linux-gnu/libpthread.a.ldscript
cp /tmp/libpthread.a /usr/lib/x86_64-linux-gnu/libpthread.a

# ビルド実行
colcon build --symlink-install --packages-select sensors models visualization \
  --cmake-args \
  -DCMAKE_C_COMPILER=gcc-10 -DCMAKE_CXX_COMPILER=g++-10 \
  -DCMAKE_CUDA_HOST_COMPILER=g++-10 \
  -DONNXRUNTIME_ROOTDIR=/workspace/autoware_vision_pilot/VisionPilot/middleware_recipes/ROS2/onnxruntime-linux-x64-1.17.1 \
  -DOpenCV_DIR=/usr/lib/x86_64-linux-gnu/cmake/opencv4 \
  -DCMAKE_BUILD_TYPE=Release
```

## 7. モデルと推論用データの配置
実行前に、推論に用いるモデルファイル群とサンプル動画を用意する必要があります。以下の構成になるようにファイルを配置してください。

```bash
cd /workspace/autoware_vision_pilot/VisionPilot/middleware_recipes/
mkdir -p data/models
```

1. **推論モデル**: リポジトリの `Models/model_library/SceneSeg/README.md` などに記載されている Google Drive のリンクから `SceneSeg_FP32.onnx` 等をダウンロードし、 `data/models/` 内に配置します。（RunPodのファイルアップロード機能か、`gdown` などのツールを使用します）
2. **サンプル動画**: 任意のテスト用動画ファイル（例：スマートフォンで撮影した車載動画やWeb上のフリー素材）を `video.mp4` という名前で `data/` 内に配置します。

## 8. 実行方法
ビルドが完了後、ターミナルで以下のコマンドを実行するだけで完了です。
（※推論パイプラインの起動と同時に、Foxglove用の通信サーバー `foxglove_bridge` も自動的に起動するように設定されています）

```bash
cd /workspace/autoware_vision_pilot/VisionPilot/middleware_recipes/ROS2
source /opt/ros/humble/setup.bash
source install/setup.bash
# シーンセグメンテーションのパイプラインを起動する例
ros2 launch models run_pipeline.launch.py pipeline:=scene_seg video_path:="../data/video.mp4"
```

### Foxglove Studio での可視化 (外部PCから)
RunPod上のWebSocket通信を安定して受信するため、RunPodの「公開用TCPポート（Exposed TCP port）」を利用してダイレクト接続します。

1. **接続用IP・ポートの確認**:
   RunPodのDashboard（Pod画面）の「Connect」設定等に表示される `Foxglove  xx.xx.xx.xx:yyyyy:8765` の情報、もしくは直接接続（TCP）用のマッピングを確認します。
   （例として `213.173.107.42:33215:8765` のように表記されている場合、`213.173.107.42`がIPアドレス、`33215`が外部からアクセス可能なポート番号です）
2. **Foxglove Studioの設定**:
   お手元のMac/PCで Foxglove Studio (デスクトップ版、または https://studio.foxglove.dev/ ) を開きます。
3. 左メニュー等からコンセントのアイコン「Open connection」をクリックします。
   - ※Foxglove Cloudの「Add Device(Agent)」画面は使いません。あくまでローカルツールとしてのダイレクト接続を利用します。
4. リストから **`Foxglove WebSocket`** を選択してください。（ROS 2等ではありません）
5. 右側のWebSocket URL入力欄を空にして、確認したIPアドレスとポート番号を入力し「Open」を押します：
   - 例: **`ws://213.173.107.42:33215`**
   - ※ `wss://` ではなく `ws://` となる点、うしろに `8765` は付けない点に気をつけてください。
6. 接続に成功したら枠が緑色になります。Imageパネル等を追加して `/autoseg/scene_seg/viz` などのトピックをサブスクライブし、リアルタイム映像を確認してください。
