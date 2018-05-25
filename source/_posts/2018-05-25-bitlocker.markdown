---
layout: post
title: "Windows ログイン時に、Cドライブ以外のBitLockerを解除する"
date: 2018-05-25 17:23
comments: true
categories: windows powershell work
---
BitLocker は Windows 10 Professional/Enterprise に標準搭載されているディスク暗号化機能。
最近、会社 PC を Windows 10 にアップデートしたので使うようになった。

BitLocker は OS ログイン前に C ドライブの暗号化を解除しろ、とパスワード入力を求めてくる。
ただし、その後、OS ログインしても、C ドライブ以外 (たとえば、D ドライブ) は暗号化されたままとなっており、いちいち当該ドライブへアクセスしてパスワード入力して暗号化を解除する必要があった。
また、暗号化解除前に D ドライブ上のファイルへアクセスするとエラーが起きてしまい、大変うざったい。

そこで、OS ログイン時に自動的に C ドライブ以外のディスク暗号化解除するために PowerShell とバッチを書いてスタートアッププログラムに登録した話を書く。

セキュリティクラスタには怒られるのかもしれないが、OS に管理者権限ユーザでログインできている時点でもういろいろ見れて当たり前やろ、という思想。

<!-- more -->

## 1. BitLocker 解除用の PowerShell を書く

ここでは D ドライブの BitLocker 解除するスクリプト `C:\PATH-YOU-WANT\unlock-bitLocker.ps1` を書く。


```ps1
$password = ConvertTo-SecureString "YOUR_PASSWORD" -AsPlainText -Force
$d_drive = "D:"

Unlock-BitLocker -MountPoint $d_drive -Password $password
```


## 2. バッチファイルを書く

スタートアッププログラムに登録するために、上記の PowerShell を呼び出すバッチファイル `C:\PATH-YOU-WANT\unlock-bitLocker.cmd` を書いてあげる。
PowerShell を直接登録したかったのだが、調査力が足りず変な構成になった…

もしくは、これくらいなら PowerShell スクリプトはいらず、バッチスクリプトだけでもよかったかもしれない。

```bat
@echo off
setlocal


set DDrive=D:
set /a SleepInterval=5

echo Unlocking bitlocker on %DDrive% drive...
@powershell -Command "Start-Process powershell.exe -ArgumentList C:\PATH-YOU-WANT\unlock_bitlocker.ps1 -Verb runas"
timeout %SleepInterval% /nobreak

if ERRORLEVEL 0 (
  dir %DDrive%
  echo Yey, unlocked bitlocker on %DDrive% drive !!
) else (
  echo Error occured: unloker script returned error code: %ERRORLEVEL%
)


endlocal
pause > nul
exit
```


## 3. バッチファイルをスタートアッププログラムに登録してあげる


1. `Windows キー + r` : ファイル名を指定して実行を起動
2. `shell:startup` を入力: スタートアップのフォルダが開く
3. 作ったバッチファイル `unlock_bitlocker.cmd` のショートカットをスタートアップフォルダの中に作成

以上で、OS ログイン後に D ドライブの BitLocker を解除できる。

正確に言うと、スクリプトの実行に管理者権限が必要なので、ログイン後に「このアプリがデバイスへ変更を加えることを許可しますか」と確認ダイアログが出る。「はい」とボタン押下し許可してあげればOK。
