# NTPサーバーの構築

## きっかけ
次のホームページをみたことです。
その1～その6まであって、読み物としてもおもしろいです。楽しく読ませていただきました。ありがとうございました。
- [GPSを使ったNTPサーバの構築 その1](https://qiita.com/R800/items/79973eb327af3c8cf7e8)
- [GPSを使ったNTPサーバの構築 その6 まとめ](https://qiita.com/R800/items/6a04548beba9668b271d)

## できたもの
RPI4+GPS+RTCで**NTPサーバー**ができました。

## さぎょう

### 部品
- Raspberry Pi 4 Model B
- [ＧＰＳ受信機　シリアル出力タイプ（先バラ）　みちびき２機（１９４／１９５）対応　１ＰＰＳ出力付　ＧＴ－５０２ＭＧＧ－Ｎ](https://akizukidenshi.com/catalog/g/gM-17980/)
- [M5Stack用HYM8563搭載 リアルタイムクロック（RTC）ユニット](https://www.switch-science.com/products/7482)

### 作業
1. PRI4にUbuntu Serverのインストール、設定
    1. Ubuntu Server 64bit をインストール
    1. インストール直後のシリアルを確認(変更前)
        ```
        ls -la /dev/serial*
        
        lrwxrwxrwx 1 root root 5 Mar 20 23:32 /dev/serial0 -> ttyS0
        lrwxrwxrwx 1 root root 7 Mar 20 23:32 /dev/serial1 -> ttyAMA0
        ```
        ```
        sudo more /boot/firmware/cmdline.txt
        
        console=serial0,115200 dwc_otg.lpm_enable=0 console=tty1 root=LABEL=writable rootfstype=ext4 rootwait fixrtc quiet splash
        ```
    1. シリアルの変更で、buletoothを無効、コンソールを無効
        ```
        sudo vi /boot/firmware/config.txt
        
        dtoverlay=disable-bt
        ```
        ```
        sudo vi /boot/firmware/cmdline.txt
        
        dwc_otg.lpm_enable=0 console=tty1 root=LABEL=writable rootfstype=ext4 rootwait fixrtc quiet splash
        ```
    1. シリアルを確認(変更後)
        ```
        ls -la /dev/serial*
        
        lrwxrwxrwx 1 root root          7 Mar 20 23:32 /dev/serial0 -> ttyAMA0
        lrwxrwxrwx 1 root root          5 Mar 20 23:32 /dev/serial1 -> ttyS0
        ```
    1. pps信号をGPIO18ピンで受信できるように変更
        ```
        sudo vi /boot/firmware/config.txt
        
        dtoverlay=pps-gpio,gpiopin=18
        ```
        ```
        sudo ls -la /dev/gps0
        
        crw------- 1 root root    246,  0 Mar 20 23:32 /dev/pps0
        ```
    1. ntpdでgpsを読み取れるように、デバイスを用意
        ```
        sudo vi /etc/udev/rules.d/10-gps.rules
        
        KERNEL=="ttyAMA0",MODE="0666",SYMLINK+="gps0"
        KERNEL=="pps0",OWNER="root",GROUP="dialout",MODE="0666",SYMLINK+="gpspps0"
        ```
    1. デバイスの確認
        ```
        sudo ls -la /dev/serial* /dev/ttyAMA0 /dev/gps0 /dev/pps0 /dev/gpspps0
        
        lrwxrwxrwx 1 root root          7 Mar 20 23:32 /dev/gps0 -> ttyAMA0
        lrwxrwxrwx 1 root root          4 Mar 20 23:32 /dev/gpspps0 -> pps0
        crw-rw-rw- 1 root dialout 246,  0 Mar 20 23:32 /dev/pps0
        lrwxrwxrwx 1 root root          7 Mar 20 23:32 /dev/serial0 -> ttyAMA0
        lrwxrwxrwx 1 root root          5 Mar 20 23:32 /dev/serial1 -> ttyS0
        crw-rw-rw- 1 root dialout 204, 64 Mar 20 23:32 /dev/ttyAMA0
        ```

1. GPS受信機の接続、設定
    1. RPI4へGPS受信機を接続
        | RPI4 | GPS受信機 |
        | -- | -- |
        | 5V : PIN4 | VCC |
        | GND : PIN6 | GND |
        | TXD0 : PIN8 | RXD |
        | RXD0 : PIN10 | TXD |
        | GPIO18 : PIN12 | PPS |
    1. gpsデータの受信状況を確認(約1秒ごとにデータが受信できていれば良い)
        ```
        sudo cat /dev/ttyAMA0
        
        $GLGSV,3,3,10,83,11,293,,88,11,088,*67
        $GAGSV,2,1,08,15,62,026,24,27,44,228,34,13,44,287,,21,34,299,*62
        ```
    1. pps信号の受信状況を確認
        ```
        sudo apt install pps-tools
        ```
        ```
        sudo ppstest /dev/pps0
        
        trying PPS source "/dev/pps0"
        found PPS source "/dev/pps0"
        ok, found 1 source(s), now start fetching data...
        source 0 - assert 1679322894.617074309, sequence: 162 - clear  0.000000000, sequence: 0
        source 0 - assert 1679322895.617085605, sequence: 163 - clear  0.000000000, sequence: 0
        source 0 - assert 1679322896.617095808, sequence: 164 - clear  0.000000000, sequence: 0
        ```
    1. gps受信機が送信するデータを限定
        ```
        $PMTK353,1,1,1,0,0
        $PMTK351,1
        $PMTK314,0,1,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0
        ```

1. NTPサーバーのインストール、設定
    1. ntpの設定
        ```
        sudo apt install ntp
        ```
        ```
        sudo vi /etc/ntp.conf
        
        #pool 0.ubuntu.pool.ntp.org iburst
        #pool 1.ubuntu.pool.ntp.org iburst
        #pool 2.ubuntu.pool.ntp.org iburst
        #pool 3.ubuntu.pool.ntp.org iburst
        #pool ntp.ubuntu.com

        server 127.127.20.0 mode 16 minpoll 4 maxpoll 4 iburst prefer
        fudge 127.127.20.0 refid GPS
        fudge 127.127.20.0 flag1 1
        server ntp2.v6.mfeed.ad.jp minpoll 6 maxpoll 10 noselect
        ```
        ```
        sudo usermod -aG dialout ntp
        ```
        ```
        sudo apt install apparmor-utils
        sudo ln -s /etc/apparmor.d/usr.sbin.ntpd /etc/apparmor.d/disable
        sudo apparmor_parser -R /etc/apparmor.d/usr.sbin.ntpd
        ```

1. リアルタイムクロック(RTC)の接続、設定
    1. リアルタイムクロック(RTC)を接続
        | RPI4 | RTCモジュール |
        | -- | -- |
        | SDA : PIN3 | SDA |
        | SCL : PIN5 | SCL |
        | 5V : PIN2 | VCC |
        | GND : PIN9 | GND |
    1. I2Cの設定
        ```
        sudo apt install i2c-tools
        
        sudo i2cdetect -y 1
        
             0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
        00:                         -- -- -- -- -- -- -- --
        10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        50: -- 51 -- -- -- -- -- -- -- -- -- -- -- -- -- --
        60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        70: -- -- -- -- -- -- -- --
        
        sudo vi /etc/modules
        
        i2c-bcm2708
        i2c-dev
        ```
        ```
        sudo vi /boot/firmware/config.txt
       
        dtoverlay=i2c-rtc,pcf8563
        ```
        ```
        sudo i2cdetect -y 1
       
             0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
        00:                         -- -- -- -- -- -- -- --
        10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        50: -- UU -- -- -- -- -- -- -- -- -- -- -- -- -- --
        60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
        70: -- -- -- -- -- -- -- --
        ```

1. 状態観察
    1. 状態確認(6時間以上起動後)
        ```
        sudo ntpq -p
        
        remote           refid      st t when poll reach   delay   offset  jitter
        =============================================================================
        oGPS_NMEA(0)     .GPS.            0 l   10   16  377    0.000   +0001   0.000
        ntp2.v6.mfeed.a  133.243.236.18   2 u    9   64  377   11.790   +1345   7.755
        ```
        ```
        ntpq -c associations
        
        ind assid status  conf reach auth condition  last_event cnt
        ===========================================================
          1 52717  971a   yes   yes  none  pps.peer    sys_peer  1
          2 52718  9014   yes   yes  none    reject   reachable  1
        ```
        ```
        timedatectl status
        
                       Local time: Mon 2023-08-14 11:27:28 JST
                   Universal time: Mon 2023-08-14 02:27:28 UTC
                         RTC time: Mon 2023-08-14 02:27:28
                        Time zone: Asia/Tokyo (JST, +0900)
        System clock synchronized: yes
                      NTP service: active
                  RTC in local TZ: no
        ```

1. まとめ
    - gps受信機は、南側の窓に設置。南側はそこそこ見渡せる環境。窓に設置することで、受信状況が劇的に良くなったので、設置場所は重要！
    - gps受信機が送信するデータを限定する前は安定しなかった。限定したらデータが減るので、デフォルトの9600bpsでもいけた。

## つぎにめざすこと
RPI4+GPS+RTCを簡単に接続できるBoardを作りたいと思う。
