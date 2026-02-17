---
title: certutil -deletehellocontainer コマンド実行時の注意点
date: 2026-02-18 09:00
tags:
  - Microsoft Entra ID
  - Windows Hello for Business
---

# certutil -deletehellocontainer コマンド実行時の注意点

こんにちは、Azure & Identity サポート チームの長谷川です。今回は certutil -deletehellocontainer コマンド実行時の注意点について説明します。

## 説明
最新の Windows 11 では certutil -deletehellocontainer コマンドを実行すると Windows Hello for Business だけでなくその Windows 上に保存されたパスキーもクリアされます。
このため、その Windows 11 上に Windows Hello for Business 以外のパスキーを保存している場合は、certutil -deletehellocontainer コマンド実行前に他のデバイスでパスキーをセットアップしておくなどして、その Windows 上でパスキーがクリアされた後もそのアカウントでサインインできるように準備してください。

certutil -deletehellocontainer コマンドはこれまで、 [既にプロビジョニングされた Windows Hello for Business をリセットする方法](../azure-active-directory/how-to-disable-whfb.md) として利用されていました。
certutil -deletehellocontainer コマンドは Windows Hello for Business のプロビジョニングによって作成されたキー情報が保存されるコンテナ空間をクリアするコマンドです。
最新の Windows 11 ではこのコンテナ空間にパスキーも保存されるようになりました。
このため certutil -deletehellocontainer コマンドでこのコンテナ空間をクリアすると、Windows Hello for Business だけでなくその Windows 11 上に保存されたパスキーもクリアされることになるという仕組みです。
よって certutil -deletehellocontainer コマンドを実行する際には事前準備が必要という流れになります。


## 補足
その Windows 11 上に保存されているパスキーは「スタート」 > 「設定」 > 「アカウント」 > 「パスキー」で確認することができます。
(下図では一番上にリストされている ***.onmicrosoft.com が Windows Hello for Business の情報で、残りの 2 つが他のサイトで作成したパスキーです。)
![Windows 上に保存されたパスキーの確認画面](./certutil-deletehellocontainer/certutil-deletehellocontainer1.jpg)


## おわりに
certutil -deletehellocontainer コマンドはトラブル解消のためなどに利用されるコマンドです。このコマンドが更なるトラブルを起こさないよう上記を参考にしてください。
製品動作に関する正式な見解や回答については、お客様環境などを十分に把握したうえでサポート部門より提供しますので、ぜひ弊社サポート サービスをご利用ください。