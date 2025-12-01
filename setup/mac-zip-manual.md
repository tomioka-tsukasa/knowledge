# Mac生成のZIPにWindowsでの不要ファイルを含めない方法

```shell
# Windowsで不要ファイルを含めないコマンド
zip ファイル名.zip -r ファイル名/ -x "*.DS_Store"
```

## 解説

macOS Finderで通常通りZIPファイル化すると、Windowsユーザーが見た時に「.DS_Store」「__MACOSX」という不要なフォルダが紛れ込こんでしまう問題が発生する。

ZIPファイル化する時に、上記に載せたコマンドで実行すると不要ファイルが紛れない。

他にもサードパーティのアプリケーションを使う方法もあるが、詳しくは下記の記事を参照。

[不要なファイルが紛れ込む問題！？ Macでzip圧縮する時に注意しておきたいこと。
](https://tcd-theme.com/2019/12/mac-zip-compression.html)

