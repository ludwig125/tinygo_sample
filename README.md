# TinyGo の Overview

https://tinygo.org/getting-started/overview/

> The TinyGo project implements the exact same programming language. However, TinyGo uses a different compiler and tools to make it suited for embedded systems and WebAssembly. It does this primarily by creating much smaller binaries and targeting a much wider variety of systems.

要約

- TinyGo は Go と同じ言語だが、（家電製品や機械用の）組み込みシステムや WebAssembly 用に独自のコンパイラや
  ツールを採用している。
- TinyGo はこの用途を考えて通常の Go より小さいバイナリサイズになる

#### 具体例

以下は、後述の TinyGo のインストール後に行ったものです

同一のプログラムコードに対して、Go と TinyGo の２パターンでビルドして見ました

hello.go

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello world!")
}

```

```
go build -o hello ./hello.go
```

```
tinygo build -o hello_tiny ./hello.go
```

```
$ ls -l
合計 2032
-rwxr-xr-x 1 ludwig125 ludwig125 1766246  1月 22 06:48 hello*
-rw-r--r-- 1 ludwig125 ludwig125      73  1月 22 06:40 hello.go
-rwxr-xr-x 1 ludwig125 ludwig125  306584  1月 22 06:48 hello_tiny*
```

このプログラムの場合、TinyGo のバイナリサイズは通常の Go の五分の一以下となりました。

# TinyGo の Install

https://tinygo.org/getting-started/install/linux/

## Quick install Linux

私の環境は以下の通り Ubuntu 20.4 です

```
$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.3 LTS
Release:        20.04
Codename:       focal
```

> If you are using Ubuntu or another Debian based Linux on an Intel processor, download the DEB file from Github and install it using the following commands:

```
wget https://github.com/tinygo-org/tinygo/releases/download/v0.21.0/tinygo_0.21.0_amd64.deb
sudo dpkg -i tinygo_0.21.0_amd64.deb
```

> You will need to ensure that the path to the tinygo executable file is in your PATH variable.

```
export PATH=$PATH:/usr/local/tinygo/bin
```

`echo $PATH`して末尾に TinyGo があることを確認します

```
$echo $PATH
XXXXXXX（他のコマンド）XXXXXXXXX:/usr/local/tinygo/bin ←これが追加されていればOK
```

> You can test that the installation is working properly by running this code which should display the version number:

```
$tinygo version
tinygo version 0.21.0 linux/amd64 (using go version go1.17 and LLVM version 11.1.0)
```

> If you are only interested in compiling TinyGo code for WebAssembly then you are now done with the installation.

私は WebAssembly が使えればいいので、このページはこれで終了にします

# TinyGo で WASM

https://github.com/golang/go/wiki/WebAssembly#getting-started

と同じコードを TinyGo で動かしてみます。

サンプル用のディレクトリを切ります

```
$ mkdir my-wasm
$ cd my-wasm
```

以下のファイルを用意します（上のドキュメントそのままです）

main.go

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, WebAssembly!")
}
```

```html
<html>
	<head>
		<meta charset="utf-8" />
		<script src="wasm_exec.js"></script>
		<script>
			const go = new Go();
			WebAssembly.instantiateStreaming(
				fetch("main.wasm"),
				go.importObject
			).then((result) => {
				go.run(result.instance);
			});
		</script>
	</head>
	<body></body>
</html>
```

通常の Go では以下のコマンドでビルドしますが、

```bash
$ GOOS=js GOARCH=wasm go build -o main.wasm
```

TinyGo では

https://tinygo.org/docs/guides/webassembly/

を参考に以下のコマンドでビルドします。

```bash
tinygo build -o main.wasm -target wasm main.go
```

これで、`main.wasm`のバイナリが作成されます。

ちなみに、上の TinyGo の WebAssembly のページのコードを使わなかった理由は、
一番シンプルな Hello で動作確認をしたかっただけです

`main.wasm`をブラウザで実行するために、Go のバイナリと Javascript を結びつけるグルーコードである、
`wasm_exec.js`が必要です

このファイルは TinyGo では、`$TINYGOROOT`以下にあります

ということで、以下で持ってこれます

```
cp $(tinygo env TINYGOROOT)/targets/wasm_exec.js .
```

以上で必要はファイルがそろいました

```
$ls
index.html  main.go  main.wasm*  wasm_exec.js
```

あとは
https://github.com/golang/go/wiki/WebAssembly#getting-started

と同様にサーバを実行します

```
$goexec 'http.ListenAndServe(`:8080`, http.FileServer(http.Dir(`.`)))'

```

でサーバを実行して、 `http://localhost:8080/`を見ると以下が表示されました

![image](https://user-images.githubusercontent.com/18366858/150657152-3cf431ef-12c1-4495-a048-e62a61ff6a0f.png)

### 付録

上述の`tinygo build -o main.wasm -target wasm main.go`ですが、
最初私の環境では、以下の結果になってしまいました。

```
$tinygo build -o main.wasm -target wasm main.go
error: could not autodetect root directory, set the TINYGOROOT environment variable to override
```

調べてみると、なんと TINYGOROOT が空になっていました（どうして。。）

```
$echo $TINYGOROOT
空
```

私の環境構築で余計なことをしてしまったのか、調べてもよくわからなかったので、以下の GOROOT を参考に、
TINYGOROOT を設定しました。

通常の Go のパス

```
$echo $GOROOT
/usr/local/go
$ls /usr/local/go
AUTHORS          CONTRIBUTORS  PATENTS    SECURITY.md  api/  codereview.cfg  lib/   pkg/  test/
CONTRIBUTING.md  LICENSE       README.md  VERSION      bin/  doc/            misc/  src/
```

TinyGo は以下になりそうです。

```
$ls /usr/local/lib/tinygo
bin/  lib/  pkg/  src/  targets/
```

そこで、`~/.zshrc` に以下を追記しました

```
$vim ~/.zshrc

以下をどこかに追記
export TINYGOROOT=/usr/local/lib/tinygo
```

`~/.zshrc`の修正を反映させると、以下のように echo で TINYGOROOT が見られるようになりました。

```
$source ~/.zshrc
$echo $TINYGOROOT
/usr/local/lib/tinygo
```

これでビルドが成功しました

```
$ tinygo build -o main.wasm -target wasm main.go
```

ちなみに、tinygo の環境変数は以下で見ることができます（通常の Go は`go env`です）

```
$tinygo env
GOOS="linux"
GOARCH="amd64"
GOROOT="/usr/local/go"
GOPATH="/home/ludwig125/go"
GOCACHE="/home/ludwig125/.cache/tinygo"
CGO_ENABLED="1"
TINYGOROOT="/usr/local/lib/tinygo"
```

以下、必要かわかりません。

# Build from source

https://tinygo.org/docs/guides/build/

TinyGo は LLVM を使います。

```
git clone --recursive https://github.com/tinygo-org/tinygo.git
cd tinygo
```

> Before copying the command below, please replace xxxxx with your distribution’s codename.

```
echo 'deb http://apt.llvm.org/xxxxx/ llvm-toolchain-xxxxx-11 main' | sudo tee /etc/apt/sources.list.d/llvm.list
```

以下のように置き換え =>

```
echo 'deb http://apt.llvm.org/focal/ llvm-toolchain-focal-11 main' | sudo tee /etc/apt/sources.list.d/llvm.list
```

> After adding the apt repository for your distribution you may install the LLVM toolchain packages:

```
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install clang-11 llvm-11-dev lld-11 libclang-11-dev
```

> After LLVM has been installed, installing TinyGo should be as easy as running the following command (cgo must be enabled so that the compile does not fail):

```
CGO_ENABLED=1 go install
```

https://tinygo.org/docs/guides/build/#with-a-self-built-llvm
これ必要なのか定かではない
あとのコマンドで以下が出たのでやっている

> [~/go/src/github.com/ludwig125/tinygo_sample/tinygo/wasm-tinygo-hello] $make build
> GOOS=js GOARCH=wasm tinygo build -target wasm -no-debug -o main.wasm
> error: could not find wasm-opt, set the WASMOPT environment variable to override
> make: \*\*\* [Makefile:2: build] エラー 1

```
sudo apt-get install build-essential git cmake ninja-build
```

https://tinygo.org/docs/guides/build/#additional-requirements

> To be able to use TinyGo to build WebAssembly binaries, you will need to compile wasi-libc:

```
make wasi-libc
```

> hese command may need to be re-run after some updates in TinyGo.

> There are also some extra tools you will need to install, depending on your operating system. These tools are gcc-avr, avr-libc, avrdude, and openocd. Check the additional requirements for your operating system:

=>

https://tinygo.org/docs/guides/webassembly/

```
cp $(tinygo env TINYGOROOT)/targets/wasm_exec.js .

```
