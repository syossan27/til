# JSXで改行を理解させる方法

## What is this?

JSXで普通に改行文字を含んだ文字列を表示させると、改行文字の部分が半角スペースとして表示される。

なので、これを `<br>` に変換してあげることが必要となる。

## dangerouslySetInnerHTMLを使う

これは基本的に使っちゃダメなやーつ。

dangerouslySetInnerHTMLで実行可能な危険なコードもそのまま出しちゃうのでダメ。

## \nを<br>に変換する

component内に変換するメソッドを作成し、それを適宜使う。

```
nl2br: function (text) {
	var regex = /(\n)/g
	return text.split(regex).map(function (line, index) {
		if (line.match(regex)) {
			return React.createElement('br', { key: index })
		} else {
			return line
		}
	});
},
```

こんな感じのメソッドを用いればOK.

map使ってるので気持ち悪いけど `<br>` にkeyを仕込まないと怒られる。
