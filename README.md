*This project has been created as part of the 42 curriculum by hsachie.*

# Born2beRoot

## Description（概要）

仮想マシンの中に、小さいサーバーを1台作る課題です。
GUI は使わず、ターミナルだけで設定しました。

- Debian をインストールした
- ディスクを暗号化して、LVM で分けた
- `hsachie` を `sudo` と `user42` のグループに入れた
- SSH をポート 4242 にして、root ではログインできないようにした
- ファイアウォール（UFW）で、4242 番だけ開けた
- パスワードのルールを厳しくした
- sudo のルールを厳しくして、使ったコマンドを全部記録するようにした
- サーバーの情報を10分ごとに表示するスクリプト（monitoring.sh）を作った

## Project description（説明）

### 比較

#### Debian と Rocky

| | Debian | Rocky |
| --- | --- | --- |
| 作っているところ | コミュニティ | Red Hat Enterprise Linux と互換。企業向け |
| ソフトの入れ方 | `apt` | `dnf` |
| セキュリティの仕組み | AppArmor | SELinux |
| ファイアウォール | UFW を自分で入れる | firewalld |
| 難しさ | かんたん | むずかしい |

#### AppArmor と SELinux

目的は同じ（プログラムがさわれる範囲を制限する）で、ルールの決め方が違います。

| | AppArmor | SELinux |
| --- | --- | --- |
| 決め方 | ファイルの**場所（パス）**でルールを決める | ファイルやプログラムに付けた**ラベル**でルールを決める |
| 設定 | かんたん | むずかしいが細かくできる |
| 使っている OS | Debian、Ubuntu | Rocky、Fedora |

#### UFW と firewalld

| | UFW | firewalld |
| --- | --- | --- |
| 使い方 | `ufw allow 4242` のようにかんたん | **ゾーン**ごとにまとめて管理する |
| 向いている | シンプルな設定 | 複雑な設定 |
| 使っている OS | Debian、Ubuntu | Rocky、Fedora |

どちらも、Linux のファイアウォール（netfilter）をかんたんに使うための道具です。

#### VirtualBox と UTM

| | VirtualBox | UTM |
| --- | --- | --- |
| 使える PC | Windows、Linux、Mac | Mac だけ |
| Apple Silicon（M1 など） | 対応が限られている | 対応している |
| ディスクのファイル | `.vdi` | `.qcow2` |

学校の PC が Linux（x86_64）なので、VirtualBox を使いました。

---

### 仮想マシン

- VirtualBox が、PC の CPU・メモリ・ディスクの一部を借りて、PC の中にもう1台パソコンを作る
- 本物の PC に影響しないので、別の OS を試したり、サーバーを作ったりできる
- 壊してもすぐ作り直せるし、スナップショットで前の状態に戻せる
- 1台の PC で何台も動かせる

---

### OS は Debian にした

**Debian 13（trixie）** にしました。課題で初心者には Debian が推奨されていたのと、情報が多く設定がしやすいからです。

| | よいところ | よくないところ |
| --- | --- | --- |
| **Debian** | 設定が簡単。情報が多い。安定している | ソフトのバージョンが少し古い |
| **Rocky** | 企業でよく使われている（Red Hat と同じ仕組み）。サポートが長い | 設定が難しい（SELinux など） |

---

### apt と aptitude

どちらもソフトを入れる道具です。

| | apt | aptitude |
| --- | --- | --- |
| 使い方 | コマンドだけ | コマンドでも、メニュー画面でも使える |
| 特徴 | シンプル | 高機能。ソフト同士がぶつかったとき、解決方法を提案してくれる |
| 最初から入っているか | 入っている | 入っていない |

この課題では、最初から入っていてシンプルな `apt` を使いました。

```
sudo apt update             # ソフトの一覧を新しくする
sudo apt install <名前>     # ソフトを入れる
dpkg -l <名前> | cat        # 入っているか確認（ii = インストール済み）
```

---

### GUI が入っていないこと

```
dpkg -l | grep -E "xserver-xorg|xwayland|gdm3|lightdm|sddm"   # 何も出ない
ls /usr/share/xsessions /usr/share/wayland-sessions          # No such file or directory
```

---

### UFW と SSH が動いていること

```
① sudo ufw status              # UFW が動いているか：Status: active
② sudo systemctl status ssh    # SSH が動いているか：active (running)（q で戻る）
```

- `ufw status`：ファイアウォール（UFW）の状態を見る。`Status: active` なら有効で、4242 の `ALLOW IN` があれば 4242 番だけ開いている
- `systemctl status ssh`：SSH が動いているか見る。`active (running)` なら動いている。下のほうの `Server listening on 0.0.0.0 port 4242.` で、4242 番で待っていることもわかる

くわしくは「ファイアウォール（UFW）」「SSH」。

---

### OS が Debian であること

```
① cat /etc/os-release    # PRETTY_NAME="Debian GNU/Linux 13 (trixie)" と出る
② uname -a               # Debian という文字が出る（カーネルの情報）
```

- `/etc/os-release`：OS の名前とバージョンが書いてあるファイル

---

### AppArmor

プログラムごとに、さわっていいファイルなどの権限を決めておく仕組みです。もしプログラムが乗っ取られても、決められた範囲の外はさわれないので、被害を最小限にできます。

ここでいうプログラムは、`ls`、`cat`、`nano` のようなコマンドや、裏でずっと動いている `sshd`（SSH の接続を待つ）、`cron`（決めた時間にコマンドを実行する）などのことです。

| | 普通の権限 | AppArmor |
| --- | --- | --- |
| 決め方 | ユーザーごと | プログラムごと |
| root で動くプログラム | 何でもできる | プロファイルにないことは止められる |

例：`man`（マニュアルを表示するプログラム）
- マニュアルのファイル → 読んでいい
- それ以外（パスワードのファイル `/etc/shadow` など）→ 止められる

- このルールを「プロファイル」と呼び、`/etc/apparmor.d/` に入っている
- Debian に最初から入っていて、起動時から動いている。自分では設定していない

```
sudo aa-status                     # apparmor module is loaded.
systemctl is-enabled apparmor
ls /etc/apparmor.d/                # プロファイルの一覧
```

---

### ユーザーとグループ

グループは、ユーザーをまとめるチームです。グループに許可を1回出しておけば、入っている全員がその許可を使えます。

| グループ | 意味 |
| --- | --- |
| `sudo` | `sudo` コマンドが使える（`/etc/sudoers` に `%sudo ALL=(ALL:ALL) ALL` とある） |
| `user42` | 特別な権限はない。課題で決められたグループ |

| コマンド | 何をするか |
| --- | --- |
| `sudo adduser <ユーザー名>` | ユーザーを作る |
| `sudo groupadd <グループ名>` | グループを作る |
| `sudo usermod -aG <グループ名> <ユーザー名>` | ユーザーをグループに追加する（`-a` がないと、もともと入っていたグループから外れる） |
| `groups <ユーザー名>` | そのユーザーが入っているグループの一覧を見る |
| `getent group <グループ名>` | グループがあるか、誰が入っているかを見る |
| `id <ユーザー名>` | ユーザーがいるか見る（いなければ `no such user`） |

グループの変更は、ログインし直すと反映されます。

#### hsachie を sudo と user42 に入れる

```
① sudo groupadd user42                 # user42 グループを作る
② sudo usermod -aG sudo hsachie        # hsachie を sudo グループに入れる
③ sudo usermod -aG user42 hsachie      # hsachie を user42 グループに入れる
④ groups hsachie                       # 確認：sudo と user42 が出る
```

#### 新しいユーザーを作る

```
① sudo adduser <ユーザー名>          # ユーザーを作る（パスワードを2回入れる。ほかの質問は Enter）
② id <ユーザー名>                    # 確認：uid=... と出ればできている
③ sudo chage -l <ユーザー名>         # 確認：期限のルール（2 / 30 / 7）が自動で入っている
④ sudo passwd -S <ユーザー名>        # 確認：2つ目が P ならパスワード設定済み
```

- パスワードのルールに合わないパスワード（短い、数字がない、など）は弾かれる
  - ルール：10文字以上 / 大文字・小文字・数字を1つ以上 / 同じ文字は3つまで / ユーザー名を入れない / 前のパスワードと7文字以上違う（root には効かない） / root にも使う
  - ルールの表示：`grep pwquality /etc/pam.d/common-password`（くわしくは「パスワードのルール」→「強さ」）
- 期限のルールは `/etc/login.defs` から、新しいユーザーにも自動で入る

#### グループを作って入れる

```
① sudo groupadd evaluating                   # evaluating グループを作る
② getent group evaluating                    # 確認：evaluating:x:1003: と出ればグループができている
③ sudo usermod -aG evaluating <ユーザー名>   # ユーザーを evaluating に入れる
④ getent group evaluating                    # 確認：最後にユーザー名が出る（evaluating:x:1003:<ユーザー名>）
⑤ groups <ユーザー名>                        # 確認：一覧に evaluating が出る
```

#### sudo グループに入れる

```
① sudo usermod -aG sudo <ユーザー名>   # ユーザーを sudo グループに入れる
② groups <ユーザー名>                  # 確認：一覧に sudo が出る
```

#### ユーザーやグループを消す

```
① sudo deluser --remove-home <ユーザー名>   # ユーザーを消す（/home の部屋も消す）
② id <ユーザー名>                           # 確認：no such user と出れば消えている
③ sudo delgroup <グループ名>                # グループを消す
④ getent group <グループ名>                 # 確認：何も出なければ消えている
```

`①` などの番号は順番を表すためのもので、打つときは番号を付けずに、`sudo` から打ちます。

---

### パスワードのルール

#### 期限

`/etc/login.defs` を書き換えました。

```
sudo nano /etc/login.defs
```

```
PASS_MAX_DAYS 30
PASS_MIN_DAYS 2
PASS_WARN_AGE 7
```

| 設定 | 数字 | 意味 |
| --- | --- | --- |
| `PASS_MAX_DAYS`（Maximum） | 30 | パスワードは30日で期限切れ |
| `PASS_MIN_DAYS`（Minimum） | 2 | 一度変えたら、2日間は変えられない（すぐ元のパスワードに戻すのを防ぐ） |
| `PASS_WARN_AGE`（Warning） | 7 | 期限切れの7日前から警告が出る |

`/etc/login.defs` の中にも、`#` で英語の説明が書いてあります（`grep PASS /etc/login.defs`）。

`login.defs` はこれから作るユーザーにしか効かないので、もういるユーザーは `chage` で設定しました。

```
sudo chage -M 30 -m 2 -W 7 hsachie
sudo chage -M 30 -m 2 -W 7 root
```

確認：

```
sudo chage -l hsachie
sudo chage -l root
grep "^PASS_" /etc/login.defs
```

#### 強さ

```
sudo apt install libpam-pwquality
sudo nano /etc/pam.d/common-password
```

`pam_pwquality.so` の行：

```
password	requisite			pam_pwquality.so retry=3 minlen=10 ucredit=-1 lcredit=-1 dcredit=-1 maxrepeat=3 reject_username difok=7 enforce_for_root
```

| 設定 | 意味 |
| --- | --- |
| `retry=3` | 3回まで入力し直せる |
| `minlen=10` | 10文字以上 |
| `ucredit=-1` | 大文字を1つ以上 |
| `lcredit=-1` | 小文字を1つ以上 |
| `dcredit=-1` | 数字を1つ以上 |
| `maxrepeat=3` | 同じ文字は3つまでしか続けられない |
| `reject_username` | ユーザー名が入ったパスワードは使えない |
| `difok=7` | 前のパスワードと7文字以上違う |
| `enforce_for_root` | root にもルールを使う |

`difok=7` は root には効きません。root がパスワードを変えるときは前のパスワードを入れないので、比べられないからです。

確認：

```
grep pwquality /etc/pam.d/common-password
```

#### 変更したファイル

- `/etc/login.defs`（期限）
- `/etc/pam.d/common-password`（強さ）
- もういるユーザーには `chage` で期限を設定

設定のあと、ルールに合うように root と hsachie のパスワードを変更しました。

#### パスワードの変え方

```
passwd                     # 自分
sudo passwd root           # root
sudo passwd <ユーザー名>   # ほかのユーザー
```

#### このルールの利点

| ルール | 利点 |
| --- | --- |
| 10文字以上、大文字・小文字・数字を入れる | 組み合わせが多くなるので、推測されにくく、総当たりで破られにくい |
| ユーザー名を入れない、同じ文字を続けない | かんたんに予想できるパスワードを防ぐ |
| 30日で期限切れ | パスワードがもれても、ずっと使われ続けない |
| 前のパスワードと7文字以上違う | 少しだけ変えて使い回すのを防ぐ |

#### 実装の利点と欠点

期限は `/etc/login.defs` と `chage`、強さは `libpam-pwquality`（`/etc/pam.d/common-password`）で実装しました。

| 利点 | 欠点 |
| --- | --- |
| パスワードを変えるときに自動でチェックするので、弱いパスワードを付けられない | 覚えにくいので、紙にメモしたり、使い回したりしがち |
| root にも同じルールを使える | 30日ごとに変えるのがめんどう。期限が切れるとログイン時に変更を求められる |
| 全員にまとめてルールを使える | チェックされるのは変えるときだけ。設定前のパスワードはチェックされないので、全員のパスワードを変え直す必要がある |
| | `login.defs` は新しいユーザーにしか効かないので、既存のユーザーには `chage` で別に設定する必要がある |

---

### ホスト名

ホスト名は仮想マシン（コンピューター）の名前で、`hsachie42`（ログイン名 + 42）です。ターミナルの `hsachie@hsachie42` の `@` の後ろがホスト名です。ユーザーの名前ではないので、ホスト名を変えても `hsachie` は変わりません。

| コマンド・ファイル | 何をするか |
| --- | --- |
| `hostname` | 今のホスト名を見る |
| `sudo hostnamectl set-hostname <新しい名前>` | ホスト名を変える（`/etc/hostname` が書き換わる） |
| `/etc/hostname` | 自分の名前が書いてあるファイル |
| `/etc/hosts` | 名前と住所（IP アドレス）の対応表。`127.0.1.1` の行が自分の名前 |

#### ホスト名を見る

```
① hostname              # hsachie42 と出る
② cat /etc/hostname     # 同じく hsachie42
③ cat /etc/hosts        # 127.0.1.1	hsachie42 の行がある
```

#### ホスト名を変える

```
① sudo hostnamectl set-hostname <新しい名前>   # ホスト名を変える
② sudo nano /etc/hosts                         # 対応表を開く
```

`/etc/hosts` の `127.0.1.1` の行だけを、新しい名前に書き換えます（`127.0.0.1 localhost` の行は変えない）。

```
127.0.0.1	localhost          ← そのまま
127.0.1.1	<新しい名前>        ← ここだけ書き換える
```

`Ctrl + O` → Enter で保存、`Ctrl + X` で閉じます。

**なぜ `/etc/hosts` も書き換えるのか**

- `/etc/hostname` は**名札**、`/etc/hosts` は**電話帳**のようなもの。`hostnamectl` が変えるのは名札だけ
- `sudo` は最初に、電話帳で自分の名前（ホスト名）を探す。電話帳に新しい名前がないと見つからず、`sudo: unable to resolve host <新しい名前>` と警告が出る
- `localhost` は、どのコンピューターでも「自分自身」を表す決まった名前なので、変えない

```
③ sudo reboot      # 再起動する（SSH は切れるので、起動後につなぎ直す）
④ hostname         # 確認：新しい名前が出る
```

- `hostnamectl` が変えるのは `/etc/hostname`（自分の名前）だけ
- `/etc/hosts` を書き換えないと、sudo を使うたびに `unable to resolve host` と警告が出る
- 再起動後、ターミナルの表示も `hsachie@<新しい名前>` に変わる

#### 元に戻す

```
① sudo hostnamectl set-hostname hsachie42   # ホスト名を hsachie42 に戻す
② sudo nano /etc/hosts                      # 127.0.1.1 の行を hsachie42 に戻す
③ sudo reboot                               # 再起動する
④ hostname                                  # 確認：hsachie42 と出る
```

`①` などの番号は順番を表すためのもので、打つときは番号を付けずに、`sudo` などから打ちます。

---

### パーティション

インストール時に「ディスク全体を使い暗号化 LVM をセットアップする」→「/home パーティションを分割」を選びました。

```
lsblk
```

```
sda                                         ← ディスク全体
├─sda1                      /boot   約1GB   ← 普通のパーティション。暗号化なし（起動用のファイル）
├─sda2                                      ← sda5 を入れるための入れ物
└─sda5                                      ← 暗号化（カギ付き）
  └─sda5_crypt              crypt           ← カギを開けた中身
    ├─hsachie42--vg-root    /       約8GB   ← LVM の部屋：Debian 本体
    ├─hsachie42--vg-swap_1  [SWAP]  約1.5GB ← LVM の部屋：メモリが足りないときの予備
    └─hsachie42--vg-home    /home   約19GB  ← LVM の部屋：ユーザーのファイル
```

| 部屋 | 何が入っているか |
| --- | --- |
| `/boot` | パソコンを起動するためのファイル |
| root（`/`） | Debian そのもの（プログラム、設定ファイル `/etc` など） |
| home（`/home`） | ユーザーのファイル（`/home/hsachie` など） |
| swap | メモリがいっぱいのときに、使っていないデータを一時的に移す場所 |

- `/boot` は起動してパスフレーズを聞くために使うので、暗号化していない。カギの中に入れると、パスフレーズを聞くためのファイルが読めなくなる
- `sda5` は暗号化されているので、パスフレーズがないと中を読めない。ディスクが盗まれても中身を守れる
- `/home` を分けておくと、ユーザーのファイルがいっぱいになってもシステムは止まらない
- TYPE が `crypt` なら暗号化、`lvm` なら LVM
- 課題の「LVM を使って、暗号化したパーティションを2つ以上作ること」を満たしている

---

### LVM

ディスクの仕切りを、あとから動かせる仕組みです。普通のパーティションは一度分けると大きさを変えにくいですが、LVM なら広げたり、ディスクを足したりできます。

押し入れでたとえると：

| もの | たとえ | 説明 |
| --- | --- | --- |
| ディスク | 大きな押し入れ | ファイルは全部ここにしまう |
| パーティション（sda1、sda5） | 釘で固定した仕切り | あとから大きさを変えにくい |
| 暗号化 | 押し入れのカギ | パスフレーズがないと中が見られない。起動するたびにカギを開けている |
| LVM（root、swap、home） | スライド式の仕切り | あとから「home をもう少し広く」のように動かせる |

**カギ付きの箱（sda5）の中を、スライド式の仕切り（LVM）で root・swap・home の3つに分けている。**

LVM は3段階になっています。

```
PV（物理ボリューム）     ← 材料になるディスク
   ↓ まとめる
VG（ボリュームグループ） ← 大きな1つの箱
   ↓ 切り出す
LV（論理ボリューム）     ← 実際に使う部屋
```

| 用語 | 意味 | 確認するコマンド |
| --- | --- | --- |
| PV（物理ボリューム） | LVM に使うディスク（`sda5_crypt`） | `sudo pvs` |
| VG（ボリュームグループ） | PV をまとめた大きな箱（`hsachie42-vg`） | `sudo vgs` |
| LV（論理ボリューム） | VG から切り出した部屋（root、swap_1、home） | `sudo lvs` |

```
lsblk                 # TYPE に crypt と lvm がある
sudo pvs
sudo vgs
sudo lvs
```

---

### sudo

- 自分のまま、**そのコマンドだけ** root の権限で実行できる
- `su -` は root そのものになる。sudo はコマンド1回分だけ
- sudo で聞かれるのは root ではなく**自分**のパスワード
- 毎回パスワードを確認するので、root でずっと作業するより安全
- 何をしたか記録が残るので、誰がいつ何をしたかあとでわかる

例：

```
① whoami              # hsachie（今の自分）
② sudo whoami         # root（sudo を付けると、そのコマンドだけ root として動く）
③ ls /root            # Permission denied（root の部屋なので見られない）
④ sudo ls /root       # 見られる
⑤ apt update          # 権限がないのでエラー
⑥ sudo apt update     # sudo を付けると実行できる
```

- `whoami`：「今の自分は誰？」を表示するコマンド（Who am I?）。結果が名前1つだけなので、sudo を付けると root になることが一目でわかる。何も変えないので何回打っても安全
- ここで打った sudo は `/var/log/sudo/sudo.log` に記録される（`sudo tail -4 /var/log/sudo/sudo.log`）

#### sudo のルール

```
sudo apt install sudo
sudo mkdir -p /var/log/sudo
sudo visudo -f /etc/sudoers.d/sudo_config
```

```
Defaults	passwd_tries=3
Defaults	badpass_message="Wrong password. Try again!"
Defaults	logfile="/var/log/sudo/sudo.log"
Defaults	log_input
Defaults	log_output
Defaults	iolog_dir="/var/log/sudo"
Defaults	requiretty
Defaults	secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
```

| 設定 | 意味 |
| --- | --- |
| `passwd_tries=3` | パスワードは3回まで |
| `badpass_message` | 間違えたときに出るメッセージ |
| `logfile` | sudo を使った記録を残すファイル |
| `log_input` / `log_output` | 打った内容と、画面に出た内容も記録する |
| `iolog_dir` | 上の記録を置く場所 |
| `requiretty` | 本物の端末からしか sudo を使えない。プログラムが勝手に sudo を使うのを防ぐ |
| `secure_path` | sudo で使えるコマンドの場所を決めておく。にせもののコマンドを実行させられるのを防ぐ |

`visudo` は、保存するときに書き間違いをチェックしてくれます（`What now?` と出たら `e` で直す。`Q` は使わない）。

確認：

```
① sudo --version                         # sudo がインストールされているか：Sudo version ... と出る
② sudo cat /etc/sudoers.d/sudo_config    # sudo のルール：上の8行が出る
```

- `sudo --version`：sudo のバージョンを表示する。入っていなければ `command not found` になる
- `dpkg -l sudo | cat` でも確認できる（一番下の行が `ii` で始まれば、インストールされている）

#### 記録の確認

`/var/log/sudo` は、sudo を使った記録を保存しているフォルダです。記録を見られたり消されたりしないように **root しか見られない**ので、`ls /var/log/sudo` では `Permission denied` になります。**必ず `sudo` を付けて**見ます。

```
① sudo ls /var/log/sudo                  # フォルダの中身：00  seq  sudo.log
② sudo cat /var/log/sudo/sudo.log        # いつ、誰が、何をしたか（全部）
③ sudo whoami                            # sudo で何か実行すると
④ sudo tail -4 /var/log/sudo/sudo.log    # 記録が増える（最後の4行だけ見る）
```

ファイルかフォルダかを見分けるときは `-l` を付けます。

```
sudo ls -l /var/log/sudo     # 一番左が d ならフォルダ、- ならファイル
```

```
drwx------ 3 root root 4096 Oct 25 14:03 00
-rw------- 1 root root    7 Oct 25 14:03 seq
-rw------- 1 root root 2048 Oct 25 14:03 sudo.log
```

`/var/log/sudo` が存在して、中に `sudo.log` というファイルがあれば、「フォルダがあって、ファイルが1つ以上ある」ことになります。

| 中身 | 何か |
| --- | --- |
| `sudo.log` | いつ、誰が、何のコマンドを sudo で実行したかの記録 |
| `00` | 打った内容と、画面に出た内容の記録が入っているフォルダ |
| `seq` | 記録の番号を数えるためのファイル |

`sudo.log` の1件分：

```
Oct 25 14:03:12 : hsachie : TTY=pts/0 ; PWD=/home/hsachie ; USER=root ;
    TSID=000012 ; COMMAND=/usr/bin/whoami
```

| 部分 | 意味 |
| --- | --- |
| `Oct 25 14:03:12` | いつ |
| `hsachie` | 誰が |
| `TTY=pts/0` | どの端末から |
| `USER=root` | 誰の権限で |
| `TSID=000012` | 入力と出力の記録の番号 |
| `COMMAND=/usr/bin/whoami` | 何のコマンドを |

入力と出力の記録は、そのときの画面を再生できます。

```
① sudo sudoreplay -l           # 今までの sudo の一覧（TSID 付き）
② sudo sudoreplay 000012       # その番号のときの画面を再生する
```

メッセージの確認：

```
① sudo -k            # パスワードの記憶をリセット
② sudo whoami        # わざと間違える → Wrong password. Try again!
```

---

### ファイアウォール（UFW）

UFW（Uncomplicated Firewall）は、サーバーの入口（ポート）を管理するファイアウォールです。必要な入口だけ開けて、ほかを全部閉じることで、攻撃されにくくなります。`ufw allow 4242` のように、かんたんに設定できます。4242 番（SSH）だけ開けました。

```
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 4242
sudo ufw enable
```

- `sudo ufw allow 4242` は `sudo ufw enable` より先にやる（忘れると SSH で入れなくなる）

確認：

```
① sudo ufw version                   # UFW がインストールされているか：ufw 0.36.2 のように出る
② sudo ufw status verbose            # 動いているか：Status: active / 4242 だけ ALLOW IN
③ sudo systemctl is-enabled ufw      # 起動時に有効になるか：enabled
```

- `dpkg -l ufw | cat` でもインストールを確認できる（一番下の行が `ii` で始まる）

#### ルールを足す・消す

```
sudo ufw allow 8080
sudo ufw status numbered
sudo ufw delete <番号>             # 1つ消すと番号がずれるので、毎回 status numbered で確認する
sudo ufw delete allow 8080         # こちらは v4 と v6 をまとめて消せる
sudo ufw status numbered           # 4242 と 4242 (v6) だけに戻っている
```

---

### SSH

SSH は、ネットワークを通して、離れたコンピューターにログインして操作する仕組みです。通信が暗号化されるので、パスワードや打った内容を途中で盗み見られません。

`/etc/ssh/sshd_config` を書き換えました。

```
sudo nano /etc/ssh/sshd_config
```

| 変更前 | 変更後 |
| --- | --- |
| `#Port 22` | `Port 4242` |
| `#PermitRootLogin prohibit-password` | `PermitRootLogin no` |

```
sudo systemctl restart ssh
```

- 行の先頭の `#` はコメント（無視される）。`#` を消して、設定として有効にした
- 22 番はよく攻撃されるので、4242 番に変えた
- root は何でもできるので、乗っ取られると被害が大きい。そのため SSH では入れないようにした

確認：

```
dpkg -l openssh-server | cat
sudo systemctl status ssh                                       # active (running)
sudo grep -E "^Port|^PermitRootLogin" /etc/ssh/sshd_config      # Port 4242 / PermitRootLogin no
sudo sshd -T | grep -Ei "^port|permitrootlogin"                 # 実際に使われている設定
sudo ss -tlnp | grep ssh                                        # 4242 で待っている
```

ホストのターミナルから：

```
ssh <ユーザー名>@localhost -p 4242     # 入れる
ssh root@localhost -p 4242            # 正しいパスワードでも Permission denied
```

何回か失敗すると `Connection closed by 127.0.0.1 port 4242` と出て、SSH が接続を切ります。

---

### monitoring.sh

サーバーの情報を、起動したときと10分ごとに全員の画面に表示するスクリプト（コマンドをまとめて書いたファイル）です。

```
sudo nano /usr/local/bin/monitoring.sh
sudo chmod +x /usr/local/bin/monitoring.sh     # 実行の許可を付ける（x = execute）
sudo /usr/local/bin/monitoring.sh              # 手動で実行
```

#### スクリプトの中身

```
#!/bin/bash

arch=$(uname -a)
pcpu=$(grep "physical id" /proc/cpuinfo | sort -u | wc -l)
vcpu=$(grep -c "^processor" /proc/cpuinfo)
mem=$(free -m | awk '$1 == "Mem:" {printf("%d/%dMB (%.2f%%)", $3, $2, $3*100/$2)}')
disk=$(df -m | grep "^/dev/" | grep -v "/boot$" | awk '{used += $3; total += $2} END {printf("%d/%dGb (%d%%)", used, total/1024, used*100/total)}')
cpu=$(vmstat 1 2 | tail -1 | awk '{printf("%.1f%%", 100 - $15)}')
lastboot=$(uptime -s | cut -d: -f1,2)
if [ $(lsblk | grep -c "lvm") -gt 0 ]; then lvm=yes; else lvm=no; fi
tcp=$(ss -ta | grep -c ESTAB)
userlog=$(users | wc -w)
ip=$(hostname -I | awk '{print $1}')
mac=$(ip link | grep "link/ether" | awk '{print $2}')
sudocmd=$(grep -c "COMMAND" /var/log/sudo/sudo.log)

msg="#Architecture: $arch
#Physical CPU: $pcpu
#vCPU: $vcpu
#Memory Usage: $mem
#Disk Usage: $disk
#CPU load: $cpu
#Last boot: $lastboot
#LVM use: $lvm
#TCP Connections: $tcp ESTABLISHED
#User log: $userlog
#Network: IP $ip ($mac)
#Sudo: $sudocmd cmd"

wall "$msg"

for pts in /dev/pts/[0-9]*; do
	echo "$msg" > "$pts" 2>/dev/null
done
```

表示例：

```
#Architecture: Linux hsachie42 6.12.111+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.111-1 (2026-09-28) x86_64 GNU/Linux
#Physical CPU: 1
#vCPU: 1
#Memory Usage: 246/1973MB (12.47%)
#Disk Usage: 1172/26Gb (4%)
#CPU load: 0.0%
#Last boot: 2026-10-17 11:01
#LVM use: yes
#TCP Connections: 1 ESTABLISHED
#User log: 3
#Network: IP 10.0.2.15 (08:00:27:82:92:67)
#Sudo: 18 cmd
```

#### 流れ

スクリプトは「**① 調べる → ② まとめる → ③ 表示する**」の3つに分かれています。

| 部分 | 行 | 何をしているか |
| --- | --- | --- |
| 最初 | `#!/bin/bash` | このファイルは bash で実行する、という宣言 |
| ① 調べる | `arch=` 〜 `sudocmd=` | 12個の情報をコマンドで調べて、結果をそれぞれ変数に入れる |
| ② まとめる | `msg="..."` | 調べた結果を、1つの文章にまとめて `msg` に入れる |
| ③ 表示する | `wall` と `for` | `msg` を全員の画面に表示する |

#### ① 調べる（arch 〜 sudocmd）

`変数=$(コマンド)` は、「コマンドを実行して、その**結果**を変数（名前の付いた入れ物）に入れる」という意味です。

| 変数 | 調べていること | コマンドの中身 |
| --- | --- | --- |
| `arch` | OS のアーキテクチャとカーネルのバージョン | `uname -a`：OS の情報を全部（all）表示する |
| `pcpu` | 物理 CPU の数 | `/proc/cpuinfo`（CPU の情報）から `physical id` の行を取り出し、`sort -u` で同じものをまとめて、`wc -l` で行数を数える |
| `vcpu` | 仮想 CPU の数 | `/proc/cpuinfo` で `processor` で始まる行（`^` = 行の始まり）を `grep -c` で数える |
| `mem` | メモリの使用量と使用率 | `free -m`（MB 単位）の `Mem:` の行から、3列目（used）と2列目（total）を取り出して、`used / total` と使用率を計算する |
| `disk` | ディスクの使用量と使用率 | `df -m` で `/dev/` で始まる行（本物のディスク）だけ取り出し、`/boot` を除いて、使用量と全体を足し合わせて使用率を計算する。全体は `/1024` で GB にする |
| `cpu` | CPU の使用率 | `vmstat 1 2` で1秒おきに2回測り、`tail -1` で2回目（今の値）を取り出して、`100 −` 15列目（idle = 何もしていない割合）で計算する |
| `lastboot` | 最後に起動した日時 | `uptime -s` で起動した日時を出して、`cut -d: -f1,2` で秒を消す |
| `lvm` | LVM を使っているか | `lsblk` で `lvm` の行を数えて、`-gt 0`（0より大きい）なら `yes`、そうでなければ `no` |
| `tcp` | つながっている接続の数 | `ss -ta`（TCP の接続の一覧）で `ESTAB`（つながっている）の行を数える |
| `userlog` | ログインしている数 | `users` でログインしているユーザーを出して、`wc -w` で単語の数を数える |
| `ip` | IP アドレス | `hostname -I` の1つ目 |
| `mac` | MAC アドレス | `ip link` の `link/ether` の行の2列目 |
| `sudocmd` | sudo を使った回数 | `/var/log/sudo/sudo.log` で `COMMAND` が入った行を数える（失敗した sudo も含む） |

- `top` や `who -b` は、値や言語の設定で表示の形が変わり、`awk` で取り出す列がずれた。そのため、いつも同じ形で出る `vmstat` と `uptime -s` を使っている
- cron から root として実行するので、スクリプトの中では `sudo` を付けていない

#### ② まとめる（msg）

```
msg="#Architecture: $arch
#Physical CPU: $pcpu
...
#Sudo: $sudocmd cmd"
```

- `"` から `"` までの文章を、`msg` という変数に入れる
- `#Architecture:` などは画面にそのまま出る見出し。`$arch` などは、①で入れた中身に置き換わる（例：`#vCPU: $vcpu` → `#vCPU: 1`）

**なぜ `msg` にまとめるのか**

- 12行を**1回でまとめて**表示できる
- `wall` と `for` の**2か所で同じ文章を使う**ので、1つにまとめておくと、同じものを2回書かなくてすむ
- 表示の形を変えたいときも、`msg` の中だけ直せばいい

#### 使っている書き方

| 書き方 | 意味 |
| --- | --- |
| `#!/bin/bash` | このファイルは bash で実行する、という宣言 |
| `\|`（パイプ） | 左のコマンドの結果を、右のコマンドに渡す |
| `grep "文字"` | その文字が入っている行だけ取り出す |
| `grep -c` / `grep -v` | 行を数える / 当てはまらない行だけ |
| `wc -l` / `wc -w` | 行の数 / 単語の数 |
| `awk '{print $2}'` | 2列目を取り出す |
| `$( ... )` | コマンドの結果に置き換える |
| `msg="..."` | 変数に入れる。`$msg` で取り出す |

#### ③ 表示する（wall と for）

```
wall "$msg"

for pts in /dev/pts/[0-9]*; do
	echo "$msg" > "$pts" 2>/dev/null
done
```

| 行 | 何をしているか |
| --- | --- |
| `wall "$msg"` | `wall` は、ログインしている全員の画面にメッセージを表示するコマンド。VirtualBox の画面（tty1）に表示される |
| `for pts in /dev/pts/[0-9]*; do` | `/dev/pts/` の中の、数字の名前のもの（SSH でつないでいる画面）を1つずつ `pts` に入れて、くり返す |
| `echo "$msg" > "$pts" 2>/dev/null` | その画面に `msg` を直接書き込む。`2>/dev/null` でエラーは表示しない |
| `done` | くり返しの終わり |

**なぜ `for` が必要なのか**

- 本当は `wall` だけで全員の画面に出るはずだが、Debian 13 では SSH の画面に届かなかった
- Debian 13 の `wall` は `loginctl` で送り先の端末を探すが、SSH のログインは `loginctl list-sessions` で TTY が空（`-`）になっていて、`wall` が SSH の画面に届かない
- TTY はログイン時に自動で決まるもので、自分で登録する設定ではない。そのため OS 側は変えず、スクリプトで対応した
- `/dev/pts/` は SSH でつないでいる画面（`0`、`1`、`2`…）が入っている場所。`for` で1つずつ取り出して、直接書き込む
- `2>/dev/null` でエラーを表示しない（課題で「エラーを表示しないこと」と決まっている）

```
tty                        # 今の画面（/dev/pts/0 など）
ls /dev/pts                # SSH でつないでいる画面の一覧
loginctl list-sessions     # SSH のセッションは TTY が -
```

#### cron で10分ごとに動かす

cron は、決めた時間に、決めたコマンドを自動で実行する仕組みです。

```
sudo crontab -u root -e
```

```
@reboot /usr/local/bin/monitoring.sh
*/10 * * * * /usr/local/bin/monitoring.sh
```

```
*/10  *  *  *  *   /usr/local/bin/monitoring.sh
 分   時  日  月  曜日
```

| 書き方 | 意味 |
| --- | --- |
| `@reboot` | 起動したとき |
| `*/10 * * * *` | 10分ごと（時計の分が 0、10、20…のとき） |
| `* * * * *` | 1分ごと |

`#` で始まる行は、cron が最初から書いている説明（コメント）です。

確認：

```
sudo crontab -u root -l
systemctl status cron
```

#### 1分ごとに変える

```
sudo crontab -u root -e     # */10 * * * * を * * * * * に変える
sudo crontab -u root -l
```

SSH をもう1つつなぐと TCP Connections と User log が増え、sudo を使うと Sudo の数が増えます。

#### スクリプトを書き換えずに止める

```
sudo systemctl disable --now cron    # 今すぐ止めて、起動しても動かないようにする
systemctl status cron                # inactive (dead)
sudo systemctl enable --now cron     # また動かす
```

- `stop` だけだと、再起動したら cron がまた動いて、`@reboot` でスクリプトが表示される
- `disable --now` は「今すぐ止める」と「起動したときに動かさない」をまとめてやる

再起動後の確認：

```
sudo reboot
ls -l /usr/local/bin/monitoring.sh     # 同じ場所にある / -rwxr-xr-x のまま / 日付が作った日のまま
cat /usr/local/bin/monitoring.sh       # 中身が同じ
sha1sum /usr/local/bin/monitoring.sh   # 止める前と同じ値なら1文字も変わっていない
```

---

### Linux のフォルダ

| フォルダ | 何が入っているか |
| --- | --- |
| **`/etc`** | **設定ファイル** |
| `/home` | ユーザーのファイル（`/home/hsachie`） |
| `/usr/bin` | プログラム（`ls`、`cat` など） |
| `/var/log` | 記録（ログ）のファイル（sudo の記録は `/var/log/sudo/`） |

`etc` は「et cetera（その他いろいろ）」の略です。

```
ls /etc            # フォルダの中身を見る
ls -l /etc/ssh     # くわしく見る（d = フォルダ、- = ファイル）
cat /etc/hostname  # ファイルの中身を見る
```

---

### 設定したファイルの一覧

Linux の設定は、ほとんどファイルに書いています。`sudo cat ファイル名` で中身を確認できます。

**主な4つ**

| 設定 | ファイル |
| --- | --- |
| SSH | `/etc/ssh/sshd_config` |
| パスワードの期限 | `/etc/login.defs` |
| パスワードの強さ | `/etc/pam.d/common-password` |
| sudo のルール | `/etc/sudoers.d/sudo_config` |

**すべて**

| 設定 | ファイル | 書いたこと | 確認するコマンド |
| --- | --- | --- | --- |
| SSH | `/etc/ssh/sshd_config` | `Port 4242`、`PermitRootLogin no` | `sudo grep -E "^Port\|^PermitRootLogin" /etc/ssh/sshd_config` |
| パスワードの期限 | `/etc/login.defs` | `PASS_MAX_DAYS 30`、`PASS_MIN_DAYS 2`、`PASS_WARN_AGE 7` | `grep "^PASS_" /etc/login.defs` |
| パスワードの強さ | `/etc/pam.d/common-password` | `pam_pwquality.so retry=3 minlen=10 ...` | `grep pwquality /etc/pam.d/common-password` |
| sudo のルール | `/etc/sudoers.d/sudo_config` | `Defaults passwd_tries=3` など8行 | `sudo cat /etc/sudoers.d/sudo_config` |
| ホスト名 | `/etc/hostname`、`/etc/hosts` | `hsachie42` | `cat /etc/hostname` / `cat /etc/hosts` |
| monitoring.sh | `/usr/local/bin/monitoring.sh` | スクリプト | `cat /usr/local/bin/monitoring.sh` |
| cron | root の crontab | `@reboot ...`、`*/10 * * * * ...` | `sudo crontab -u root -l` |
| apt | `/etc/apt/sources.list` | `deb cdrom:` の行の先頭に `#`（ISO を探しに行って止まるのを防ぐ） | `grep cdrom /etc/apt/sources.list` |

- `grep` はファイルの中から決めた文字の行だけを表示するコマンド
- 書き換えは `sudo nano ファイル名` で行った（sudo の設定は `sudo visudo -f /etc/sudoers.d/sudo_config`、cron は `sudo crontab -u root -e`）

**ファイルではなく、コマンドで設定したもの**

| 設定 | コマンド |
| --- | --- |
| グループ | `sudo groupadd user42`、`sudo usermod -aG sudo hsachie`、`sudo usermod -aG user42 hsachie` |
| 既存ユーザーのパスワードの期限 | `sudo chage -M 30 -m 2 -W 7 hsachie`（root も） |
| UFW | `sudo ufw default deny incoming`、`sudo ufw allow 4242`、`sudo ufw enable` など |
| 表示言語 | `sudo dpkg-reconfigure locales`（`en_US.UTF-8`。コンソールは日本語を表示できないため） |

---

### 入れたもの

| ソフト | 用途 |
| --- | --- |
| `openssh-server` | SSH |
| `ufw` | ファイアウォール |
| `sudo` | 決められた範囲で管理者の権限を使う |
| `libpam-pwquality` | パスワードの強さのチェック |
| `cron` | monitoring.sh を定期的に動かす（最初から入っている） |

GUI は入れていません。

---

## Instructions（使い方）

### 起動のしかた

1. VirtualBox で仮想マシンを起動する
2. `Please unlock disk sda5_crypt:` と出たら、ディスクの暗号化のパスフレーズを入れる
3. `hsachie42 login:` と出たら、`hsachie` とパスワードを入れる
4. `hsachie@hsachie42:~$` と出たらログインできている

### SSH でつなぐ

VirtualBox のポートフォワーディングで、ホストの 4242 番を仮想マシンの 4242 番につないでいます。
（Settings → Network → Adapter 1（NAT）→ Advanced → Port Forwarding）

```
ssh hsachie@localhost -p 4242
```

root ではログインできません。

```
ssh root@localhost -p 4242     # Permission denied
```

### 止め方

```
sudo poweroff
```

VirtualBox の × で閉じるときは「Send the shutdown signal」を選びます。

---

## 評価項目と確認方法

評価シートの順番に、各項目の確認方法をまとめています。

### 0. 事前確認

| 項目 | 確認方法 |
| --- | --- |
| `signature.txt` がリポジトリの一番上にある | `ls` |
| `.vdi` の場所 | ホストで `VBoxManage list hdds`（`Location:`）、または VirtualBox Manager → File → Tools → Media |
| 指紋が同じ | `sha1sum <.vdi の場所>` と `cat signature.txt` を比べる |
| スナップショットがない | VirtualBox Manager → Snapshots（0個） |
| 仮想マシンが起動していない | VirtualBox Manager で Powered Off |
| 起動 | 先にスナップショットを取ってから Start（`.vdi` 本体を書き換えないため） |

---

### 1. README.md

| 項目 | 場所 |
| --- | --- |
| 1行目 | `*This project has been created as part of the 42 curriculum by hsachie.*` |
| 必要な項目 | Description / Instructions / Resources / Project description |
| OS の選択と長所・短所 | 「OS は Debian にした」 |
| 主な設計（パーティション、セキュリティ、ユーザー管理、サービス） | Project description の各項目 |
| 4つの比較 | 「比較」 |

---

### 2. プロジェクトの概要

| 項目 | 説明 |
| --- | --- |
| 仮想マシンの仕組み | VirtualBox が、PC の CPU・メモリ・ディスクの一部を借りて、PC の中にもう1台パソコンを作る |
| 仮想マシンの目的 | 本物の PC に影響を与えずに、別の OS を試したり、サーバーを作ったりできる。壊しても作り直せる |
| OS の選択 | Debian。課題で推奨されていて、情報が多く設定がしやすい |
| Debian と Rocky の違い | Debian はコミュニティ製で `apt`・AppArmor。Rocky は Red Hat 互換の企業向けで `dnf`・SELinux。設定が難しい |
| apt と aptitude の違い | apt はシンプル。aptitude はメニュー画面でも使え、ソフト同士がぶつかったときに解決方法を提案する高機能なもの |
| AppArmor | プログラムごとにさわっていい範囲を決める仕組み。乗っ取られても被害を最小限にできる（`sudo aa-status`） |
| スクリプト | 評価中、10分ごとに monitoring.sh がサーバーの情報を表示する |

---

### 3. 簡単な設定の確認

| 項目 | 確認方法 |
| --- | --- |
| GUI がない | 起動時にターミナルだけ。`dpkg -l \| grep -E "xserver-xorg\|xwayland\|gdm3\|lightdm\|sddm"`（何も出ない） |
| 接続前のパスワード | 起動時にディスクの暗号化のパスフレーズを聞かれる |
| root ではないユーザーでログイン | `hsachie` |
| UFW が動いている | `sudo ufw status` → `Status: active` |
| SSH が動いている | `sudo systemctl status ssh` → `active (running)` |
| OS | `cat /etc/os-release` → Debian 13 |

---

### 4. ユーザー（1）

| 項目 | 確認方法 |
| --- | --- |
| ログイン名のユーザーがいる | `id hsachie` |
| `sudo` と `user42` に入っている | `groups hsachie` |
| 新しいユーザーを作る | `sudo adduser <名前>`（ルールに合わないパスワードは弾かれる） |
| 新しいユーザーにも期限のルールが入っている | `sudo chage -l <名前>` → 2 / 30 / 7 |
| パスワードのルールを設定したファイル | `/etc/login.defs`（期限）、`/etc/pam.d/common-password`（強さ）。既存ユーザーには `chage` |

```
grep "^PASS_" /etc/login.defs
grep pwquality /etc/pam.d/common-password
```

---

### 5. ユーザー（2）

| 項目 | 確認方法 |
| --- | --- |
| `evaluating` グループを作る | `sudo groupadd evaluating` |
| 新しいユーザーを入れる | `sudo usermod -aG evaluating <名前>` |
| 入っているか確認 | `groups <名前>` |
| パスワードのルールの利点 | 推測されにくく破られにくい。期限があるので、もれても使われ続けない |
| 実装の利点 | 変えるときに自動でチェックされる。root にも使える |
| 実装の欠点 | 覚えにくく、紙にメモしたり使い回したりしがち。30日ごとの変更がめんどう。設定前のパスワードはチェックされないので変え直しが必要 |

くわしくは「パスワードのルール」→「このルールの利点」「実装の利点と欠点」。

---

### 6. ホスト名とパーティション

| 項目 | 確認方法 |
| --- | --- |
| ホスト名が `ログイン名42` | `hostname` → `hsachie42` |
| ホスト名を変えて再起動 | `sudo hostnamectl set-hostname <名前>` → `/etc/hosts` の `127.0.1.1` の行も変更 → `sudo reboot` → `hostname` |
| 元に戻す | 同じ手順で `hsachie42` |
| パーティション | `lsblk`（`crypt` の下に `lvm` の root・swap_1・home） |
| LVM | ディスクの仕切りを、あとから動かして大きさを変えられる仕組み。暗号化した sda5 の中を LVM で root・swap・home に分けている |

---

### 7. sudo

| 項目 | 確認方法 |
| --- | --- |
| sudo が入っている | `sudo --version` → `Sudo version ...`（または `dpkg -l sudo \| cat` → `ii`） |
| 新しいユーザーを sudo グループに入れる | `sudo usermod -aG sudo <名前>` → `groups <名前>` |
| sudo とは | そのコマンドだけ root の権限で実行する。root でずっと作業するより安全で、記録が残る（例：`apt update` はエラー、`sudo apt update` は実行できる） |
| sudo のルール | `sudo cat /etc/sudoers.d/sudo_config` |
| `/var/log/sudo/` にファイルがある | `sudo ls /var/log/sudo` |
| 記録の中身 | `sudo cat /var/log/sudo/sudo.log` |
| 記録が増える | `sudo whoami` → `sudo tail -4 /var/log/sudo/sudo.log` |

---

### 8. UFW

| 項目 | 確認方法 |
| --- | --- |
| UFW が入っている | `sudo ufw version` → `ufw 0.36.2` など（または `dpkg -l ufw \| cat` → `ii`） |
| 動いている | `sudo ufw status verbose` → `Status: active` |
| UFW とは | 入口（ポート）を管理するファイアウォール。必要な入口だけ開けて、攻撃されにくくする |
| ルールの一覧（4242 がある） | `sudo ufw status numbered` |
| 8080 を足す | `sudo ufw allow 8080` → `sudo ufw status numbered` |
| 8080 を消す | `sudo ufw delete allow 8080`、または `sudo ufw delete <番号>`（v6 の分も） |

---

### 9. SSH

| 項目 | 確認方法 |
| --- | --- |
| SSH が入っている | `dpkg -l openssh-server \| cat` → `ii` |
| 動いている | `sudo systemctl status ssh` → `active (running)` |
| SSH とは | ネットワークを通して、離れたコンピューターにログインして操作する仕組み。通信が暗号化される |
| 4242 だけを使っている | `sudo grep -E "^Port\|^PermitRootLogin" /etc/ssh/sshd_config` / `sudo ss -tlnp \| grep ssh` |
| 新しいユーザーで SSH ログイン | ホストで `ssh <名前>@localhost -p 4242` |
| root では SSH できない | ホストで `ssh root@localhost -p 4242` → `Permission denied`（`su -` では root になれるので、パスワードは正しい） |

---

### 10. スクリプト

| 項目 | 確認方法 |
| --- | --- |
| コードと動き | `cat /usr/local/bin/monitoring.sh`（調べる → まとめる → 表示する） |
| cron とは | 決めた時間に、決めたコマンドを自動で実行する仕組み |
| 起動時と10分ごとの設定 | `sudo crontab -u root -l`（`@reboot` と `*/10 * * * *`） |
| 1分ごとに変える | `sudo crontab -u root -e` → `* * * * *` |
| 値が変わる | SSH をもう1つつなぐ → TCP / User log が増える。sudo を使う → Sudo が増える |
| cron が動いている | `systemctl status cron` → `active (running)` |
| 書き換えずに止める | `sudo systemctl disable --now cron`（今すぐ止めて、起動しても動かないようにする。`stop` だけだと再起動で動き出す） |
| 止まったか確認 | `systemctl status cron` → `inactive (dead)` |
| 再起動後もスクリプトが動かない | `sudo reboot` → `systemctl status cron` が `inactive (dead)` のまま、表示が出ない |
| 同じ場所・同じ権限・同じ中身 | `ls -l /usr/local/bin/monitoring.sh`（`-rwxr-xr-x`、日付が変わっていない）、`cat /usr/local/bin/monitoring.sh` |
| 元に戻す | `sudo systemctl enable --now cron`（スナップショットに Restore するなら不要） |

---

### 11. ボーナス

ボーナスは実装していません。

---

### 12. 評価のあと

1. 仮想マシンを止める（`sudo poweroff`）
2. 評価の前に取ったスナップショットに戻す（Restore。「Create a snapshot of the current machine state」のチェックは外す）
3. そのスナップショットを消す（Delete）
4. `sha1sum <.vdi の場所>` で、指紋が `signature.txt` と同じか確認する

## Resources（参考にしたもの）

### 資料

- Born2beRoot の課題PDF と評価シート
- Debian 管理者ハンドブック：https://debian-handbook.info/
- 各コマンドのマニュアル（`man sshd_config`、`man ufw`、`man login.defs`、`man chage`、`man pam_pwquality`、`man sudoers`、`man crontab`、`man wall`、`man awk`）
- VirtualBox のマニュアル：https://www.virtualbox.org/manual/

### AI の使い方

Claude（AI）を、わからないところの補佐として使いました。用語や設定の意味がわからないとき、エラーの原因がわからないときに質問しました。
