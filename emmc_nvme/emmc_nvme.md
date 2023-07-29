## emmcブート，かつ，NVMe SSDをrootfs用overlayに設定する

SDカード無しで起動するようにし，かつ，大容量のSSDをrootfsのoverlayに設定することで，ストレージの容量を気にせずに使えるようにする．

ここでは，OpenWRTではなくて，ubuntu用にセットアップすることを前提としている．

### セットアップに必要なもの

* NanoPiR5S 本体
* キーボード (nanopiへ接続しておく)
* HDMIモニタ (nanopiへ接続しておく)
* NVMe SSD (nanopiに内蔵する)
* SDカード
* SDカードリーダ・ライタを接続したLinuxマシン

### SSDの搭載

利用可能なNVMe SSDは2280サイズ．
NanoPiR5Sの裏ブタを開けて，SSDを搭載する．

### eflasher用のイメージの入手と書き込み

こちらの情報を参照．
https://wiki.friendlyelec.com/wiki/index.php/NanoPi_R5S#Option_2:_Install_OS_via_TF_Card

上記のページに，ダウンロード用リンクが書かれているので，そこから辿って

```
rk3568-eflasher-friendlycore-lite-focal-5.10-arm64-YYYYMMDD.img.gz
```

をダウンロードし，gzを展開してimgファイルにし，ddコマンドでSDカードへ書き込む．

### dtboファイルの書き変え

書き込んだSDカードの，パーティション1をLinuxマシンでマウントする．lsをすると以下のような状態になっている．

```
$ ls -al
total 104
drwxr-xr-x 3 root root 32768 Jul 29 10:05 .
drwxr-xr-x 1 root root  4096 Aug 21  2022 ..
-rwxr-xr-x 1 root root  1035 Jul 20 20:33 eflasher.conf
drwxr-xr-x 2 root root 32768 Jul 20 20:32 friendlycore-focal-arm64
```

カレントディレクトリをfriendlycore-focal-arm64に移すとこうなっている．

```
$ cd friendlycore-focal-arm64
$ ls -al
total 2867424
drwxr-xr-x 2 root root      32768 Jul 20 20:32 .
drwxr-xr-x 3 root root      32768 Jul 29 10:05 ..
-rwxr-xr-x 1 root root     469440 Feb 21 15:05 MiniLoaderAll.bin
-rwxr-xr-x 1 root root    8072140 Mar 14 18:11 boot.img
-rwxr-xr-x 1 root root       1408 Jul 28 23:38 dtbo.img
-rwxr-xr-x 1 root root     299008 Apr 14 12:24 idbloader.img
-rwxr-xr-x 1 root root         65 Jul 20 20:32 info.conf
-rwxr-xr-x 1 root root   33327124 Jul 20 17:50 kernel.img
-rwxr-xr-x 1 root root      49152 Apr 13  2022 misc.img
-rwxr-xr-x 1 root root        470 Jul 20 20:32 parameter.txt
-rwxr-xr-x 1 root root    5225472 Jun 28 12:22 resource.img
-rwxr-xr-x 1 root root 2884109000 Jul 20 20:32 rootfs.img
-rwxr-xr-x 1 root root    4194304 Jun 27 19:12 uboot.img
-rwxr-xr-x 1 root root     159868 Jul 20 20:32 userdata.img
```

overlay fsの設定が，dtbo.imgファイルに書かれている．
バイナリエディタを使って，overlay用のファイルシステムの設定部分を書き変える．
0x103バイト目から data=/dev/mmcblk2p9 と書かれた部分があるので，これを data=/dev/nvme0n1p1 と書き変える．

SDカードをumountし，SDカードをLinuxマシンから取り出す．

### emmcへの書き込み

作成したSDカードをnanopiへ差し込み，nanopiの電源を入れる．
自動的にemmcへの書き込みが始まるので，しばらく待つ．
完了の表示が出たら，nanopiの電源を切り，SDカードを取り外す．

### NVMe SDDのフォーマット

nanopiの電源を入れる．しばらく待つとlogin:のプロンプトが出るので，
ユーザーpi，パスワードpiでログインする．
この段階ではまだNVMeは使われていない．
```
$ ls /dev
```
としたときに，nvme0n1 というのがあるはず．このデバイスにfdiskでパーティションを作成する．

```
$ sudo fdisk /dev/nvme0n1
```

新規パーティションを作成する．容量すべてを単一のLinuxパーティションにしておく．

パーティションの作成が完了したら，ext4でフォーマットする．

```
$ sudo mkfs.ext4 /dev/nvme0n1p1
```

これで準備完了．再起動する．

```
$ sudo reboot
```

ログイン後，dfコマンドでファイルシステムを確認し，overlayの部分にSSDが見えていればOK．下記の例は512GのSSDを搭載した状態．
(例はapt-get等を実行した後なので，Usedの部分が大きくなっている)

```
pi@NanoPi-R5S:~$ df
Filesystem     1K-blocks   Used Available Use% Mounted on
udev             1987240      0   1987240   0% /dev
tmpfs             399736    552    399184   1% /run
overlay        491134192 691844 465420636   1% /
tmpfs            2000924      0   2000924   0% /dev/shm
tmpfs               5120      4      5116   1% /run/lock
tmpfs            2000924      0   2000924   0% /sys/fs/cgroup
tmpfs             400184      0    400184   0% /run/user/1000

```
