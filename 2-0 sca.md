# ソフトウェアコンポジション解析 (SCA)：ライブラリの脆弱性

どんなに注意深くソースコードを書いたとしても、脆弱性が0になることはありません。なぜならアプリケーションを動かすためのライブラリやミドルウェアにも脆弱性は存在しうるためです。

Rails アプリケーションも多くの Gem や Javascript ライブラリを使って開発します。ソースコードに問題がなくても Gem に脆弱性があれば Web アプリケーションは危険にさらされることになります。

本章ではツールを使ってライブラリの脆弱性を発見し、その脆弱性のプロダクトに対する影響の有無を確認します。

- bundler-audit を使ってGemの脆弱性を検出する
- retire.js を使ってJavascriptライブラリの脆弱性を検出する
- 検出された脆弱性の詳細を NATIONAL VULNERABILITY DATABASE (NVD) で調査し、プロダクトが影響を受けるか否かを確かめる