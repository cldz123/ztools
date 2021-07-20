# ztools

`ztools` 是个具有多功能的命令行工具，目前主要的功能是对模块打静态补丁，辅助模块注入。

### 一、静态补丁功能

命令格式 `ztools.exe patch` 后面接参数。

每个参数含义如下：

|  参数        | 含义  | 备注 |
|  ---         | ---   | --- |
| `type`         | 静态补丁类型   | 1. `gap` - 空隙补丁<br/> 2. `hook` - 静态hook补丁<br/> 3. `depend` - 模块依赖<br/> 4. `hijack` - 库劫持|
| `injectee`     | 目标模块的路径 | 文件完整路径 |
| `injecter`     | 注入模块的路径 | 文件完整路径 |
| `patch-offset` | 被打补丁的代码偏移  | 十六进制整型数 |
| `patch-expsym` | 被打补丁的导出函数名称 | 内部通过导出函数获取被打补丁的代码偏移 |
| `load-path`    | `dlopen`/`LoadLibraryA` 加载时模块的路径| 如果未设置，则默认为 `injecter` 指定的文件名称 |
| `scene-mode`   | 指令现场保护模式 | `gat/hook` 类型有效<br/> 1. `none` 一般模式<br/> 2. `small` - 最小保护模式<br/> 3. `normal` - 普通模式 <br/> 4. `media` - 多媒体模式|
| `callee-mode`  | 覆盖函数后原函数的调用时机 | `gat` 类型有效<br/> 1. `after` - 加载模块之后调用函数<br/> 2. `before` - 加载模块之前调用函数|
| `export-sym`   | 注入模块的导出函数名称 | `hook` 类型有效<br/>如果注入模块存在指定的导出函数，则该导出函数会填充静态 `hook` 回调指令<br/>如果不存在，工具会自动添加空隙以及导出函数，用于填充静态hook回调指令|
| `add-loader`   | 查找不到加载库的必要函数时是否导入这些函数 | 默认值为 `true`<br/>`true` 表示找不到 `dlopen/dlsym` 等必要函数，就自动添加<br/> `false` 表示找不到 `dlopen/dlsym` 等必要函数，就返回失败|
| `bak`          | 是否备份 | 默认值为 `true`<br/>`true` 表示修改模块前先备份模块<br/>`false` 表示不备份|
| `hijack-name`  | 库劫持后的名称 | `hijack` 类型有效<br/>库劫持后，被劫持库修改后的名称 |


目前主要包含四种方式

- 空隙补丁
- 静态hook补丁
- 模块依赖
- 模块劫持

#### 1. 空隙补丁

**命令格式**

```
ztools.exe patch --type=gap  --injectee="D:\\git\\gat\\test\\gencode" --patch-offset=0x0081A7B0 --load-path="libloader2.so" --scene-mode=small --callee-mode=after --add-loader=true --bak=true
```

其中 `--scene-mode` `--callee` `--add-loader` `--bak` 是可选的，如果没设置均有默认值。

在当前命令格式下，需要获取一个函数地址，然后替换这个函数地址，达到劫持程序执行流程的目的。

此时 `--patch-offset` 的值

- 如果是代码段的偏移，则对应的指令必需是 `CALL\JUMP` 指令
- 如果是数据段偏移，则必须从该处获取数据是个代码函数地址（可以是初始化列表、虚表、`ELF`头的`e_entry`、导出表等）

#### 2. 静态HOOK补丁

**命令格式**

```
ztools.exe patch --type=hook --injectee="D:\\git\\gat\\test\\gencode" --patch-offset=0x0007C5A0 --load-path="libloader2.so" --scene-mode=small --injecter="D:\\git\\gat\\test\\libloader2.so" --export-sym=_Z3asfv --add-loader=true --bak=true
```

其中 `--scene-mode` `--add-loader` `--bak` 是可选的，如果没设置均有默认值。

在当前命令格式下，不需要函数地址，`--patch-offset` 可以是任意一块代码的偏移，该偏移处的代码会被修改成补丁代码。这样可以选择任意的时机加载指定模块。还可以通过 `--patch-expsym` 指定 `--patch-offset` 为某个导出函数的偏移。

#### 3. 模块依赖

**命令格式**

```
ztools.exe patch --type=depend --injectee="D:\\git\\gat\\test\\gencode" --load-path="libloader2.so" --bak=true
```

其中 `--bak` 是可选的，如果没设置均有默认值。

当前命令格式相对简单，只是在目标模块里面添加一个依赖模块。

#### 4. 模块劫持

**命令格式**

```
ztools.exe patch --type=hijack --injectee="D:\\git\\gat\\test\\libc.so" --hijack-name=libc2.so --injecter="D:\\git\\gat\\test\\libloader2.so" --add-loader=true --bak=true
```

其中 `--add-loader` `--bak` 是可选的，如果没设置均有默认值

当前命令格式下，`--injectee` 指定被劫持的模块，`--injecter` 指定修改的目标模块也就是注入模块。工具将读取 `--injectee` 指定模块的所有导出函数，并加入到 `--injectee` 模块中。

参数 `--hijack-name` 指定被劫持的模块替换后的模块名称。

#### 静态注入原理

可以参考公众号 过既冲云 系列文章

[静态注入](https://mp.weixin.qq.com/s?__biz=MzIzNDA0OTc3OA==&amp;mid=2247483776&amp;idx=1&amp;sn=b7e0b057e1548b93337e920d702c9693&amp;chksm=e8fd181ddf8a910bf97dfaafcd0cf2ea0a8cc0a712a93357255d588ba6b7c84736fef848643f&token=1399599051&lang=zh_CN#rd)

[静态注入二](https://mp.weixin.qq.com/s?__biz=MzIzNDA0OTc3OA==&amp;mid=2247483800&amp;idx=1&amp;sn=5fc2843d4a845216561b2c1020167cf7&amp;chksm=e8fd1805df8a91134b72746a37a39eae63b7fcf7b332f8011460004a95e18852c26a6f952e3d&token=1399599051&lang=zh_CN#rd)

[修改ELF文件](https://mp.weixin.qq.com/s?__biz=MzIzNDA0OTc3OA==&amp;mid=2247483810&amp;idx=1&amp;sn=f518974936a3f14ae1387a073a129e96&amp;chksm=e8fd183fdf8a9129ea8141280f720f893d22cf59e1c7513accf4c659a8ecdfbef1c5297f467f&token=1399599051&lang=zh_CN#rd)

[库劫持](https://mp.weixin.qq.com/s?__biz=MzIzNDA0OTc3OA==&amp;mid=2247483817&amp;idx=1&amp;sn=439614ab6338846468816535a4e19e65&amp;chksm=e8fd1834df8a91223528b9a713d44559fdc0674bafd0fe39affb5d993c996e32ea84a3cb37ef&token=1399599051&lang=zh_CN#rd)


### 公众号

![过既冲云](./qrcode.jpg)

### QA

