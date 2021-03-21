# 静的アプリケーションセキュリティテスト (SAST)：ソースコードの脆弱性

本章では [railsgoat](https://github.com/OWASP/railsgoat) の脆弱性をSASTツールで検出し、結果を目視で精査します。また、独自のルールによるセキュリティコードレビューの実例も紹介します。

## 目次

1. 静的セキュリティテストの準備
2. Rails のコードをBrakemanで静的解析する
3. Rails のコードを独自のルールでレビューする
4. javascript のコードを CodeQL で静的解析する