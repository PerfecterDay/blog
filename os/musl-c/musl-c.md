
make clean && ./configure --enable-debug CFLAGS="-O0" 保留debug 信息编译，这样编译出来的 obj 文件带有调式信息，debug 的时候能用
musl-gcc -g -O0 -static hello.c -o hello