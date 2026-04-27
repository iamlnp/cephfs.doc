文件位置： 
- `rgw_file.cc`  
- `librgw.cc`


```c++
//rgw_file_int.h
class RGWLibFS {

public:
    int unlink(RGWFileHandle* rgw_fh, const char *name, uint32_t flags = FLAG_NONE)
}
```


```c++
static RGWLib rgwlib; //全局静态
int librgw_create(librgw_t* rgw, int argc, char **argv)
```