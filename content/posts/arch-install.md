---
title: "Arch Linuxのインストール (UEFI boot)"
date: 2023-11-07T15:12:00+09:00
draft: false
---

約一年振りにArch Linuxのインストールを行いました。
今後のインストールの参考のために、その流れを記録しておきました。

## インストールを行う環境
インストールの詳細に影響を及ぼす点を挙げると以下の通りです。
- ラップトップ
- JISキーボード
- LANポートなし
- 日本にてインストール

## キーレイアウトの設定
今回はJISキーボードの端末へのインストールですが、
Arch LinuxのデフォルトのキーマップはUSになっているようですので、
キーマップをJISキーボードに対応するものに設定しなければ快適な入力ができません。
利用可能なキーマップの一覧は
```shell
localectl list-keymaps
```
で表示できます。この一覧から`jp`で始まるコードを探すことで、JISキーボードのキーマップが`jp106`であることが分かります。これを用いて
```shell
loadkeys jp106
```
によってキーレイアウトを設定します。

## パーティションの設定
ブロックデバイスの一覧を
```shell
lsblk
```
で取得します。
表示されたデバイスの中から、名前や容量を判断の材料にして、Arch Linuxをインストールするデバイスを決めます。私の環境では`nvme0n1`にインストールを行うことになりました。異なる場合は、以下のコマンドや説明に含まれる`nvme0n1`を適宜置き換えてお読みください。

インストールするブロックデバイスが決まったら、パーティションの設定に移ります。
パーティションの形式にはMBRとGPTがありまして、それぞれで設定の仕方が異なります。今回はGPTです。パーティションの設定に使えるツールはいくつかありますが、今回はTUIでパーティショニングができる`cgdisk`を使います。
```shell
cgdisk /dev/nvme0n1
```
を実行するとツールが起動します。
今回は、既存のパーティションは全て削除して、一からパーティショニングを行います。
パーティションの構成には色々と流儀があるようですが、今回は

1. EFI system partition: `512 MB`
2. Linux filesystem: 残り全て

というシンプルな構成にしました。パーティションの作成が終わったら、
```shell
lsblk
```
を実行して、思ったとおりにパーティショニングできたかどうかを確認しておきます。
私の環境では
1. `/dev/nvme0n1p1`: EFI system
2. `/dev/nvme0n1p2`: Linux filesystem

となりました。

## パーティションのフォーマット
各パーティションを適切なファイルシステムでフォーマットします。

```shell
mkfs.fat -F 32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p2
```
を実行することで、EFI systemは`FAT32`ファイルシステムにて、
Linux filesystemは`ext4`ファイルシステムにて、それぞれフォーマットされます。

## パーティションのマウント
フォーマットを終えたら、それぞれのパーティションをマウントします。
Linux filesystemは`/mnt`に、EFI systemは`/mnt/boot`にマウントすることにしまして、
```shell
mount /dev/nvme0n1p2 /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/boot
```
を実行します。
マウントできているかどうかは`mount`の出力によって確認できます。

```shell
mount | grep mnt
```

が見やすいです。

## インターネットへの接続
以降の過程ではインターネットへの接続が必要になります。
今回は`iwd`という無線デーモンを使用して接続を行います。

まず
```shell
iwctl station list
```
を実行することで、無線LANデバイスの名前を確認します。
私の環境では`wlan0`でした。次に
```shell
iwctl station wlan0 get-networks
```
で使用したいSSIDを確認します。
それを`<Network Name>`と仮名します。ネットワークに接続するために
```shell
iwctl station wlan0 connect <Network Name>
```
を実行します。
対話型のプロンプトが起動しますので、
Passphraseを入力して接続します。終わりましたら
```shell
ping -c 5 google.com
```
などで、本当に接続できているか確認しておきます。

## システムクロックのNTPサーバーとの同期
インターネットに接続できた段階で
```shell
timedatectl set-ntp true
```
を実行してシステムクロックがNTPサーバーと同期されるようにします。

## ミラーリストの設定
以降ではミラーからパッケージをダウンロードしてArch Linuxをインストールしていきます。
使用されるミラーは`/etc/pacman.d/mirrorlist`に書かれているものですが、
デフォルトだと遠方のサーバーが設定されているため、変更する必要があります。

手打ちではない変更の方法はいくつかあるようですが、今回は
```shell
reflector --sort rate --country Japan --latest 10 --save etc/pacman.d/mirrorlist
```
を実行することで設定します。終わりましたら`/etc/pacman.d/mirrorlist`を確認してみますと、地理的に近そうなミラーに書き換わっていることが分かります。

## パッケージのインストール

インターネットに接続して、ミラーも適切に設定できましたので、色々なパッケージをインストールしていきます。
```shell
pacstrap /mnt base base-devel linux linux-firmware linux-headers
```
を実行することでOSをインストールします。また、後述の設定においてファイルの編集が必要になりますので、テキストエディタもインストールしておくと良いです。ここでは
```shell
pacstrap /mnt vim
```
を実行することでvimをインストールします。

次に無線関連のパッケージを入れました。
再起動後にWi-Fiに繋がらなくなると面倒ですので、汚いですし重複もあるかもしれませんが、念のためにいくつかの無線関連のパッケージを
```shell
pacstrap /mnt netctl iw iwd dhcpcd wpa_supplicant networkmanager
```
でインストールしておきます。

## fstabの生成
ブートのときのパーティションのマウントを
```shell
genfstab -U /mnt >> /mnt/etc/fstab
```
を実行することで指定します。

## インストールしたOSへの移動
次にインストールしたOSに
```shell
arch-chroot /mnt
```
を実行することで入ります。

## タイムゾーンの設定
タイムゾーンを設定します。東京で良ければ
```shell
ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
```
を実行します。

### ハードウェアクロックの現在時刻への反映
```shell
hwclock --systohc
```
を実行することで、ハードウェアクロックを現在時刻とします。

## ロケールの設定
`/etc/locale.gen`を編集して
```
ja_JP.UTF-8 UTF-8
en_US.UTF-8 UTF-8
```
をコメントアウトすることで、ロケールをja_JPとen_USに設定します。

## キーマップの設定
`/etc/vconsole.conf`を作成して
```
KEYMAP=jp106
```
と書き込むことでキーマップを設定します。

## ホスト名の設定
ホスト名を設定します。ここでは`arch`と仮名します。

`/etc/hostname`を作成して
```
arch
```
と書き込みます。

## ブートローダーのインストール
ブートローダーには色々な種類がありますが、今回はGRUBを使うことにします。
まず
```shell
pacman -S grub
```
を実行することでGRUBをインストールします。次に
```shell
pacman -S efibootmgr
```
を実行することでツールをインストールしてから、
```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
grub-mkconfig -o /boot/grub/grub.cfg
```
を実行することでEFIシステムにGRUBをインストールします。

## NetworkManagerの有効化
OSの再起動後はNetworkManagerを用いてWi-Fiの接続を行うことにします。
そのためにここで
```shell
systemctl enable NetworkManager
```
を実行することで有効化しておきます。

## rootのパスワードの設定
rootのパスワードを設定します。
```shell
passwd
```
を実行することでrootのパスワードを設定します。

## ユーザーの作成とパスワードの設定
そしてユーザーを作成します。今回は`<User Name>`と仮名します。
ログインシェルにはzshを使いたいので、
```
pacman -S zsh
```
でzshをインストールしてから、
```shell
useradd -m -g users -G wheel -s /bin/zsh <User Name>
```
を実行することでユーザーを作成します。そして
```shell
passwd <User Name>
```
を実行することで、作成したユーザーのパスワードを設定します。

## ユーザーのsudoの使用
ユーザーでもroot権限が一時的に欲しい機会は多いと思います。そのために`sudo`を
```shell
pacman -S sudo
```
を実行してインストールした後、`/etc/sudoers`を編集して

```shell
%wheel ALL=(ALL:ALL) ALL
```
をアンコメントします。

ここで一旦
```shell
exit
reboot
```
で再起動を実行します。

## ネットワークへの接続
再起動ができましたら、ユーザーでログインします。
そして
```shell
nmtui
```
を実行することでインターネットに接続します。

## フォントのインストール
日本語の表示のためにもフォントをインストールします。
ここではNotoフォントを入れることにしまして、
```shell
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji
```
を実行します。

## X Window Systemのインストール
GUIを使うことにします。
ディスプレイサーバーにはX Window Systemを選びます。
```shell
sudo pacman -S xorg-server
```
を実行することでインストールします。

## ディスプレイマネージャーのインストール
ディスプレイマネージャーにはLightDMを使います。

```shell
sudo pacman -S lightdm lightdm-gtk-greeter
```
を実行してインストールした後、
```shell
sudo sytemctl enable lightdm
```
で有効化します。

## ウィンドウマネージャーのインストール
ウィンドウマネージャーにはi3を使います。
```shell
sudo pacman -S i3-wm i3blocks
```
を実行してインストールします。


## dotfilesのインストール
細かい設定を`dotfiles`を用いて行います。
私のGitHubリポジトリの`cohsh/.dotfiles`を使います。
まず
```shell
sudo pacman -S git
```
を実行することで`git`をインストールした後、
```shell
git clone https://github.com/cohsh/.dotfiles.git
```
を実行することでダウンロードします。インストールの前に必要なパッケージをインストールします。
```shell
sudo pacman -S wezterm rofi
```
を実行してターミナルエミュレーターWezTermと、ランチャーRofiをインストールします。
終わったら
```shell
cd .dotfiles
./install.sh
```
を実行してインストールします。

ここで
```
reboot
```
を実行して再起動をすれば、`LightDM + i3`のGUI環境が使えるはずです。

# 参考文献
1. [Zenn - 私的Arch Linuxインストール講座](https://zenn.dev/ytjvdcm/articles/0efb9112468de3)

    インストールの流れの大部分はこの記事に沿わせていただきました。

2. [Zenn - Arch Linuxのインストール](https://zenn.dev/imzrust/articles/42420891968a72)

3. [AzSky - Arch Linux インストール](https://aznote.jakou.com/archlinux/index.html)

3. [Qiita - Arch Linuxの最小限インストール](https://qiita.com/j8takagi/items/235e4ae484e8c587ca92)

4. [ArchWiki](https://wiki.archlinux.jp)