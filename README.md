# Rails security testing guide

## 前提知識

- [セキュリティテスターのためのRuby on Rails コードリーディング](https://docs.google.com/presentation/d/18zITuFTR0AvYEZhuBc-drzuc6OTyAGovwO0GSuXZu5s/edit?usp=sharing)

## テストガイド

- 4.01 情報収集
  - [06 エントリーポイントの特定](01-Information_Gathering/06-Identify_Application_Entry_Points.md)
- 4.05 認可
  - [02 Bypassing Authorization Schema](05-Authorization_Testing/02-Testing_for_Bypassing_Authorization_Schema.md)
  - [04 Insecure Direct Object Reference (IDOR)](05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References.md)
- 4.06 セッション管理
  - [05 Cross Site Request Forgery (CSRF)](06-Session_Management_Testing/05-Testing_for_Cross_Site_Request_Forgery.md)
- 4.07 入力値検証
- 4.10 ビジネスロジック
  - [03 Testing Integrity Checks - Mass Assignment](10-Business_Logic_Testing/03-03-Test_Integrity_Checks_Mass_Assignment.md)
- 4.11 クライアントサイド
  - [11 Cross Origin Resource Sharing (CORS)](11-Client-side_Testing/07-Testing_Cross_Origin_Resource_Sharing.md)
- 4.12 API
  - [x1 Ransack による情報漏洩](12-API_Testing/xx-Testing_Ransack.md)
- 4.xx 未分類
  - [x1 JSONからの情報漏洩](XX-misc/XX-JSONからの情報漏洩.md)
  - [x2 Denial of Service (DoS) - 上限値がない](XX-misc/XX-Denial_of_Service_上限値がない.md)
