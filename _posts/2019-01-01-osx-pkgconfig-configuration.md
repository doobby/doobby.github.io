---
title: OSX pkg-config 配置问题小记
layout: post
tags: dev linux osx
author: larry
---



# 问题

OSX 上安装 gitpage 本地调测环境时，`nokogiri` 编译报错：无法找到 `libxml2` 头文件：

```make
conftest.c:3:10: fatal error: 'libxml/xmlversion.h' file not found
#include <libxml/xmlversion.h>
         ^~~~~~~~~~~~~~~~~~~~~
1 error generated.
checked program was:
/* begin */
1: #include "ruby.h"
2:
3: #include <libxml/xmlversion.h>
/* end */
```

# 解法

通过分析错误日志编译时的依赖关系是通过 `pkg-config` 获取的

```bash
✗ pkg-config --cflags-only-I libexslt
-I/usr/include/libxml2
```

系统上的 pkg-config 和 libxml2 已经通过 brew 安装。系统并没有 `/usr/include/libxml2`这个目录。实际上 brew 安装的 libxml2 应该在 `/usr/local/Cellar/libxml2` 目录下。

`pkg-config` 的工作机制是查找 PKG_CONFIG_PATH 中的 `.pc` （描述信息）来获得库的依赖关系的。OSX homebrew 的 `PKG_CONFIG_PATH` 可以从 [ruby 脚本](https://github.com/Homebrew/homebrew-core/blob/master/Formula/pkg-config.rb) 中找到：

```ruby
      #{HOMEBREW_PREFIX}/lib/pkgconfig
      #{HOMEBREW_PREFIX}/share/pkgconfig
      /usr/local/lib/pkgconfig
      /usr/lib/pkgconfig
      #{HOMEBREW_LIBRARY}/Homebrew/os/mac/pkgconfig/#{MacOS.version}
    ].uniq.join(File::PATH_SEPARATOR)
```

加入 `--debug` 参数查看详细加载过程

```bash
✗ pkg-config --debug --cflags libxml-2.0
Error printing enabled by default due to use of output options besides --exists, --atleast/exact/max-version or --list-all. Value of --silence-errors: 0
Error printing enabled
Adding virtual 'pkg-config' package to list of known packages
Looking for package 'libxml-2.0'
Looking for package 'libxml-2.0-uninstalled'
Reading 'libxml-2.0' from file '/usr/local/Homebrew/Library/Homebrew/os/mac/pkgconfig/10.12/libxml-2.0.pc'
Parsing package file '/usr/local/Homebrew/Library/Homebrew/os/mac/pkgconfig/10.12/libxml-2.0.pc'
  line>prefix=/usr
 Variable declaration, 'prefix' has value '/usr'
  line>exec_prefix=${prefix}
 Variable declaration, 'exec_prefix' has value '/usr'
  line>libdir=${exec_prefix}/lib
 Variable declaration, 'libdir' has value '/usr/lib'
  line>includedir=${prefix}/include
 Variable declaration, 'includedir' has value '/usr/include'
  line>modules=1
 Variable declaration, 'modules' has value '1'
  line>
  line>Name: libXML
  line>Version: 2.9.4
  line>Description: libXML library version2.
  line>Requires:
  line>Libs: -L${libdir} -lxml2
  line>Libs.private: -lpthread -lz  -lm
Unknown keyword 'Libs.private' in '/usr/local/Homebrew/Library/Homebrew/os/mac/pkgconfig/10.12/libxml-2.0.pc'
  line>Cflags: -I${includedir}/libxml2
Path position of 'libxml-2.0' is 4
Adding 'libxml-2.0' to list of known packages
Package libxml-2.0 has -L /usr/lib in Libs
Removing -L /usr/lib from libs for libxml-2.0
 post-recurse: libxml-2.0
adding CFLAGS_OTHER string ""
 post-recurse: libxml-2.0
 original: libxml-2.0
   sorted: libxml-2.0
adding CFLAGS_I string "-I/usr/include/libxml2 "
returning flags string "-I/usr/include/libxml2"
-I/usr/include/libxml2
```

实际上 `pkg-config` 命中了 `PKG_CONFIG_PATH` 中的最后一条 `/usr/local/Homebrew/Library/Homebrew/os/mac/pkgconfig/10.12/libxml-2.0.pc`

```pc
prefix=/usr
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include
modules=1

Name: libXML
Version: 2.9.4
Description: libXML library version2.
Requires:
Libs: -L${libdir} -lxml2
Libs.private: -lpthread -lz  -lm
Cflags: -I${includedir}/libxml2
```

在 libxml2 安装目录中我们找到真正的 .pc 文件：`/usr/local/Cellar/libxml2/2.9.8/lib/pkgconfig/libxml-2.0.pc`

```pc
prefix=/usr/local/Cellar/libxml2/2.9.8
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${prefix}/include
modules=1

Name: libXML
Version: 2.9.8
Description: libXML library version2.
Requires:
Libs: -L${libdir} -lxml2
Libs.private:  -lpthread -lz   -liconv -lm
Cflags: -I${includedir}/libxml2
```

需要修改 `PKG_CONFIG_PATH` ，指定 libxml2 的 pc 目录，再次进行安装

```bash
export PKG_CONFIG_PATH=/usr/local/Cellar/libxml2/2.9.8/lib/pkgconfig:${PKG_CONFIG_PATH}
bundle install
# OK
```

# 后记

折腾一圈，发现实际在安装 libxml2 时 brew 已经提示我们要做这个配置

```bash
~ brew info libxml2
...
If you need to have libxml2 first in your PATH run:
  echo 'export PATH="/usr/local/opt/libxml2/bin:$PATH"' >> ~/.zshrc

For compilers to find libxml2 you may need to set:
  export LDFLAGS="-L/usr/local/opt/libxml2/lib"
  export CPPFLAGS="-I/usr/local/opt/libxml2/include"

For pkg-config to find libxml2 you may need to set:
  export PKG_CONFIG_PATH="/usr/local/opt/libxml2/lib/pkgconfig"

```

不要忽略任何提示信息，_Now is better than never_ :-)
