---
title: ローカル管理者パスワードがバックアップされているデバイス一覧を取得する
date: 2026-01-29 09:00
tags:
  - Microsoft Entra
  - Device
  - Windows LAPS
toc:
  enabled: true
  min_depth: 1
  max_depth: 4
  list_number: false
---
# ローカル管理者パスワードがバックアップされているデバイス一覧を取得する

こんにちは、Azure & Identity サポート チームの長谷川です。
今回は Microsoft Entra ID にローカル管理者パスワードがバックアップされているデバイス一覧を取得する方法のサンプルを紹介します。

Windows LAPS の機能を利用すると、Microsoft Entra 参加済みデバイスと Microsoft Entra ハイブリッド参加済みデバイスのローカル管理者アカウントとそのパスワード情報を Entra ID 上に保存することができます。
この Entra ID 上に保存したローカル管理者アカウントの情報は、管理者が Entra 管理センター上でデバイスごとに確認することができます。
しかしながら、Entra 管理センター上では一覧としてダウンロードする機能は実装されておりません。
そこで、Microsoft Graph PowerShell モジュールのコマンドを使用して Microsoft Entra ID にローカル管理者パスワードがバックアップされているデバイス一覧を CSV に出力する方法のサンプルを紹介します。

サンプルでは、CSV にローカル管理者アカウントとパスワードを含めた出力方法と含めない出力方法の二種類を紹介します。



## CSV にローカル管理者アカウントとパスワードを含めた出力方法サンプル 
1. PowerShell を管理者権限で起動します。
2. 以下のコマンドを実行し、Microsoft Graph PowerShell モジュールをインストールします。(既にモジュールがインストールされている場合はスキップしてください)
~~~
Install-Module Microsoft.Graph -Force
~~~
 
3. 以下のコマンドを実行し、グローバル管理者でサインインします。([要求されているアクセス許可] という画面が表示された場合は、[承諾] を押下します)
※もしグローバル管理者を利用しない場合は、実行するユーザーに二つのロール (クラウド アプリケーション管理者とクラウド デバイス管理者) が付与されていれば実行可能です。
~~~
Connect-MgGraph -Scopes "DeviceLocalCredential.Read.All", "Directory.Read.All" 
~~~

4. 以下のコマンドを順に実行して、ローカル管理者アカウントのパスワードがバックアップされているデバイスの一覧をパスワード関連の情報を含んだ形で CSV ファイルとしてデスクトップに出力します。
~~~
$outfile = "$env:USERPROFILE\Desktop\DevicelistWithLapsIncludingPassword.csv"
$data = @()
$targetDevices = Get-MgDirectoryDeviceLocalCredential -All
foreach ($targetDevice in $targetDevices) {
    $data += Get-LapsAADPassword -DeviceIds $targetDevice.Id -IncludePasswords -AsPlainText | select @{Name="Id"; Expression={$_.DeviceId}}, DeviceName, PasswordUpdateTime, PasswordExpirationTime, Account, Password
}
$data | Export-Csv $outfile -encoding "utf8" -NoTypeInformation
~~~

5. 作業完了後、以下のコマンドでセッションを切断し作業を終了します。
~~~
Disconnect-MgGraph
~~~

上記コマンド実行時の PowerShell 画面イメージは以下の通りです。
![](./get-laps-password/get-laps-password1-1.jpg)

## CSV にローカル管理者アカウントとパスワードを含めた出力方法 CSV のサンプル
CSV にローカル管理者アカウントとパスワードを含めて出力した CSV のサンプルは以下の通りです。
![](./get-laps-password/get-laps-password1-2_DevicelistWithLapsIncludingPassword.jpg)


## ローカル管理者パスワードがバックアップされているデバイス一覧のみを CSV に出力する方法のサンプル 
1. PowerShell を管理者権限で起動します。
2. 以下のコマンドを実行し、Microsoft Graph PowerShell モジュールをインストールします。(既にモジュールがインストールされている場合はスキップしてください)
~~~
Install-Module Microsoft.Graph -Force
~~~
 
3. 以下のコマンドを実行し、グローバル管理者でサインインします。([要求されているアクセス許可] という画面が表示された場合は、[承諾] を押下します)
※もしグローバル管理者を利用しない場合は、実行するユーザーに二つのロール (クラウド アプリケーション管理者とクラウド デバイス管理者) が付与されていれば実行可能です。
~~~
Connect-MgGraph -Scopes "DeviceLocalCredential.Read.All", "Directory.Read.All" 
~~~

4. 以下のコマンドを順に実行して、ローカル管理者アカウントのパスワードがバックアップされているデバイスの一覧をパスワード関連の情報を含んだ形で CSV ファイルとしてデスクトップに出力します。
~~~
$outfile = "$env:USERPROFILE\Desktop\DevicelistWithLaps.csv"
$data = @()
$data = Get-MgDirectoryDeviceLocalCredential -All | select Id, DeviceName, @{Name="PasswordUpdateTime"; Expression={$_.LastBackupDateTime.AddHours(9)}}, @{Name="PasswordExpirationTime"; Expression={$_.RefreshDateTime.AddHours(9)}}
$data | Export-Csv $outfile -encoding "utf8" -NoTypeInformation
~~~

5. 作業完了後、以下のコマンドでセッションを切断し作業を終了します。
~~~
Disconnect-MgGraph
~~~

上記コマンド実行時の PowerShell 画面イメージは以下の通りです。
![](./get-laps-password/get-laps-password2-1.jpg)

## ローカル管理者パスワードがバックアップされているデバイス一覧のみを出力した際の CSV のサンプル
CSV にローカル管理者パスワードがバックアップされているデバイス一覧のみを出力した CSV のサンプルは以下の通りです。
![](./get-laps-password/get-laps-password2-2_DevicelistWithLaps.jpg)


## 免責事項
本サンプル コードは、あくまでも説明のためのサンプルとして提供されるものであり、製品の実運用環境で使用されることを前提に提供されるものではありません。
本サンプル コードおよびそれに関連するあらゆる情報は、「現状のまま」で提供されるものであり、商品性や特定の目的への適合性に関する黙示の保証も含め、明示・黙示を問わずいかなる保証も付されるものではありません。
マイクロソフトは、お客様に対し、本サンプル コードを使用および改変するための非排他的かつ無償の権利ならびに本サンプル コードをオブジェクト コードの形式で複製および頒布するための非排他的かつ無償の権利を許諾します。
但し、お客様は以下の 3 点に同意するものとします。
(1) 本サンプル コードが組み込まれたお客様のソフトウェア製品のマーケティングのためにマイクロソフトの会社名、ロゴまたは商標を用いないこと
(2) 本サンプル コードが組み込まれたお客様のソフトウェア製品に有効な著作権表示をすること
(3) 本サンプル コードの使用または頒布から生じるあらゆる損害（弁護士費用を含む）に関する請求または訴訟について、マイクロソフトおよびマイクロソフトの取引業者に対し補償し、損害を与えないこと


## おわりに
本記事では Microsoft Entra ID にローカル管理者パスワードがバックアップされているデバイス一覧の取得方法を紹介しました。製品動作に関する正式な見解や回答については、お客様環境などを十分に把握したうえでサポート部門より提供しますので、ぜひ弊社サポート サービスをご利用ください。
