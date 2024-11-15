---
title: Apple M1 编译 electron sqlite3
date: 2024-11-15 20:00:03
tags:
  - electron
  - sqlite3
categories:
  - code
---
使用 cnpm 安装的 sqlite3 会出现内核不匹配问题，报错信息如下：

```log
App threw an error during load
Error: dlopen(/Users/liull/00_beluga/slient-printer-app/node_modules/.store/sqlite3@5.1.7/node_modules/sqlite3/build/Release/node_sqlite3.node, 0x0001): 
tried: '/Users/liull/00_beluga/slient-printer-app/node_modules/.store/sqlite3@5.1.7/node_modules/sqlite3/build/Release/node_sqlite3.node'
 (mach-o file, but is an incompatible architecture (have 'x86_64', need 'arm64')),
  '/System/Volumes/Preboot/Cryptexes/OS/Users/liull/00_beluga/slient-printer-app/node_modules/.store/sqlite3@5.1.7/node_modules/sqlite3/build/Release/node_sqlite3.node' (no such file), 
  '/Users/liull/00_beluga/slient-printer-app/node_modules/.store/sqlite3@5.1.7/node_modules/sqlite3/build/Release/node_sqlite3.node' (mach-o file, but is an incompatible architecture (have 'x86_64', need 'arm64'))
```

核心就是下载的 sqlite3 内核是 x86_64 而 apple M1 的内核是 arm64。一种办法是从[仓库](https://registry.npmmirror.com/binary.html?path=sqlite3/v5.1.7/) 找准下载替换本地对应 `node_modules/sqlite3/build/Release/node_sqlite3.node` 路径文件。一种办法是依据源码进行编译。

1. 需要全局安装 electron `cnpm install electron -g`

2. 需要全局安装 node-gyp `cnpm install node-gyp -g`

3. 下载 sqlite3 源码 `cnpm install sqlite3 --ignore-scripts`，其中 `--ignore-script` 是为了防止 sqlite3 自动执行编译

4. 手动执行脚本进行编译

   ```shell
    cd node_modules/sqlite3
    node-gyp rebuild --target=electron版本 --arch=arm64 --dist-url=https://electronjs.org/headers --module-name=node_sqlite3 --module_path=../lib/binding/napi-v3-darwin-arm64
   ```

5. 在项目中运行 `cnpm run dev` 运行项目后，正常启动