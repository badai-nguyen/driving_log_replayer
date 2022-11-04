# 点群生成の評価

Autoware の点群処理のプロセス(sensing→perception)が動作して、/perception/obstacle_segmentation/pointcloud が意図通りに出力されるかどうかを評価する。

点群が意図通りに出力されているかの判定は、t4_dataset と点群を用いて行う。以下の評価を同時に行う。

1. 事前にアノテーションしておいた車両や歩行者などが検知出来ているかの評価（detection: 検知）
2. レーンとシナリオで定義した自車両周りのポリゴンが重なるエリアに余分な点群が出ていないかの評価（non_detection: 非検知）

アノテーションツールは[Deepen](https://www.deepen.ai/)が推奨であるが、t4_dataset への変換がサポートされているツールであればよい。
変換ツールさえ作成できれば複数のアノテーションツールを利用することが可能である。

## 評価方法

obstacle_segmentation.launch.py を使用して評価する。
launch を立ち上げると以下のことが実行され、評価される。

1. launch で C++の評価ノードと、Python の評価ノード、logging_simulator.launch、ros2 bag play を立ち上げる
2. bag から出力されたセンサーデータを autoware が受け取って、/perception/obstacle_segmentation/pointcloud を出力する
3. C++の評価ノードが/perception/obstacle_segmentation/pointcloud を subscribe して、header の時刻で非検知エリアの polygon を計算する。
4. 非検知エリアのポリゴン、pointcloud を/driving_log_replayer/obstacle_segmentation/input に publish する
5. Python の評価ノードが/driving_log_replayer/obstacle_segmentation/input を subscribe して、callback で perception_eval を使って評価する。結果をファイルに記録する
6. bag の再生が終了すると自動で launch が終了して評価が終了する

## 評価結果

topic の subscribe 1 回につき、以下に記述する判定結果が出力される。

### 検知正常

シナリオで指定した UUID を持つ bounding box 内に、指定した点数以上の点群(/perception/obstacle_segmentation/pointcloud)が入っていること。
複数の UUID を指定した場合は指定した全ての bounding box に対して条件を満たしていること。
かつ、autoware が提供する診断機能で点群の出力レートがエラーではないこと。デフォルトでは 1.0Hz 以下でエラー

### 検知警告

シナリオで指定した UUID を持つ bounding box の visibility が none(occlusion 状態)であり、評価出来ない場合。

### 検知異常

検知警告でも、検知正常でもない場合

### 非検知正常

非検知エリアに点群が 1 点もないこと。

非検知エリアは評価方法のステップ 3 で C++のノードで計算される領域。

### 非検知異常

非検知エリアに点群が出ていること。

## 評価ノードが使用する Topic 名とデータ型

- subscribe

| topic 名                                        | データ型                                     |
| ----------------------------------------------- | -------------------------------------------- |
| /perception/obstacle_segmentation/pointcloud    | sensor_msgs::msg::PointCloud2                |
| /diagnostics_agg                                | diagnostic_msgs::msg::DiagnosticArray        |
| /map/vector_map                                 | autoware_auto_mapping_msgs::msg::HADMapBin   |
| /tf                                             | tf2_msgs/msg/TFMessage                       |
| /planning/scenario_planning/status/stop_reasons | tier4_planning_msgs::msg::StopReasonArray    |
| /planning/scenario_planning/trajectory          | autoware_auto_planning_msgs::msg::Trajectory |

- publish

| topic 名                                          | データ型                                                 |
| ------------------------------------------------- | -------------------------------------------------------- |
| /driving_log_replayer/obstacle_segmentation/input | driving_log_replayer_msgs::msg:ObstacleSegmentationInput |
| /driving_log_replayer/marker/detection            | visualization_msgs::msg::MarkerArray                     |
| /driving_log_replayer/marker/non_detection        | visualization_msgs::msg::MarkerArray                     |
| /driving_log_replayer/pcd/detection               | sensor_msgs::msg::PointCloud2                            |
| /driving_log_replayer/pcd/non_detection           | sensor_msgs::msg::PointCloud2                            |
| /planning/mission_planning/goal                   | geometry_msgs::msg::PoseStamped                          |

## logging_simulator.launch に渡す引数

autoware の処理を軽くするため、評価に関係のないモジュールは launch の引数に false を渡すことで無効化する。以下を設定する。

- localization: false
- control: false

## simulation

シミュレーション実行に必要な情報を述べる。

### 入力 rosbag に含まれるべき topic

t4_dataset で必要なトピックが含まれていること

## evaluation

評価に必要な情報を述べる。

### シナリオフォーマット

```yaml
Evaluation:
  UseCaseName: obstacle_segmentation
  UseCaseFormatVersion: 0.1.0
  Datasets:
    - 63800729-18d2-4383-91e9-fea7bad384f4:
        VehicleId: ps1/20210620/CAL_000015 # データセット毎にVehicleIdを指定する
        LocalMapPath: $HOME/map/obstacle_segmentation # データセット毎にLocalMapPathを指定する
  Conditions:
    ObstacleDetection:
      PassRate: 99.0 # 評価試行回数の内、どの程度(%)評価成功だったら成功とするか
    NonDetection:
      PassRate: 99.0 # 評価試行回数の内、どの程度(%)評価成功だったら成功とするか
      ProposedArea: # base_linkを中心に非検知のエリアを一筆描きのpolygonで記述する。時計周りに記述する
        polygon_2d: # xy平面でpolygonを時計回りで記述する
          - [10.0, 1.5]
          - [10.0, -1.5]
          - [0.0, -1.5]
          - [0.0, 1.5]
        z_min: 0.0 # 3Dにするときのz下限値
        z_max: 1.5 # 3Dにするときのz上限値
  SensingEvaluationConfig:
    evaluation_config_dict:
      evaluation_task: sensing # 固定値
      target_uuids: # detectionで対象とするバウンディングボックスのID
        - 1b40c0876c746f96ac679a534e1037a2
      box_scale_0m: 1.0 # バウンディングボックスを距離に応じて拡大縮小する倍率0m地点
      box_scale_100m: 1.0 # 100m地点の倍率、0から100mまで距離に応じて線形補完で倍率が決定する
      min_points_threshold: 1 # バウンディングボックスに最低何個の点が入っていればDetectionを成功とするかのしきい値
```

### 評価結果フォーマット

obstacle_segmentation では、検知(Detection)と非検知(NonDetection)の 2 つを評価している。
1 回の点群の callback で同時に評価しているが、それぞれ別にカウントしている。
Result は検知と非検知両方のパスしていれば true でそれ以外は false 失敗となる。

以下に、フォーマットを示す。
ただし、結果ファイルフォーマットで解説済みの共通部分については省略する。

```json
{
  "Frame": {
    "FrameName": "評価に使用したt4_datasetのフレーム番号",
    "FrameSkip": "objectの評価を依頼したがdatasetに75msec以内の真値がなく評価を飛ばされた回数",
    "Detection": {
      "Result": "Success or Warn or Fail",
      "Info": [
        {
          "Annotation": "アノテーションされたバンディングボックスの情報、位置姿勢、ID",
          "PointCloud": "評価した点群の情報、バウンディングボックス内の点の数と、base_linkからの最近傍の点の位置"
        }
      ]
    },
    "NonDetection": {
      "Result": "Success or Fail",
      "Info": [
        {
          "PointCloud": "非検知エリアに出ている点の数と、base_linkからの距離毎の分布"
        }
      ]
    },
    "StopReasons": "Planning moduleが出力する停止理由。参考値",
    "TopicRate": "点群の出力レートが正常かどうかを示すdiagの結果"
  }
}
```