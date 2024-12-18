#　Flatcar のインストール

1. iso ファイルをダウンロード
* https://www.flatcar.org/docs/latest/installing/bare-metal/booting-with-iso/#docslatest

2. KVMで起動
cockpit から操作

3. インストール
  * flatcar を操作するマシンで、公開鍵を付与したignition.yaml を作成
```
cat ~/.ssh/id_rsa.pub
```
`ignition.yaml`
```
variant: flatcar
version: 1.0.0
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDGdByTgSVHq.......
```
  * flatcarのコンソールに戻ってignition.yamlを取得する
```
scp <user>@<IP>:~/ignition.yaml ./
sudo flatcat-install -d /dev/sda -C stable
```
  * yaml から json に transpile
```
cat cl.yaml | docker run --rm -i quay.io/coreos/butane:latest > ignition.json
```
  * スクリプト実行
```
sudo fdisk -l
sudo flatcar-install -d /dev/sda -C stable -i ~/ignition.json
```
4. リブートする
```
reboot
```

5. インストールされたflatcar にアクセスする
```
ssh core@<IP> -i ~/.ssh/id_rsa
```
※IPは起動した画面に表示されている

## 参考
* Installing Flatcar Container Linux to disk
https://www.flatcar.org/docs/latest/installing/bare-metal/installing-to-disk/