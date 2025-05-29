# Tcc (Tiny C Compiler)

版本：

```
tcc version 0.9.28rc 2025-05-25 mob@83de5325 (riscv64 Linux)
```

简单程序测试：

```
cd tcc && tcc bubble.c -o bubble
```

TCC编译器自举：

```
cd tcc && tcc tcc.c -o new_tcc
```

测试新的编译器：

```
./new_tcc ../a.c -o a
./a
```
