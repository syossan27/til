# assetsの読み込みを早くする

## 余計なファイルを読み込んでいないか？

当たり前の話だが、余計なファイルを読み込んでいる場合は遅くなる。
例えば、assets以下にpc/spとディレクトリを分けている場合に `application.js` と `pc/application.js` 、`sp/application.js` で同じJSをimportすると遅延が発生する。

なので、必要なものだけを読み込むようにしよう。

## config.assets.debugがtrueになっていないか？

development環境の設定によくあるのが `config.assets.debug = true` という設定。
`config.assets.debug` 自体はJS/CSSファイルを連結するかどうかの設定である。
これがfalseになっている場合には、個別のJS/CSSファイルを参照できるのでデバッグがしやすい。

ただ、連結しない場合は遅いので開発時にproduction環境と同じ早さでassetsを読み込みたい時には `false` にする。
