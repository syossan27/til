# Gitのhooksスクリプトの作成

## スクリプトファイルの場所

`$ .git/hooks`の中にある。
ただし、ファイル名に.sampleとついてるので使用する際は.sampleを外してあげる。

## 作ってみたpre-commitスクリプト

```pre-commit
#!/bin/sh
if git rev-parse --verify HEAD >/dev/null 2>&1
then
>-against=HEAD
else
>-# Initial commit: diff against an empty tree object
>-against=4b825dc642cb6eb9a060e54bf8d69288fbee4904
fi

# Redirect output to stderr.
exec 1>&2

IS_ERROR=0
echo "\033[0;33mrubocopとreekを実行しよう！\033[0;39m"
```

参考記事：
[gitのpre-commit hookを使って、綺麗なPHPファイルしかコミットできないようにする](http://blog.manaten.net/entry/645)

ついでの参考にechoの文字色を変えるリンクをば。
[シェルでechoの文字に色をつける方法](http://d.hatena.ne.jp/R-H/20110119/1295452747)
