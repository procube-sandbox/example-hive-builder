# Office 365 との連携設定

IDaaS に SP として Office 365 を登録する際の登録方法を説明します。

#### 補足
> この説明では、Office 365 のユーザの ImmutableID は IDaaS の uid と一致させる前提となっています。
> 文中にでてくる procube-demo.jp は Office365 にフェデレーションドメインとして登録するドメイン名に置き換えてください。


## テナントに対する SP の登録

テスト用のテナントとして tenant1 を追加してください。

### SP の追加
テナントに Office 365 の SP　を以下のように登録してください。

| 属性名          | 値        |
| --------------- | ---------------- |
|公開名             | Microsoft365 |
|SP メタデータ|https://nexus.microsoftonline-p.com/federationmetadata/saml20/federationmetadata.xml からダウンロードしたメタデータ（先頭の <?xml .. ?> は削除してください。）|
|サービスURL|https://www.office.com/|
|配信対象SAML属性|IDPEmail|
|SAMLアサーションを暗号化する|false|
|応答に署名する|false|
|SAMLアサーションに署名する|true|

### SAML属性の追加

また、SAML属性を以下の通り追加してください。

| 属性名          | 値        |
| --------------- | ---------------- |
|属性定義ID|IDPEmail|
|属性|UID|

### テストユーザ

テスト対象のユーザは以下のCSVで登録してください。

```dotnetcli
,tenantid,uid,cn,idmRole,userPassword,dn,accountLock,passwdChangedDatetime,expireDate,data1,data2,data3,data4,data5
Create,tenant1,test1@procube-demo.jp,テスト太郎,IDM_ADMIN,YVKvZzz4z5NGrpV,,FALSE,,,649ea0a4-a981-4a89-94de-c00bc78aaa67,,,,
```

## Office 365側の設定

Office 365 に上記テストユーザの uid と同じ UPN を持つユーザを登録してください。

### ドメインのフェデレーションの設定

Windows 上の Powershell でユーザのドメインをフェデレーションドメインに設定してください。
#### 補足
> MacOS/Linux や Azure Cloud Shell の  PowerShell ではエラーになります。必ず Windows 上のものを使用してください。

```powershell
Install-Module -Name AzureAD
Import-Module AzureAD
Install-Module MSOnline

$dom = "procube-demo.jp" 
$BrandName = "CITS-IDaaS IdP" 
$LogOnUrl = "https://am0.idaas.procube-demo.jp/tenant1/profile/SAML2/POST/SSO" 
$LogOffUrl = "https://am0.idaas.procube-demo.jp/tenant1/profile/SAML2/Redirect/SLO" 

$MyURI = "https://am0.idaas.procube-demo.jp/tenant1/shibboleth" 
$MySigningCert = ".hive/production/saml-certs/saml_cert.pem の内容の最初の行と最後の行を削ったもの" 
$Protocol = "samlp" 
Set-MsolDomainAuthentication `
  -DomainName $dom `
  -FederationBrandName $BrandName `
  -Authentication Federated `
  -PassiveLogOnUri $LogOnUrl `
  -SigningCertificate $MySigningCert `
  -IssuerUri $MyURI `
  -LogOffUri $LogOffUrl `
  -PreferredAuthenticationProtocol $Protocol

```

間違えた値を設定し、変更しても反映されない場合は、一旦 -Authentication managed に設定してから再実行してください。

```powershell
Set-MsolDomainAuthentication -DomainName $dom -Authentication managed 
```

### ユーザの設定

ユーザの ImmutalbleID に IDM 上の uid の値を設定してください。

```powershell
$User = "test1@procube-demo.jp"
Set-MsolUser -UserPrincipalName $User -ImmutableID $User
```

間違えた ImmutableID を設定した場合は、一度フェデレーションじゃないドメインにユーザを移動して、 ImuutableID を書き換え、もう一度もとのドメインに戻してください。

```powershell
$DummyUser = "dummy@procube4.onmicrosoft.com"
Set-MsolUserPrincipalName -UserPrincipalName $User -NewUserPrincipalName $DummyUser
Set-MsolUser -UserPrincipalName $DummyUser -ImmutableID $User
Set-MsolUserPrincipalName -UserPrincipalName $DummyUser -NewUserPrincipalName $User
```

ImmutableID を確認する場合は以下を実行してください。

```powershell
Get-MsolUser -UserPrincipalName $User  | Select-Object ImmutableID
```

### デフォルトドメインの設定

以下でデフォルトドメインを設定してください。

```powershell
$DefaultDom = "フェデレーションを設定していないonmicrosoftのドメイン"
Set-MsolDomain -Name $DefaultDom -IsDefault
```

## エラーメッセージに対する対処

### ケース1.
Access Manager のログイン画面が出ずに以下のエラーが出る場合
> AADSTS500082: SAML assertion is not present in the token.

テナントに追加したSPオブジェクトの署名暗号化関連の属性の値が間違っている可能性があります。
このページの「SP の追加」の節にかかれている内容と一致しているか見直してください。

### ケース2.
Access Manager のログイン後に以下のエラーが出る場合
> AADSTS50107: The requested federation realm object 'https://am0.idaas.procube-demo.jp/tenant1/shibboleth' does not exist.

PowerShell から以下のコマンドで protocol を設定してください。

```powershell
Set-MsolDomainFederationSettings -DomainName $dom  -IssuerUri $MyURI -PreferredAuthenticationProtocol samlp
```

### ケース3.
Access Manager のログイン後に以下のエラーが出る場合
> AADSTS51004: The user account test1@procube-demo.jp does not exist in the b1964c1f-dbbb-4300-8e9e-cdd4c727db40 directory. To sign into this application, the account must be added to the directory.

ImmutableID の設定ができていない可能性があります。IDM上のuid と同じ値が設定されているか、このページの「ユーザの設定」の節を参照して確認してください。
