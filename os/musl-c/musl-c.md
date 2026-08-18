# musl-c 编译语安装
{docsify-updated}

> https://wiki.musl-libc.org/

```
make clean && ./configure --enable-debug CFLAGS="-O0" 保留debug 信息编译，这样编译出来的 obj 文件带有调式信息，debug 的时候能用
musl-gcc -g -O0 -static hello.c -o hello
```

交叉编译，在 linux-aarch64 上编译 x86_64 的 musl-gcc :
```
sudo dnf install -y gcc-x86_64-linux-gnu binutils-x86_64-linux-gnu

./configure --target=x86_64-linux-gnu --prefix=/usr/local/musl-x86_64 --enable-wrapper=gcc CROSS_COMPILE=x86_64-linux-gnu-
make -j4
sudo make install

./tools/install.sh -D obj/musl-gcc /usr/local/musl-x86_64/bin/musl-gcc
```