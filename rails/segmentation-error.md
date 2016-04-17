# Segmentation fault at 0x00000000000000が発生した時の対処

原因は `therubyracer` ？？

対処として https://github.com/cowboyd/therubyracer/issues/341 を参考に `therubyracer` のversionを変更するか、
`therubyracer` と `libv8` をGemから外し、 `bundle clean` を行う。
