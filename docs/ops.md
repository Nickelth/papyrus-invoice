### 機密情報のマスク

アクセスキーやアカウントIDをログに載せてコミットしてしまったとき用。

`~/docs/replacement.txt`に記載の`regex`に従いログファイルの文字を置換する。

```bash
git filter-repo --replace-text /docs/replacements.txt
```