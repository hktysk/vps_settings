
### conohaVPSにArch Linuxのサーバーを建てる
---
### サーバー構築編

<details>
<summary>参考URL</summary>
***conoha-iso***
[公式conoha-iso説明](https://www.conoha.jp/guide/clitools.php)
***arch-install***
[Arch-Wiki インストールガイド](https://wiki.archlinux.jp/index.php/%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%82%AC%E3%82%A4%E3%83%89)
***lvm***
[lvmを使用したインストール](https://qiita.com/niwatolli3/items/926563abb65e1a9a8d4b)
[公式-lvmを使用した際のinitramfsなど](https://wiki.archlinux.jp/index.php/LVM)
***GRUB***
[BIOS, UEFIのGRUBのインストール](https://qiita.com/Gen_Arch/items/da296b7cbe5d87abc5a4)
[公式-lvmを使用したgrub.cfgの追記など](https://wiki.archlinux.jp/index.php/GRUB)
[lvm構築時にgrub-mkconfig実行時エラーが出る場合](https://www.taruki.com/wp/?p=6374)
</details>

##### API conoha-isoを使用してisoファイルをVPSに挿入（ローカル環境）

1. conohaVPSの管理画面でAPIユーザーを作成
2. conoha-isoをGitHubからダウンロード
```bash
$ curl -sL https://github.com/hironobu-s/conoha-iso/releases/download/current/conoha-iso-linux.amd64.gz | zcat > conoha-iso && chmod +x ./conoha-iso
```
3. 環境変数に認証情報を代入
```
$ export OS_USERNAME=[APIユーザ名]
$ export OS_PASSWORD=[APIパスワード]
$ export OS_TENANT_NAME=[テナント名]
$ export OS_AUTH_URL=[Identity Endpoint]
$ export OS_REGION_NAME=[リージョン]
```
4. インストールするISOイメージをダウンロード
ローカルでなくサーバーがダウンロードする。ダウンロードには少し時間がかかる。
```
$ ./conoha-iso download -i http://~[ISOファイルのあるリンク]
```
5. VPSに挿入できるISOイメージの一覧を取得
```
$ ./conoha-iso list
```
6. VPSにISOイメージを挿入。
必ずVPSをシャットダウンしてから行う。
```
$ ./conoha-iso insert
```
7. conohaVPSの管理画面からVPSを起動
8. インストール等の処理が終わったらVPSをシャットダウンし、排出
```
./conoha-iso eject
```

##### (VPS環境)Arch Linuxをインストール
*-lvm構成(BIOS起動, VPSはUEFIを使用不可)*
UEFI対応しているかは、以下で確認できる。
```
# ls /sys/firmware/efi/efivars
```
1. conoha-isoでISOファイルを挿入し、VPSを起動。
2. 追加ディスク200GBがあると想定し、/dev/vdbも含めて説明。
まず初期パーティションは、以下のようになっている。
```
/dev/vda1 => boot
/dev/vda2 => filesystem
/dev/vdb => 追加ディスク
```
一度ext4などに初期化することでcfdiskで再度開いたときにディスク中身を全削除し、gpt, dosなどの形式が選択可能になるため、これらを新しい形式にフォーマットする。  (gptはUEFI, dosはBIOS)。
/dev/vda1のbootファイルは同じディスクのvda2をext4にするだけで一緒に消せるのでフォーマットしなくても構わない。
```
mkfs.ext4 /dev/vda2
mkfs.ext4 /dev/vdb
```
3. cfdiskでパーティションを作成
```
cfdisk /dev/vda
```
ディスクの中身が全て消えるがすすめるか？
みたいな確認が赤文字で下部に表示されるので、Enterで進む。
次にgptかdosかなどを選択する表示になり、BIOSの場合はdosを選択。
/dev/vdaのパーティションは以下のように区切る。(vdaの容量は50Gと仮定)。
```
/dev/vda1 * boot
/dev/vda2 linux lvm
```
/dev/vda1はブートファイルにする。cfdiskのメニューに[boot]みたいなのがあればそれを選択してもいいし、ファイルタイプを選ぶ場合は必ずブートフラグをつける必要がある。UEFIならEFIファイルシステムを選択。
/dev/vda2はlvm用のファイルにする。
続いて、/dev/vdbは追加になるのですべてlvmにして構わない。
```
/dev/vdb linux lvm
```
これでパーティションは完了したので、次はlvmの設定をする。
4. lvmの作成
```
# lvm
```
を実行し、lvmのコンソールに入る。
lvmの/devの指定は必ず右端の数字まで入れる。

  1. PV(Physical Volume)を作成
  PVは物理ボリュームの単位
  ```
  lvm > pvcreate /dev/vda2 /dev/vdb1
  ```
  2. VG(Volume Group)を作成
  main=任意のグループ名
  ```
  lvm > vgcreate main /dev/vda2 /dev/vdb1
  ```
  3. LV(Logical Volume)を作成
  容量と名称は任意。データを保存するLVとswapを基本は作成。
  これでパーティション（LV）は完成。
  イメージとしては、pvcreateでまとめるパーティションをすべて指定（まとめたいパーティションはlvm対応の形式）→vgcreateでパーティションをまとめてひとつの仮想ディスクにする→lvcreateで仮想ディスクのパーティションを切る。
  ```
  lvm > lvcreate -L 235G -n root main
  lvm > lvcreate -L 8G -n swap main
  ```
  4. lvmで作ったパーティションとブート用パーティションをフォーマット
  ```
  # mkfs.vfat -F32 /dev/vda1
  # mkfs.ext4 /dev/main/root
  # mkswap /dev/main/swap
  ```


  5. インストール前の準備

    1. マウント
    作成したディスクをマウントする。lvmで作成したディスクは、  /dev/<volume_group>/<physical_volume>
    で呼び出す。
    ```
    # mount /dev/main/root /mnt
    # mkdir /mnt/boot
    # mount /dev/vda1 /mnt/boot
    # swapon /dev/main/swap
    ```
    2. インターネットの接続確認
    ```
    # ping www.google.co.jp
    ```
    3. システムクロックの更新
    ```
    timedatectl set-ntp true
    ```

  6. インストール

    1. ベースシステム
    ```
    # pacstrap /mnt base base-devel
    ```
    2. fstabの更新
    ```
    # genfstab -U /mnt >> /mnt/etc/fstab
    ```
    3. 各種基本設定
    ```
    # arch-chroot /mnt
    # pacman -S vim
    ```
    を実行し、システムをインストールしたルートにchrootで入る。

      1. タイムゾーン
      ```
      # ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
      ```
      2. ロケール
      /etc/locale.genを編集し、en_US.UTF-8 UTF-8とja_JP.UTF-8 UTF-8をアンコメント後、以下を実行。
      ```
      # locale-gen
      # echo LANG=en_US.UTF-8 > /etc/locale.conf
      # echo KEYMAP=jp106 > /etc/vconsole.conf
      ```
      3. ホストネーム
      hostname=任意のホストネーム
      ```
      # echo myhostname > /etc/hostname
      ```
      次に/etc/hostsにも任意のホストネームを設定。アドレスは127.0.0.1か、VPSの固定IPアドレスがある場合はそちら。
      以下/etc/hostsに書く内容。
      ```
      127.0.0.1   localhost
      ::1         localhost
      127.0.1.1   myhostname.localdomain myhostname
      ```
      4. パスワードの変更
      ```
      # passwd
      ```
   4. GRUB
   いずれかを選択(UEFIかBIOS)
    lvmの場合は各種設定をする必要がある。
   -lvm BIOS
   一度chrootをexitで抜ける。
   isoのrootから、
   ```
   # mkdir /mnt/hostrun
   # mount --bind /run /mnt/hostrun
   ```
   再びchrootで/mntに入り、grubを設定
   ```
   # arch-chroot /mnt /bin/bash
   # mkdir /run/lvm
   # mount --bind /hostrun/lvm /run/lvm
   # pacman -S os-prober grub
   # grub-install --target=i386-pc --recheck /dev/vda
   # grub-mkconfig -o /boot/grub/grub.cfg
   ```
   grubをインストールしたら、/boot/grub/grub.cfgを編集。
   108行目辺りにある（なかったら探す）、menuentry '"Arch Linux" Linux'の{}の中に追記。
    main-rootは(Volume_Group)-(Physical_Volume)で任意。ルートのパーティションはどれかを指定している。
   ```
    insmod lvm
    set root=lvm/main-root
   ```
   -BIOS
   ```
   # pacman -S os-prober grub
   # grub-install --target=i386-pc --recheck /dev/vda
   # grub-mkconfig -o /boot/grub/grub.cfg
   ```
   -64bit UEFI
   ```
   # pacman -S grub dosfstools efibootmgr
   # grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub --recheck
   # mkdir /boot/EFI/boot
   # cp /boot/EFI/grub/grubx64.efi  /boot/EFI/boot/bootx64.efi
   # grub-mkconfig -o /boot/grub/grub.cfg
   ```
   5. initramfs
   initramfsはルートファイルを読み込む前に実行されるファイル。
   lvmで構築した場合は、ここに追記しなければならない場合がある。
   まずはinitramfsを設定
   ```
   mkinitcpio -p linux
   ```
   /etc/mkinitcpio.confを開き、
   ```
   HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)
   ```
   上の行を、
   ```
   HOOKS=(base systemd autodetect modconf block sd-lvm2 filesystems keyboard fsck)
   ```
   に書き換える。
   udev => systemd
   sd-lvm2を追加
   これを書かないとboot時にlvmで作成したディスクを読み込まない。
   すでに書いてあったらそのままでOK。
   最後にもう一度mkinitcpioを再生成。
   これでsd-lvm2などが読み込まれている。
   ```
   mkinitcpio -p linux
   ```


##### VPSサーバー設定
1. ufw
ufwをインストールし、ポートを閉じる。
systemctlで再起動後も自動起動。
```
# pacman -S ufw
# ufw enable
# ufw default deny
# systemctl start ufw
# systemctl enable ufw
```
2. ユーザーを追加
ユーザーを-mのホームディレクトリ付きで作成。
後でwheelグループにのみsudoの実行権限を制限するため、gpasswdでwheelグループに追加。
hkt=任意のユーザー名
```
# useradd -m hkt
# passwd hkt
# gpasswd -a hkt wheel
```
visudoを実行し、以下の行をアンコメント。
sudoの実行権限をwheelグループのユーザーのみに制限。
```
# %wheel ALL=(ALL) ALL
↓
%wheel ALL=(ALL) ALL
```
/etc/pam.d/suを編集し、以下の行をアンコメント。
suでrootに昇格できるユーザーをwheelグループのみに設定。
```
# auth required pam_wheel.so use_uid
↓
auth required pam_wheel.so use_uid
```
***再起動し、以降は一般ユーザーで進める***

3. SSH
opensshをインストールし、sshd.serviceの起動、セキュアなSSH接続を設定。

  1. opensshとopensslをインストール
  ```
  $ sudo pacman -S openssh openssl
  ```
  2. sshdを起動
  sshd.serviceを起動することで外部からSSH接続をすることができる。
  ネットワークはVPSだからか設定しなくてもdhcpcdが繋がっている(繋がっていなかったらdhcpcdで接続)。
  ufwでSSH用の22番ポートを開ける。
  ```
  $ sudo ufw allow 22
  $ sudo systemctl restart ufw
  $ sudo systemctl start sshd.service
  $ sudo systemctl enable sshd.service
  ```
  3. パスワード認証のみでSSH接続
  鍵を生成する前に一度接続する必要(鍵のコピーをする)があるため、パスワードで認証できるようにする。
  /etc/ssh/sshd_configを編集し、以下を追記またはアンコメントしてnoならばyesに変更。
  ```
  PasswordAuthentication yes
  ```
  4. ローカル環境から一般ユーザーにSSH接続
  ローカル環境にopensshをインストールし、接続。
  user=一般ユーザー名
  xxx=接続先(VPS)のIPアドレス
  ```
  $ ssh user@xxx.xxx.xxx.xxx
  ```
  ***以降はSSHで一般ユーザーに接続して進める***

  5. SSHの秘密鍵を作成
  opensshはデフォルトで~/.ssh/authorized_keysに秘密鍵を読み込みにいくよう設定されているので、ホームディレクトリにこれらを作る。
  ちなみに.sshとauthorized_keysのパーミッションが弱いとSSHが認証されない。
  ```
  $ mkdir .ssh
  $ chmod 700 .ssh
  $ cd .ssh
  ```
  秘密鍵を作成。
  ```
  $ ssh-keygen -t rsa -b 2048
  ```
  3つの問に答える。
  1つ目は任意の鍵ファイル名。
  2つ目はこの鍵ファイルにパスワードをかけるか、かけないなら何も入力しないでEnter。
  3つ目はパスワードの再確認。かけないならそのままEnter。
  実行すると、 .ssh/以下に
  ```
  [鍵ファイル名]　[鍵ファイル名].pub
  ```
  の2つのファイルが作成される。
  .pubがサーバー側の認証ファイル、拡張子なしが接続側の認証ファイル。
  .pubをauthorized_keysに名称を変更する。
  ```
  $ mv [鍵ファイル名].pub authorized_keys
  ```
  接続側の鍵を.pemファイルに変換。
  .pemファイルは秘密鍵などを保存するファイル。
  必ず.pubでなかったもう一方を変換する。
  ssh_key=任意のファイル名
  ```
  $ openssl rsa -passin pass:password -in id_rsa -outform pem > id_rsa.pem
  ```
  .ssh/以下に新しくssh_key.pemというファイルができるので、この中身を
  ```
  $ cat ssh_key.pem
  ```
  で参照し、ローカル環境に空の.pemファイルを作ってそのなかにコピペする。
  次からのSSH接続はこの秘密鍵を使用する。
  コピペしたら、[鍵ファイル名]とssh_key.pemを削除する。接続用の鍵があるともしものときに閲覧され、悪用される可能性があるため。
  ```
  $ rm [鍵ファイル名]　ssh_key.pem
  ```

  6. /etc/ssh/sshd_confでセキュアな設定
  以下の通りに記述があるかを確認し、追記またはアンコメントなど変更する。
  ```
  RSAAuthentication yes
  PubkeyAuthentication yes
  AuthorizedKeysFile .ssh/authorized_keys
  PasswordAuthentication no
  PermitRootLogin no
  ```
  対応する内容(上から順に)
    1. RSA認証の許可
    2. 公開鍵認証の許可
    3. 公開鍵のファイル場所の指定
    4. パスワードでのログインを禁止
    5. rootへのSSH接続を禁止

  7. sshd.serviceを再起動
  ```
  $ sudo systemctl restart sshd.service
  ```
  8. sshd.serviceのログを確認する方法
  ```
  $ sudo journalctl -u sshd | tail -100
  ```




### 公開サーバー環境構築編
<details>
  <summary>参考URL</summary>
    ***nginx***
    [公式nginx](https://wiki.archlinux.jp/index.php/Nginx)
    ***nginx, php-fpmの設定***
    [ArchLinuxでnginx + php (php-fpm)とか](http://opamp.hatenablog.jp/entry/2013/08/11/011642)
    [CentOS7 + Nginx + PHP-FPM でPHPを実行する環境を整える](https://qiita.com/noraworld/items/fd491a77af9d4e1ccffa)
    [Cento7にphp-fpmをインストールし、nginxと連携する](https://qiita.com/inakadegaebal/items/d59fa99d2ee66a4ffe98)
    ***php.iniのセキュアな設定***
    [セキュリティーを考慮したphp.iniの構成](http://wepicks.net/phpsecurity-directive/)
    [安全なphp.ini設定](https://php-fan.org/%E5%AE%89%E5%85%A8%E3%81%AAphp-ini%E8%A8%AD%E5%AE%9A.html)
</details>

##### 各種インストール
  ```
  $ sudo pacman -S php php-fpm nginx mariadb mysql
  ```

##### 公開用フォルダの作成
  1. /homeにWeb公開用ディレクトリを作成
    nginxはユーザー「http」としてスクリプトを実行するため、分かりやすいようhttpユーザーのホームディレクトリを作成。
    「http」ユーザーは最初から存在するユーザーで、ホームディレクトリは作られていない。
    ```
    $ sudo mkdir /home/http
    ```
  2. /home/http以下にWeb公開用ディレクトリを作成
    一目で分かるように、Web公開用ディレクトリを/home/http以下に作成する。
    nginxが出力するファイルは/home/http/www以下に設置する(PHPの使用可能なディレクトリ制限などメリットが多い)。
    ```
    $ sudo mkdir /home/http/www
    ```
  3. サービスごとに公開用ディレクトリを作成
    サービスが複数ある場合はこの/home/http/www以下でディレクトリを区切る。
    今回は「Alight -ノベル版-」の公開用ディレクトリを作成。
    ```
    $ sudo mkdir /home/http/www/alight_novel
    ```

##### nginxの設定, 起動
  nginxは/etc/nginx/nginx.confでserver{}ごとにドメインを管理できる。
  各server{}のserver_nameにドメインを記述することで、複数のドメインを簡単に運用できる。
  centOSは/etc/nginx/conf.d/default.confかもしれない。
  1. /etc/nginx/nginx.confに記述
  １つめのserver{}の中にサンプルがあるので、書き直す。
  ポートは80のままで良い。
  ここでは便宜上、「Alight -ノベル版-」の運用として、公開用ディレクトリである/home/http/www/以下のalight_novelを指定する。
  ```
     location / {
         root   /home/http/www/alight_novel;
         index  index.html index.htm index.php;
     }

     location ~ \.php$ {
         root           /home/http/www/alight_novel;
         fastcgi_pass   127.0.0.1:9000;
         fastcgi_index  index.php;
         fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
         include        fastcgi_params;
     }
   ```
  2. nginxを起動
  ```
    $ sudo systemctl start nginx
    $ sudo systemctl enable nginx
  ```

##### php-fpmの設定, 起動
  1. /etc/php/php-fpm.d/www\.confに記述
  実行するユーザーを確認、違っていたら書き直す。
   ```
   user = http
   group = http
   listen = 127.0.0.1:9000
   listen.owner = http
   listen.group = http
   ```
  2.php-fpmを起動
   ```
   $ sudo systemctl start php-fpm
   $ sudo systemctl enable php-fpm
   ```

##### ufwとconohaVPSのWebポート開放
  ufwでnginx.confで指定したポートを開放。今回は80番ポート。
  またconohaVPSの管理側でもポートを閉じているため、管理画面からWeb用ポートを開放。
  以下はufwの実行コマンド。
  ```
  $ sudo ufw allow 80
  $ sudo systemctl restart ufw

  ```

##### ファイルを作成し、ブラウザからアクセスして起動を確認
1. テスト用ファイルを作成
中身は適当にphpinfo()などとし、phpが実行できるかを確認する。
```
$ sudo touch /home/http/www/alight_novel/index.php
```
2. ユーザーとパーミッションを適切に変更
/home/http以下を今までsudoで作成したため、今はすべてのディレクトリとフルがrootをユーザーにしている。
nginxはhttpとして使用するため、以下に変更。
```
$ sudo chown -R http:http /home/http
$ sudo chmod 700 /home/http
$ sudo chmod -R 755 /home/http/www
```
3. ブラウザにVPSのIPアドレス+ポート番号を入力して確認
```
http://xxx.xxx.xxx.xxx:80
```

##### mariadbの設定, 起動
mysqlはmariadbの後継のようなものであり、mysqlをインストールしたのはmariadb自体は入っていないので、必ずmariab本体もインストールする。
恒例のエラーで、
```
mysql -u root -p
```
を実行すると、 .sockファイルがないというエラーに対応。
```
$ sudo chown -R mysql:mysql /var/lib/mysql
$ sudo rm -R /var/lib/mysql/*
$ sudo mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
```
これでインストールが開始したら成功。
起動する。
```
$ sudo systemctl start mariadb
$ sudo systemctl enable mariadb
```

##### php.iniのセキュアな設定
まずエラーログの出力用ファイルとセッションを保存するディレクトリを作成。
```
$ sudo touch /home/http/www/php_error.log
$ sudo mkdir /home/http/www/session_temp/

$ sudo chown http:http /home/http/www/php_error.log
$ sudo chown -R http:http /home/http/www/session_temp/

$ sudo chmod 740 /home/http/www/php_error.log
$ sudo chmod -R 755 /home/http/www/session_temp/
```
/etc/php/php.iniの設定を以下に書き直す、または追記する。
```
// UTF-8と日本語
default_charset = UTF-8
mbstring.language = Japanese
mbstring.internal_encoding = UTF-8
mbstring.http_input = pass
mbstring.http_output = pass
mbstring.encoding_translation = Off
mbstring.detect_order = auto
```
```
// 外部からのファイルの許可
allow_url_fopen = Off
allow_url_include = Off
```
```
// タイムゾーン
date.timezone = “Asia/Tokyo”
```
```
// HTTPヘッダーに追加されるX-Powered By :PHP/5.3.0
expose_php = Off
```
```
// アップロード機能を使ってないならoff
file_uploads = Off
```
```
// エラーをブラウザに表示しない
display_errors = Off
```
```
// すべてのタイプのエラーログを出力する
log_errors = On
error_reporting = E_ALL
error_log = “/home/http/www/php_error.log”
```
```
// PHPによるアクセスを特定のディレクトリ以下に制限
open_basedir = /home/http/www/:/home/http/php_data/
```
```
// 危険だと思う関数の実行を禁止
disable_functions = phpinfo, eval, system, exec, passthru, popen
```
```
// セッション
session.cookie_secure = On
session.cookie_httponly = On
```
```
// セッション名の変更
session.name = ALIGHTSESSID
```
```
// セッションを保存するディレクトリを指定
session.save_path = “/home/www/session_temp/”
```
