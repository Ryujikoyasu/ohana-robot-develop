# バリカン除草用マニピュレータの作り方【ソフトウェア編】

ここで紹介する機能は，  
1. バリカン制御：バリカンのスイッチを自動でON/OFFする
2. アーム制御：アームの関節角を好きな位置に移動する

の２つです．

**Ubuntu22.04で，ROS 2 Humbleを用いています.**

私の場合，任天堂の[Joy-Con](https://store-jp.nintendo.com/list/hardware-accessory/controller/HAC_A_JAVAF.html)をリモコンがわりにしています．
当然，他の物でも代替可能ですし，自然言語指示からの駆動も少しのアレンジで直ぐにできます．  
<img src="../img/joycon.png" alt="画像1" style="width: 15%;">


１のバリカン制御は，初心者でもできます．
２のアーム制御は，Moveit 2というライブラリの構造を熟知する必要があります．

また，用いる計算機のアーキテクチャによってJoy-conのパッケージが異なります．
私は，ThinkPadのラップトップ（amd64アーキテクチャ）とJetson orin nano（arm64アーキテクチャ）の２つの方法で実装しました．  
Joy-conのボタンの番号の割り当てがかなり異なることと，arm64の方ではスティックボタンが0,1の実数値しかなりません．
従って，Raspberry PiやJetsonに代表されるarm64の計算機を用いる場合，スティックボタンを割り当てるのはやめておくのが賢明でしょう．


# バリカン制御

Moveit 2でエンドエフェクタとして一括で制御する方法もありますが，私は独立させています．ON/OFF制御程度では変わらないです．

処理の流れは，Joy-conのボタンを押す /rightarrow トピックとしてpublish /rightarrow バリカンのON・OFF切り替え用のコマンド（0/1の数値）をマイコンに送信 
/rightarrow マイコン側で受け取った数値によってリレー回路を操作

となります．よって私たちが実装する必要のあるコードは以下の３つです．

1. ROS 2側：Joy-conの特定のボタンが押されたら0/1を切り替える．  
2. ROS 2側：バリカンをON/OFFするコマンドをマイコン側に送信する．  
3. マイコン側：受け取った信号を元に，ON/OFFする．

上の３つはシンプルなので，LLMが書いてくれます．

また，Joy-conを使うためのパッケージは次のコマンドで入手します．

`sudo apt install ros-humble-joy*`

Joy-conの設定ファイルを適宜変更しましょう．
`/opt/ros/humble/share/ros-humble-joy`
というディレクトリにダウンロードされています．このうち，`teleop-launch.py`と`/opt/ros/humble/share/teleop_twist_joy/config/ps3.config.yaml`
を変更します．
read-onlyファイルなので，viで編集した場合，`:w !sudo tee %`というコマンドで強制上書き保存ができます．

以上．


# アーム制御  

### Moveit 2を用いたシステムの全体像

私は [Moveit 2](https://moveit.picknik.ai/humble/index.html) を用いて制御しています．

![moveit](../img/moveit.png)

この図は，今回作るシステムの概要です．

右側のRobotのブロックは，今回の場合３つのモータを搭載したロボットアームそれ自身，つまりハードウェアです，  
左側のMoveit!のブロックは，ロボットアームの頭脳です．例えば目標位置に移動するための軌跡のプランニングをします．このブロックはパッケージのインストールによって構築できます．  
Controllerのブロックは，ハードウェアと頭脳を繋ぐものです．つまり，モータとのやり取りをするためにコマンドを用いてシリアル通信します．目標値や現在値を取得します．
もう一つ．あらかじめ決めておいた位置にアームを動かしたいときに，Moveit!にその位置を教えてやる機能も今回は必要です．これはMoveit!のブロックとして考えても良いし，外部のものと考えても良いです．


### Moveit!のブロックの手順

1. Moveit 2のインストール
2. Moveit 2でシミュレーションの環境構築：アームのSTLファイルを適切に配置して，`moveit_setup_assistant`を用いて制御に必要なファイル群を自動生成する．
3. Moveit 2からアームを制御：シリアル通信によって，シミュレーションとリアルの挙動を一致させる．

### Controllerのブロックの手順

`arm_motor_exe`という実行名にしました．
シリアル通信を開始したら，
・モータの現在の関節角度を常時読み取るコマンド：20Hzのtimer_callback関数で．
・モータに目標値を送信するコマンド：ActionServerのcallback関数で．
の２つを実装します．


### ROS 2のノードを立ち上げるためのコマンド

```
ros2 run <pkg_name> arm_motor_exe　  //  アームの現在関節角度読み取り，コマンド送信 : Controllerブロック
ros2 launch <pkg_name_of_config> demo.launch.py　// moveit2の立ち上げ：Moveit!ブロック
ros2 run <pkg_name> arm_controller　　//　決めておいた関節角度をpublish
```
