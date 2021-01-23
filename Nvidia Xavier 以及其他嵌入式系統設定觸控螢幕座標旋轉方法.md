# Nvidia Xavier 以及其他嵌入式系統設定觸控螢幕座標旋轉方法

###### tags: `Nvidia` `Xavier` `顯示`

## 1. 確認設備使用的驅動  
```shell=
xinput --list
xinput --list-props [設備ID]
```

如果查詢到的資訊如下

```shell=
DISPLAY=:0 xinput --list-props 6
Device 'WaveShare WaveShare Touchscreen':
        Device Enabled (114):   1
        Coordinate Transformation Matrix (115): 1.000000, 0.000000, 0.000000, 0.000000, 1.000000, 0.000000, 0.000000, 0.000000, 1.000000
        libinput Calibration Matrix (246):      0.000000, 1.000000, 0.000000, -1.000000, 0.000000, 1.000000, 0.000000, 0.000000, 1.000000
        libinput Calibration Matrix Default (247):      1.000000, 0.000000, 0.000000, 0.000000, 1.000000, 0.000000, 0.000000, 0.000000, 1.000000
        libinput Send Events Modes Available (248):     1, 0
        libinput Send Events Mode Enabled (249):        0, 0
        libinput Send Events Mode Enabled Default (250):        0, 0
        Device Node (251):      "/dev/input/event0"
        Device Product ID (252):        3823, 5
```

可以看到該驅動方式採用的是**libinput**

## 2. 檢查是否安裝xserver-xorg-input-libinput驅動

請查看以下目錄

```shell=
/usr/share/X11/xorg.conf.d/
```

確認目錄下是否有`40-libinput.conf`這個檔案，如果沒有，則需要安裝xserver-xorg-input-libinput(基本上Xavier系統都有安裝)

```shell=
sudo apt install xserver-xorg-input-libinput
```

安裝完成後請重啟，並且確認是否有`40-libinput.conf`

## 3. 設定參數

接著複製`40-libinput.conf`到`/etc/X11/xorg.conf.d/`目錄下，一開始`xorg.conf.d`這個目錄在`/etc/X11`可能沒有，需要自己建立

```shell=
sudo mkdir xorg.conf.d
```

複製該檔案到xorg.conf.d 目錄下

```shell=
sudo cp /usr/share/X11/xorg.conf.d/40-libinput.conf /etc/X11/xorg.conf.d/
```

接著編輯該檔案

```shell=
cd /etc/X11/xorg.conf.d/
sudo nano 40-libinput.conf
```

找到以下段落(不會完全一樣，要找touchscreen)

```shell=
Section "InputClass"
        Identifier "libinput touchscreen catchall"
        MatchIsTouchscreen "on"
        MatchDevicePath "/dev/input/event*"
        Driver "libinput"
EndSection
```

將這段程式碼加入一行設定參數

```shell=
Option "CalibrationMatrix" "0 1 0 -1 0 1 0 0 1"
```

修改後

```shell=
Section "InputClass"
        Identifier "libinput touchscreen catchall"
        Option "CalibrationMatrix" "0 1 0 -1 0 1 0 0 1"
        MatchIsTouchscreen "on"
        MatchDevicePath "/dev/input/event*"
        Driver "libinput"
EndSection
```

然後重啟生效

這樣的修改也是同樣修改為翻轉90度，如果需要修改為其他角度，請參考libinput的演算法
https://wayland.freedesktop.org/libinput/doc/latest/absolute_axes.html
