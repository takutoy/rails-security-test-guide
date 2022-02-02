# Rails security testing guide

Rails アプリケーションの脆弱性を発見し検証するためのセキュリティテストガイドです。脆弱性ごとに観点と手法を解説します。

1. 静的テスト（ソースコードレビュー）による脆弱性の発見
2. 動的テスト（脆弱性診断）による脆弱性の検証

## 想定する利用者

Railsでアプリケーション開発をしている組織のセキュリティテスターを想定しています。Railsのソースコードをある程度読めるスキルが必要です。

セキュリティテスター以外でも、脆弱性を作りこみたくない Rails アプリ開発者やQA部門のテスターにも役立つかもしれません。

## 目次

番号は [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) に合わせています。

- 前提知識
  - [セキュリティテスターのためのRuby on Rails コードリーディング](https://docs.google.com/presentation/d/18zITuFTR0AvYEZhuBc-drzuc6OTyAGovwO0GSuXZu5s/edit?usp=sharing)
- 4.01 情報収集
  - [06 エントリーポイントの特定](01-Information_Gathering/06-Identify_Application_Entry_Points.md)
- 4.05 認可
  - [02 Bypassing Authorization Schema](05-Authorization_Testing/02-Testing_for_Bypassing_Authorization_Schema.md)
  - [04 Insecure Direct Object Reference (IDOR)](05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References.md)
- 4.06 セッション管理
  - [05 Cross Site Request Forgery (CSRF)](06-Session_Management_Testing/05-Testing_for_Cross_Site_Request_Forgery.md)
  - [07 Session Timeout](06-Session_Management_Testing/07-Testing_Session_Timeout.md)
- 4.07 入力値の取り扱い
  - [05 SQL Injection](07-Input_Validation_Testing/05-Testing_for_SQL_Injection.md)
  - [11 Code Injection](07-Input_Validation_Testing/11-Testing_for_Code_Injection.md)
  - [12 Command Injection](07-Input_Validation_Testing/12-Testing_for_Command_Injection.md)
- 4.10 ビジネスロジック
  - [03 Testing Integrity Checks - Mass Assignment](10-Business_Logic_Testing/03-Test_Integrity_Checks_Mass_Assignment.md)
  - [03 Testing Integrity Checks - 外部キーの不正な更新](10-Business_Logic_Testing/03-Test_Integrity_Checks_Foreign_Key_Manipulation.md)
- 4.11 クライアントサイド
  - [11 Cross Origin Resource Sharing (CORS)](11-Client-side_Testing/07-Testing_Cross_Origin_Resource_Sharing.md)
- 4.99 未分類
  - [JSONからの情報漏洩](99-misc/Improper_Json_Serialization_Information_Leakage.md)
  - [Denial of Service (DoS) - 上限値がない](99-misc/Denial_of_Service_Unlimited_Number.md)
  - [Denial of Service (DoS) - ReDoS](99-misc/Denial_of_Service_ReDoS.md)
  - [変数のスコープの誤りによる競合や漏洩](99-misc/Improper_Variable_Scope_Integrity_Compromise.md)
  - [Ransack からの情報漏洩](99-misc/Ransack_Information_Leakage.md)
  - [脆弱性が報告されているGemの使用](99-misc/Using_Vulnerable_Gem.md)