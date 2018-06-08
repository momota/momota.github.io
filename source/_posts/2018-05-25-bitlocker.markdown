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


全体の流れは以下。

1. BitLocker 解除用の PowerShell を書く
2. バッチファイル unlock-bitLocker.cmd を書く
3. バッチファイル unlock-bitLocker.cmd をタスクスケジューラに登録する
4. タスクを呼び出すバッチファイル trigger_unlock-bitLocker.cmd を書く
5. バッチファイル unlock-bitLocker.cmd をスタートアッププログラムに登録してあげる

<!-- more -->

## 1. BitLocker 解除用の PowerShell を書く

ここでは D ドライブの BitLocker 解除するスクリプト `C:\PATH-YOU-WANT\unlock-bitLocker.ps1` を書く。


```ps1
$password = ConvertTo-SecureString "YOUR_PASSWORD" -AsPlainText -Force
$d_drive = "D:"

Unlock-BitLocker -MountPoint $d_drive -Password $password
```


複数のドライブがある場合は以下。

```ps1
$password = ConvertTo-SecureString "YOUR_PASSWORD" -AsPlainText -Force
$drives = @("D:", "E:", "F:", "G:", "H:")

Foreach($drive in $drives) {
  Unlock-BitLocker -MountPoint $drive -Password $password
}
```


## 2. バッチファイル unlock-bitLocker.cmd を書く

上述の PowerShell スクリプトを管理者権限で実行し、かつ、スタートアッププログラムに登録するために、上記の PowerShell を呼び出すバッチファイル `C:\PATH-YOU-WANT\unlock-bitLocker.cmd` を書いてあげる。
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


## 3. バッチファイル unlock-bitLocker.cmd をタスクスケジューラに登録する

`unlock-bitLocker.cmd` をスタートアッププログラムに登録すると、実行時に管理者権限で実行するかという確認ダイアログがでて毎回うざったい。
そこで、いったん上記のバッチスクリプトをタスクスケジューラに管理者権限実行のタスクとして登録してそのタスクをスタートアッププログラムから起動する。

1. `Windows キー + r` : ファイル名を指定して実行を起動
2. `taskschd.msc` を入力: タスクスケジューラが開く
3. 新規タスクの作成: 
    - 「タスクの作成」から `unlock_bitlocker.cmd` を起動するタスクを作成する
    - 「全般」タブの「最上位の特権で実行する」にチェックを入れる
    - 「操作」タブから「新規」ボタンを押す。操作は「プログラムの開始」、プログラム/スクリプトに `C:\PATH-YOU-WANT\unlock-bitLocker.cmd` を指定
    - 「トリガー」は空でOK


## 4. タスクを呼び出すバッチファイル trigger_unlock-bitLocker.cmd を書く


```bat
@echo off
setlocal
rem ------------------------------------------------------------------

schtasks.exe /run /tn unlock_bitlocker

rem ------------------------------------------------------------------
endlocal
exit
```

## 5. バッチファイル unlock-bitLocker.cmd をスタートアッププログラムに登録してあげる


1. `Windows キー + r` : ファイル名を指定して実行を起動
2. `shell:startup` を入力: スタートアップのフォルダが開く
3. 作ったバッチファイル `trigger_unlock-bitLocker.cmd` のショートカットをスタートアップフォルダの中に作成

以上で、OS ログイン後に D ドライブの BitLocker を解除できる。



BitLocker を有効にしていると、初回起動時に NumLock が有効になっているのを無効化したいんだが、だれか方法を教えてほしい。
