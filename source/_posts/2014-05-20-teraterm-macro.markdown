---
layout: post
title: "初めてのteraterm macro"
date: 2014-05-20 01:17
comments: true
categories: teraterm macro juniper junos cisco
---
先週末の5/17、teraterm マクロを初めて書いたので記念パピコ。

通信断時間の計測のため、わけあって急遽Cisco ルータ (892J) から継続pingを実行する必要があった。
892J のpingは、リピート回数に上限があるので、上限回数を超えた時に再度pingコマンドを叩く必要があった。
人間がpingの終わりを観察して終わったことを確認して再度実行するのは「ないわーーー」なので初めてのテラタームマクロを書いた。

<!-- more -->

pingが終わったら、打ち直す的な単純な処理だったら、意外と簡単に書けるので好き嫌いせず書いてみるのも悪くないと思った。


```vim
; ----------------------------------------------------------------------
; Cisco ルータのpingリピート回数に上限があるので、上限回数を超えた時に
; 再度pingを実行するマクロ
; 拡張pingを実行するため、enableモードで本マクロを読み込むこと
; ----------------------------------------------------------------------
LOOP_LIMIT = 1000

sendln
for i 1 LOOP_LIMIT
  call ping
next
end

:ping
  wait 'YOUR-ROUTER#'
  sendln 'ping vrf VR-NAME TARGET-IP repeat 10000 timeout 1'
return
```

linux/unix環境だとshell scriptでやったほうが手っ取り早いけど、ネットワーク機器みたいな特殊な環境だとteraterm macroみたいなのが便利なんだなといまさらながら思った。

意外に楽しくなってきたので、もっと書きたくなってきた。


Juniperの情報取得については、改行区切りの複数のコマンドをコピペで流し込むと出力結果の途中に入力コマンドが入り込んでしまう。プロンプトが返ってくるのを待ってから1行ずつコマンドを実行すれば本事象は回避できるので、そのようなteraterm macroを利用しているが、その中身が自分好みではなかったので書き換えてみた。

```vim
; ----------------------------------------------------------------------
; commandlist.txt から1行ずつコマンドを読み込み、実行する
; ----------------------------------------------------------------------
sendln

commandlist = './commandlist.txt'
fileopen filehandle commandlist 0

while 1
  filereadln filehandle command
  if result = 1 break

  call wait_prompt
  sendln command
endwhile

fileclose filehandle
end

; ----------------------------------------------------------------------
; Juniper のプロンプトを待つサブルーチン
:wait_prompt
  wait '@YOUR-HOSTNAME>'
return
```

上記、`.ttl` ファイルと同じところに `commandlist.txt` を置く。
`commandlist.txt` には実行したいコマンドを改行区切りで列挙する以下はEXシリーズのコマンド例。

```
show system uptime | no-more
show version | no-more
show configuration | display set | no-more
show chassis environment | no-more
show chassis routing-engine | no-more
show virtual-chassis | no-more
show virtual-chassis vc-port | no-more
show ethernet-switching table brief | no-more
show arp | no-more
show lacp interfaces | no-more
show route | no-more
show vlans detail | no-more
show interfaces terse | no-more
show firewall | no-more
show log messages | no-more
```

