## 検討事項

### アプリケーションより下のレイヤの脆弱性

Rails では静的コンテンツの配布に nginx を使う場合もあるが、nginx の脆弱性は bundler-audit や retire.js では検出できない。

もしセキュリティテストのスコープに、インフラ・OS・ミドルウェア等も含めている場合、別の手段で脆弱性情報を収集する必要がある。

### 代わりのツール

- bundler-audit の代わりに、GitHubで使えるDependabotを使ってもよい。
- コンテナ：ECR Image Scanning、
- VMだったらVulsとか？