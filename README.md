# nanopi R5S をセットアップするためのメモ書き

## 概要

nanopi R5S をubuntu箱として使うための，セットアップ方法及び，知っておくべき点等のメモ．ラズパイは使ったことがあるが，nanopiはよくわからん，という人向け．(自分用)

nanpiに関する情報は以下のwikiに書かれている．

https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R5S

＊注＊

ネットワークのテスト用にnanopiを使うことが多いため，そういう用途を前提としたセットアップになっている．いわゆるデスクトップPC的な使い方は想定していないので注意．特にネットワーク周りの自動的な設定は極力排除する方針．


## SDカードのパーティション

32GのSDカードを使って初期イメージを書き込んだ状態で，以下のようにパーティションが設定される．

```shell
$ sudo parted /dev/mmcblk0 print
Model: SD DR032 (sd/mmc)
Disk /dev/mmcblk0: 30.9GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name      Flags
 1      8389kB  12.6MB  4194kB               uboot
 2      12.6MB  16.8MB  4194kB               misc
 3      16.8MB  21.0MB  4194kB               dtbo
 4      21.0MB  37.7MB  16.8MB               resource
 5      37.7MB  79.7MB  41.9MB               kernel
 6      79.7MB  113MB   33.6MB               boot
 7      113MB   147MB   33.6MB               recovery
 8      147MB   2533MB  2387MB  ext4         rootfs
 9      2533MB  30.9GB  28.4GB  ext4         userdata

```

ラズパイの場合は，大抵はbootとlinux(rootfs)の2つしかなかったが，nanopiの場合かなり細かく分けられている．また，パーティション8,9はファイルシステムになっているが，それ以外はバイナリが直接置かれている状態でファイルシステムとしてはアクセスできない．

起動後に見えるファイルシステムは，userdataがrootfsの上にoverlayとして乗せられた状態になる．
このため，userdataの中を全て消去すれば，SDカード作成直後の状態に簡単に戻るし，userdata部分だけバックアップしておけば，環境の再現も出来る．

## 一番最初にやるべき事はSDイメージの再構築

nanpi R5S は，apt-getでカーネルヘッダを入れることも出来ず，そのままではドライバのビルドの際に困ることになる．また，カーネルのconfigを確認するときにも困る．
本家のwikiには，/optにヘッダが入っている旨が書かれているが，実際にイメージをダウンロードしてSDに書いてみると入っていない．

このため，カーネルのビルドを含め，SDカードのイメージを一度手元で構築しておくと後々楽になる．

イメージ構築用のスクリプトがあるので，それを使ってSDカードを作成する．その際，クロスコンパイル環境が必要で，x86マシン(64bit linux)が必須．
恐らく，ビルド用スクリプトに多少手を入れる程度でセルフコンパイル用に修正できると思われるが，ビルド時間は非常に長いため，あまり現実的ではない．CPUパワーがある環境で実行すべき．core i5で数十分かかる．

### ビルド環境を整える

最低限，カーネルのビルドができるだけのツールがx86マシンに入っていることが必要．apt-getで入れる．

```shell
$ sudo apt-get install -y build-essential bc bison flex libelf-dev libssl-dev libncurses5-dev
```

### ビルドスクリプトの入手

スクリプトは下記リポジトリに存在する．

https://github.com/friendlyarm/sd-fuse_rk3568

ビルド方法も上記リポジトリに書かれているがあまり親切ではない．
まずは，スクリプト本体と，ファイルシステムのイメージをダウンロードする．
ちなみに，ここでビルドするターゲットは friendlycore-focal-arm64 であるとする．

```
$ git clone https://github.com/friendlyarm/sd-fuse_rk3568
$ cd sd-fuse_rk3568
$ wget http://112.124.9.243/dvdfiles/RK3568/images-for-eflasher/friendlycore-focal-arm64-images.tgz
$ tar zxvf friendlycore-focal-arm64-images.tgz
```

### クロスコンパイラの入手

ビルドスクリプトを実行すると，クロスコンパイラを入れろ，と出てくるので，入れる．一応手順が表示されるが，その内容は以下の通り．この作業は，クロスコンパイラをダウンロードして展開するだけのもので，どのディレクトリで行っても構わない．ちなみにインストール先は /opt/FriendlyARM になる．

```
$ git clone https://github.com/friendlyarm/prebuilts.git -b master --depth 1
$ cd prebuilts/gcc-x64
$ cat toolchain-6.4-aarch64.tar.gz* | sudo tar xz -C /
```

### スクリプトの実行

カレントディレクトリをsd-fuse_rk3568に移した状態で
```
$ sudo ./build-kernel.sh friendlycore-focal-arm64
```

これで，必要なファイルのダウンロードとビルドが実行される．
configを書き変えたいような場合であっても，まずは一度スクリプトを最後まで動かすことで，必要なファイル一式を揃えさせる．

### カーネルのconfigとデバイスツリーを書き変えるには

次の2つのファイルを使ってカーネルが構築されるので，configやデバイスツリーを書き変えたい場合には以下のファイルを編集する．

* sd-fuse_rk3568/out/kernel-rk3568/arch/arm64/configs/nanopi5_linux_defconfig
* sd-fuse_rk3568/out/kernel-rk3568/arch/arm64/boot/dts/rockchip/rk3568-nanopi5-rev01.dts

kernelのmenuconfigを使いたい場合は，
```
$ cd sd-fuse_rk3568/out/kernel-rk3568
$ make ARCH=arm64 menuconfig
```

として，.config を作成し，そのファイルで上記のnanopi5_linux_defconfigを上書きする．

書き変えた後は，前回同様にスクリプトを実行する．
```
$ sudo ./build-kernel.sh friendlycore-focal-arm64
```

### SDカードのイメージを作成

```
$ ./mk-sd-image.sh friendlycore-focal-arm64
```

これで img ファイルが生成されるので，SDリーダを接続してddコマンドでSDカードへ書き込む．
SDカードへイメージを書き込むと，SDカードのパーティション8,9がファイルシステムになっているので，必要に応じてマウントしてファイルを書き変える．

## ブートと更新

作成したSDカードをnanopi R5Sに挿入し，電源を入れる．
friendlycore-focal-arm64の場合，デフォルトのログイン ID/PWは pi/pi になっているので，それでログインする．ログイン後，まずは，アップデートを行う．

```shell
$ sudo apt-get update
$ sudo apt-get upgrade
```

エディタ等，必要なツールを入れる．ここはそれぞれ好みで．

```shell
$ sudo apt-get install nano
```

＊ 自分の作業にとって重要なツールは apt-get で入れるのは避けた方が良い ＊

バージョンが古かったり，予期せぬ設定があったりして，ロクなことがない．自前でビルドした方が良い．

## タイムゾーンの設定

```shell
$ sudo dpkg-reconfigure tzdata
```

## localeの設定

必要に応じて．

```shell
$ sudo dpkg-reconfigure locales
```

## LANのLEDが点灯するようにする

デフォルトの状態では，本体裏側のLEDは赤いSYSがチカチカしているだけで，LANケーブルを挿しても光らない．光るように設定する．以下の内容を /etc/rc.local へ追加する．(exit 0の前へ)

```shell
modprobe ledtrig-netdev

echo netdev > /sys/class/leds/wan_led/trigger
echo eth0 > /sys/class/leds/wan_led/device_name
echo 1 > /sys/class/leds/wan_led/link
echo 1 > /sys/class/leds/wan_led/tx

echo netdev > /sys/class/leds/lan1_led/trigger
echo eth1 > /sys/class/leds/lan1_led/device_name
echo 1 > /sys/class/leds/lan1_led/link
echo 1 > /sys/class/leds/lan1_led/tx

echo netdev > /sys/class/leds/lan2_led/trigger
echo eth2 > /sys/class/leds/lan2_led/device_name
echo 1 > /sys/class/leds/lan2_led/link
echo 1 > /sys/class/leds/lan2_led/tx
```

## 固定IPにする

/etc/network/interfaces.d にある，インターフェース名と同名のファイルを編集する．例えば eth0 を 192.168.0.123, デフォルトGWを192.168.0.1 にするのであれば，eth0というファイルを以下の内容にする．

```text
auto eth0
iface eth0 inet static
        address 192.168.0.123
        netmask 255.255.255.0
        broadcast 192.168.0.255
        gateway 192.168.0.1
```

/etc/resolv.conf を編集する．このファイルはシンボリックリンクになっているので，まずは削除してしまい，通常のファイルで置き換える．(mDNSを使いたくないので．)

```shell
$ sudo rm /etc/resolv.conf
$ sudo nano /etc/resolv.conf
```

resolv.conf に，nameserverを書く．例えば 192.168.0.1 がDNSサーバになっているなら
```text
nameserver 192.168.0.1
```

## CUIモードにする

GUIは要らないので，グラフィカルな画面が立ち上がらないようにする．

```shell
$ sudo systemctl set-default multi-user.target
```


## 余計なサービスを止める

何が動いているかは次のコマンドで確認
```shell
$ systemctl -l
```

明らかに要らないやつを止める．
```shell
$ sudo systemctl disable apt-daily-upgrade.timer
$ sudo systemctl disable apt-daily.timer
$ sudo systemctl disable motd-news.timer
$ sudo systemctl disable systemd-resolved.service
$ sudo systemctl disable lcd2usb.service
$ sudo systemctl disable avahi-daemon
$ echo "AVAHI_DAEMON_DETECT_LOCAL=0" | sudo tee "/etc/default/avahi-daemon"
```


### timesyncdを止める

chrony等，時刻同期のツールを入れる場合にはこれを止めておかないとトラブる．もちろん，時刻同期のツールを入れないのであれば残しておく．

```shell
$ sudo systemctl stop systemd-timesyncd.service
$ sudo systemctl disable systemd-timesyncd.service
```

### NetworkManagerを止める

インターフェースの操作に介入してくる，トラブルの元凶．ネットワークのテスト等をする用途であれば確実に停止しておくべき．

```shell
$ sudo systemctl stop NetworkManager.service
$ sudo systemctl disable NetworkManager.service
```

## Networkのバッファサイズの設定

大きめのウインドウを指定したい場合等があるので，大きめのメモリを割り当てて置く．

```shell
$ sudo cp /etc/sysctl.conf /etc/sysctl.conf.bak
$ sudo sh -c "cat >> /etc/sysctl.conf" << EOF
net.core.rmem_max = 33554432
net.core.wmem_max = 33554432
net.core.rmem_default = 16777216
net.ipv4.tcp_rmem = 4096 87380 33554432
net.ipv4.tcp_wmem = 4096 65536 33554432
net.ipv4.udp_mem = 65536 131072 262144
net.ipv4.udp_rmem_min = 16384
net.ipv4.udp_wmem_min = 16384
EOF
```

