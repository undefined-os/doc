# sqlite3

编译：

```bash
cd sqlite3
tcc sqlite3.c shell.c -o sqlite_new -DSQLITE_THREADSAFE=0 -DSQLITE_MAX_MMAP_SIZE=0 -DNDEBUG -DSQLITE_ENABLE_MATH_FUNCTIONS -DSQLITE_OMIT_LOAD_EXTENSION=1 -lm -Wl,-rpath,/usr/local/lib
```

运行脚本：

```bash
sqlite -echo < 1.sql
./sqlite_new -echo < 2.sql
```
