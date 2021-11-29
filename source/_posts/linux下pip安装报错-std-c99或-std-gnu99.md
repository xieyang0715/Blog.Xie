---
title: linux下pip安装报错-std=c99或-std=gnu99
date: 2021-08-31 09:21:41
tags:
- 生产小经验
---



# 如图

```bash
    'configure' finished successfully (1.211s)
    'make_all' finished successfully (0.004s)
    Waf: Entering directory `/tmp/pip-install-pd6p8p_f/pyinstaller/bootloader/build/debug'
    [ 1/19] Compiling src/pyi_python.c
    [ 2/19] Compiling src/pyi_global.c
    [ 3/19] Compiling src/pyi_archive.c
    ../../src/pyi_archive.c: 在函数‘_pyi_find_cookie_offset’中:
    ../../src/pyi_archive.c:417:9: 错误：只允许在 C99 模式下使用‘for’循环初始化声明
             for (size_t i = chunk_size - MAGIC_SIZE + 1; i > 0; i--) {
             ^
    ../../src/pyi_archive.c:417:9: 附注：使用 -std=c99 或 -std=gnu99 来编译您的代码
    
    Waf: Leaving directory `/tmp/pip-install-pd6p8p_f/pyinstaller/bootloader/build/debug'
    Build failed
     -> task in 'OBJECTS' failed with exit status 1 (run with -v to display more information)
    ERROR: Failed compiling the bootloader. Please compile manually and rerun setup.py
    
    ----------------------------------------
Command "/root/.pyenv/versions/3.6.6/envs/magedu366/bin/python3.6 -u -c "import setuptools, tokenize;__file__='/tmp/pip-install-pd6p8p_f/pyinstaller/setup.py';f=getattr(tokenize, 'open', open)(__file__);code=f.read().replace('\r\n', '\n');f.close();exec(compile(code, __file__, 'exec'))" install --record /tmp/pip-record-ojjxli1o/install-record.txt --single-version-externally-managed --compile --install-headers /root/.pyenv/versions/3.6.6/envs/magedu366/include/site/python3.6/pyinstaller" failed with error code 1 in /tmp/pip-install-pd6p8p_f/pyinstaller/
You are using pip version 10.0.1, however version 21.2.4 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
```



pip install 添加编译选项

```bash
CFLAGS="-std=gnu99" pip install pyinstaller
```

