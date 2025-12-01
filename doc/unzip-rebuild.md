Download and navigate to unzip source code

  ```
  git clone https://github.com/LuaDist/unzip.git
  cd unzip
  ```

INSTALL file shows that compilation can be done as below

  ```
  make -f unix/Makefile generic
  ```

But I notice the below

```
file unzip
unzip: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=8ef0a4aeb7a1f597b10b35d880efb36786f0b200, for GNU/Linux 3.2.0, stripped
```

`unzip` stripped and does not have debuginfo, which will lead to broken stack again such as with the OS unzip

I experimented with the compilation options and came to (not clean) an approach, but it did the trick (as strip and optimization options are hardcoded in configure and Makefile and it didn't take my environment variables -g -fno-omit-frame-pointer)

```
sed -i -e 's/STRIP = strip/STRIP = /' -e 's/CFLAGS = -O/CFLAGS = -g -fno-omit-frame-pointer/' -e 's/O3/-g -fno-omit-frame-pointer/' -e 's/^CF_NOOPT.*/CF_NOOPT = -I. -I$(IZ_BZIP2) -DUNIX $(LOC) $(CFLAGS)/' unix/Makefile
sed -i -e 's/CFLAGS_OPT.*-O3.*/CFLAGS_OPT=/' -e 's/^LFLAGS2.*/LFLAGS2=/' unix/configure
```

Now by executing the above make command again we get the debuginfo included along with the a **no omit** frame pointer

```
file unzip
unzip: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=fe373f206b4534e3c888138b4ef6575a5b51fb64, for GNU/Linux 3.2.0, with debug_info, not stripped
```






