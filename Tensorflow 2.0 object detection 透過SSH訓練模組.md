# Tensorflow 2.0 object detection 透過SSH訓練模組

###### tags: `Tensorflow` `Nvidia` `Xavier` `Docker`

## 資料準備
0. 如要使用Nvidia Server，請透過[Google表單申請](https://docs.google.com/forms/d/e/1FAIpQLSfnuFTez1JujmLShcEDch0y6Iz6GI4h6J1mAwA-pAUUMVz5vA/viewform)，申請完成後請等候管理人員回覆，會收到一組SSH使用的Port，登入帳號為root、密碼為abc123

1. 將欲訓練的圖片準備好，並且Label完畢
2. 將所有的圖片以及xml文件，分成Test、Train資料夾，Test占10%、Train佔90%。Train是神經網路實際進行訓練的資料，Test是訓練過程中，驗證訓練成果的資料(驗證是否能夠object_detection出框選的物件)
3. 下載[AutoPbtxt.zip](https://drive.google.com/file/d/1U4YUB-J77s8T7jxlKbwerFF8Rjt7Rnss/view?usp=sharing)，並解壓縮。解壓縮後將Test、Train資料夾放入img資料夾，並且執行`AutoPbtxt.py`，程式會依據Img資料夾內圖片以及Xml檔，產生`data.pbtxt`，文件內會儲存圖片Label標籤，並且依照順序排列

```shell=
python AutoPbtxt.py
```

`data.pbtxt`內容格式如下，不同模組的data.pbtxt請勿混用

```shell=
item {
	id: 1
	name: 'MED_COL'
}
item {
	id: 2
	name: 'MED_COU'
}
...
```
4. 下載[Code.zip](https://drive.google.com/file/d/1IIcx1gOE0V6k1JmCzu0d0NqKcHj4p7gx/view?usp=sharing)，並且解壓縮
4. 透過SFTP上傳資料，如果是使用Nvidia Server請上傳至/userData內，請先在/userData內新增img資料夾，並且上傳`code`、`Train`、`Test`、`data.pbtxt`

## 訓練

以下使用Nvidia Server的目錄作為範例，使用其他電腦請根據不同電腦的目錄調整步驟中的參數

1. 透過SSH連線至Server
2. 移動至userData

```shell=
cd usetData/
```

3. 將圖片以及Xml檔案整合成`.record`檔案。Test與Train需要分成兩次轉檔。

```shell=
python ./code/MakeTfrecord.py \
-x ./img/test/ \
-l ./img/data.pbtxt \
-o ./test.record


python ./code/MakeTfrecord.py \
-x ./img/train/ \
-l ./img/data.pbtxt \
-o ./train.record
```

4. 下載模組，請先至[TensorFlow 2 Detection Model](https://github.com/tensorflow/models/blob/master/research/object_detection/g3doc/tf2_detection_zoo.md)，複製要使用模組的網址後透過Wget下載，請將模組下載至/userData。下載後解壓縮檔案，檔案名稱就是模組名稱，請自行替換。

```shell=
wget http://download.tensorflow.org/models/object_detection/tf2/20200711/centernet_resnet50_v1_fpn_512x512_coco17_tpu-8.tar.gz

tar zxvf centernet_resnet50_v1_fpn_512x512_coco17_tpu-8.tar.gz
```

5. 修改Config檔案，先將Config文件複製到/userData

```shell=
cp ./centernet_resnet50_v1_fpn_512x512_coco17_tpu-8/pipeline.config .
```

num_classes:Label標籤的數量，可以從`data.pbtxt`查看一共有幾個標籤

```shell=
model {
  center_net {
    num_classes: 90
...
```
修改範例
```shell=
model {
  center_net {
    num_classes: 4
...
```

batch_size:單一一次訓練的樣本數量，須注意GPU VRam的大小。Nvidia Server的依照模組預設的跑基本上不會出錯
num_steps:訓練步數

```shell=
...
train_config: {

  batch_size: 128
  num_steps: 250000
...
```
修改範例
```shell=
...
train_config: {

  batch_size: 64
  num_steps: 100000
...
```

data_augmentation_options:圖片預處理(隨機切割、改變亮度、翻轉......)，可依訓練模組的需求使用，使用方法請參考官方文件。
如果沒有對圖片做預處理的需求，請將data_augmentation_options標籤(包含內容)從config文件刪除

```shell=
...
  data_augmentation_options {
    random_horizontal_flip {
    }
  }

  data_augmentation_options {
    random_crop_image {
      min_aspect_ratio: 0.5
      max_aspect_ratio: 1.7
      random_coef: 0.25
    }
  }


  data_augmentation_options {
    random_adjust_hue {
    }
  }

  data_augmentation_options {
    random_adjust_contrast {
    }
  }

  data_augmentation_options {
    random_adjust_saturation {
    }
  }

  data_augmentation_options {
    random_adjust_brightness {
    }
  }

  data_augmentation_options {
    random_absolute_pad_image {
       max_height_padding: 200
       max_width_padding: 200
       pad_color: [0, 0, 0]
    }
  }
...
```

fine_tune_checkpoint:模組Checkpoint的位置，需要注意有些模組是ckpt-0、有些是clpt-1，請自行到解壓縮的模組目錄中查看
fine_tune_checkpoint_type:如果訓練中碰到以下錯誤，則將參數改為fine_tune
![](https://i.imgur.com/rkOcLPo.png)

```shell=
...
  fine_tune_checkpoint_version: V2
  fine_tune_checkpoint: "PATH_TO_BE_CONFIGURED"
  fine_tune_checkpoint_type: "classification"
}
...
```
修改範例
```shell=
...
  fine_tune_checkpoint_version: V2
  fine_tune_checkpoint: "/root/userData/centernet_resnet50_v1_fpn_512x512_coco17_tpu-8/checkpoint/ckpt-0"
  fine_tune_checkpoint_type: "fine_tune"
}
...
```

train_input_reader:train record設定
eval_input_reader:test record設定
其中
label_map_path:.config文件路徑
input_path:.record文件路徑

```shell=
...
train_input_reader: {
  label_map_path: "PATH_TO_BE_CONFIGURED/label_map.txt"
  tf_record_input_reader {
    input_path: "PATH_TO_BE_CONFIGURED/train2017-?????-of-00256.tfrecord"
  }
}

eval_config: {
  metrics_set: "coco_detection_metrics"
  use_moving_averages: false
  batch_size: 1;
}

eval_input_reader: {
  label_map_path: "PATH_TO_BE_CONFIGURED/label_map.txt"
  shuffle: false
  num_epochs: 1
  tf_record_input_reader {
    input_path: "PATH_TO_BE_CONFIGURED/val2017-?????-of-00032.tfrecord"
  }
}
```
修改範例
```shell=
...
train_input_reader: {
  label_map_path: "/root/userData/img/data.pbtxt"
  tf_record_input_reader {
    input_path: "/root/userData/train.record"
  }
}

eval_config: {
  metrics_set: "coco_detection_metrics"
  use_moving_averages: false
  batch_size: 1;
}

eval_input_reader: {
  label_map_path: "/root/userData/img/data.pbtxt"
  shuffle: false
  num_epochs: 1
  tf_record_input_reader {
    input_path: "/root/userData/test.record"
  }
}
```

以上，修改完成後存檔

6. 執行訓練指令

為了能夠讓離開SSH後訓練可以持續進行，會使用screen這個程式，讓訓練以及tensorboard能夠在後台運作

```shell=
screen
```

執行後會建立一個新的Shell，這時候再輸入訓練的指令。訓練的時候請回到home目錄(輸入cd後按Enter)
為了防止程式出錯，`--pipeline_config_path`、`--model_dir`的路徑都是使用絕對路徑，如果有放在不同路徑的請自行更改

```shell=
python tensorflow/models/research/object_detection/model_main_tf2.py \
--pipeline_config_path=/root/userData/pipeline.config \
--model_dir=/root/userData/model \
--alsologtostderr
```

執行之後，如果沒有出錯，就可以退出目前的Screen，按Ctrl+A，之後按D，就會退回原本的SSH Shell

7. 訓練中監控 

一樣先建立一個新的Screen
```shell=
screen
```

```shell=
tensorboard --logdir /root/userData/model (訓練中儲存模組路徑)
```
用瀏覽器打開管理員給的另一個Port就可以看到目前的訓練狀況

8. 訓練完成後模組固化

回到訓練用的Screen，可以用`screen -ls` 環境的ID

```shell=
screen -r [id]
```

輸入固化的指令，注意事項如訓練指令
```shell=
python tensorflow/models/research/object_detection/exporter_main_v2.py \
--input_type image_tensor \
--pipeline_config_path=/root/userData/pipeline.config \
--trained_checkpoint_dir=/root/userData/model \
--output_directory=/root/userData/output/
```

最後輸出的模組會在`/root/userData/output/`