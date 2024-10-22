## chapter01

### 1. 配置环境

使用 Homebrew 安装

```bash
brew install sdl # 这样最后的display_library是sdl2
brew install bochs
```

这样没有意外的话（当前环境为 macOS 14 Sonoma，Homebrew 4.1.24） bochs 就安装在目录 `/opt/homebrew/Cellar/bochs/2.8/` 中啦～

### 2. 创建软盘

进入项目目录，输入：

```bash
bximage
```

```dotnetcli
❯ bximage

ERROR: Parameter -func missing - switching to interactive mode.

========================================================================
                                bximage
  Disk Image Creation / Conversion / Resize and Commit Tool for Bochs
                                  $Id$
========================================================================

1. Create new floppy or hard disk image
2. Convert hard disk image to other format (mode)
3. Resize hard disk image
4. Commit 'undoable' redolog to base image
5. Disk image info

0. Quit

Please choose one [0] 1     # 选择 1 创建新软盘或硬盘

Create image

Do you want to create a floppy disk image or a hard disk image?
Please type hd or fd. [hd] fd # 输入 `fd` 创建软盘

Choose the size of floppy disk image to create.
Please type 160k, 180k, 320k, 360k, 720k, 1.2M, 1.44M, 1.68M, 1.72M, or 2.88M.
 [1.44M] # 不输入就默认 1.44M 软盘，只能按上面说的几个选

What should be the name of the image?
[a.img] # 不输入就默认软盘文件名称叫 a.img，输入就按输入的名字来

The disk image 'a.img' already exists.  Are you sure you want to replace it?
Please type yes or no. [no] yes

Creating floppy image 'a.img' with 2880 sectors

The following line should appear in your bochsrc:
  floppya: image="a.img", status=inserted
```

这里我之前创建过了所以有 `The disk image 'a.img' already exists.  Are you sure you want to replace it?
Please type yes or no. [no] yes` 这一项。这里就将原来的替换成新的了。

### 3. 编写汇编代码

`boot.asm`: 

```armasm
    org	07c00h			    ; 告诉编译器程序加载到7c00处
    mov	ax, cs
    mov	ds, ax
    mov	es, ax
    call	DispStr		    ; 调用显示字符串例程
    jmp	$			        ; 无限循环
DispStr:
    mov	ax, BootMessage
    mov	bp, ax			    ; ES:BP = 串地址
    mov	cx, 16			    ; CX = 串长度
    mov	ax, 01301h		    ; AH = 13,  AL = 01h
    mov	bx, 000ch		    ; 页号为0(BH = 0) 黑底红字(BL = 0Ch,高亮)
    mov	dl, 0
    int	10h			        ; 10h 号中断
    ret
BootMessage:		db	"Hello, OS world!"
times 	510-($-$$)	db	0	; 填充剩下的空间，使生成的二进制代码恰好为512字节
dw 	0xaa55				    ; 结束标志
```

### 4. 编译汇编代码

这里使用 `nasm` 汇编器：

```bash
nasm boot.asm -o boot.bin
```

生成 `boot.bin` 文件（命令执行后该文件会出现在项目目录下）。

这里好像在前面安装 sdl 和 bochs 的时候已经安装过 nasm 了，如过没有就另外安装：

```bash
brew install nasm
```

### 5. 将编译后的代码写入软盘中

使用如下命令，`if`（代表输入文件）和 `of`（代表输出设备）。编译汇编代码生成的文件作为输入，输出为创建的软盘名。

```bash
dd if=boot.bin of=a.img bs=512 count=1 conv=notrunc
```

现在的目录结构：

```dotnetcli
.
├── a.img
├── boot.asm
└── boot.bin
```

### 6. 配置启动信息 `bochsrc`

当前环境下我们可以在 `/opt/homebrew/Cellar/bochs/2.8/share/doc/bochs/` 下找到一个名为 `bochsrc-sample.txt`，这是一个完整的配置文件模版。根据这个模版我们可以新创建一个 `bochsrc` 文件，只需注意配置好如下信息即可：

```
###############################################################
# Configuration file for Bochs
###############################################################

# how much memory the emulated machine will have
megs: 32

display_library: sdl2

# filename of ROM images
romimage: file=/opt/homebrew/Cellar/bochs/2.8/share/bochs/BIOS-bochs-latest
vgaromimage: file=/opt/homebrew/Cellar/bochs/2.8/share/bochs/VGABIOS-lgpl-latest

# what disk images will be used
floppya: 1_44=a.img, status=inserted

# choose the boot disk.
boot: floppy

# where do we send log messages?
# log: bochsout.txt

# disable the mouse
mouse: enabled=0

# enable key mapping, using US layout as default.
keyboard: keymap=/opt/homebrew/Cellar/bochs/2.8/share/bochs/keymaps/sdl2-pc-us.map
```

注意模版中如果有 `$BXSHARE` 指的应该是 `/opt/homebrew/Cellar/bochs/2.8/share/bochs` 这个目录，

注意 `display_library: sdl2`，这里需要指定显示库为sdl2（还记得我们下载的就是它）。

关于 `boot: floppy`，这里我们只有一个软盘所以可以这么做，要载入多个软盘的话就得指定是哪个了。

还有一个一些教程忽略的就是键盘映射那句，注意指定的映射文件是 `sdl2-pc-us.map` 在后续实验中才能读取输入。很多教程指定的是 `x11-pc-us.map`，应该是其他的库导致的，这里亲测会导致不能映射到键盘的 panic。

其他目前未发现问题。

### 7. bochs，启动！

进入我们的目录，这时候结构如下：

```dotnetcli
.
├── a.img
├── bochsrc
├── boot.asm
└── boot.bin
```

一家人整整齐齐，准备就绪，现在我们就可以启动 bochs，执行我们要测试的汇编代码：

```bash
bochs -f bochsrc
```

这里 `-f` 指定了 bochs 的配置文件。其实也可以不指定，它会默认搜索名为：

- `.bochsrc`
- `bochsrc`
- `bochsrc.txt `
- `bochsrc.bxrc`（仅对Windows有效）

的文件来启动，但如果有多个配置文件就需要了。随后我们看到了一串信息：

```dotnetcli
❯ bochs -f bochsrc
00000000000i[      ] LTDL_LIBRARY_PATH not set. using compile time default '/opt/homebrew/Cellar/bochs/2.8/lib/bochs/plugins'
========================================================================
                        Bochs x86 Emulator 2.8
             Built from GitHub snapshot on March 10, 2024
                Timestamp: Sun Mar 10 08:00:00 CET 2024
========================================================================
00000000000i[      ] BXSHARE not set. using compile time default '/opt/homebrew/Cellar/bochs/2.8/share/bochs'
00000000000i[      ] lt_dlhandle is 0x11c72ad00
00000000000i[PLUGIN] loaded plugin libbx_serial.so
00000000000i[      ] lt_dlhandle is 0x11d808360
00000000000i[PLUGIN] loaded plugin libbx_speaker.so
00000000000i[      ] lt_dlhandle is 0x11d808a30
00000000000i[PLUGIN] loaded plugin libbx_unmapped.so
00000000000i[      ] lt_dlhandle is 0x11d808cb0
00000000000i[PLUGIN] loaded plugin libbx_parallel.so
00000000000i[      ] lt_dlhandle is 0x11d80a080
00000000000i[PLUGIN] loaded plugin libbx_biosdev.so
00000000000i[      ] lt_dlhandle is 0x11d80a3a0
00000000000i[PLUGIN] loaded plugin libbx_iodebug.so
00000000000i[      ] lt_dlhandle is 0x11d80a5f0
00000000000i[PLUGIN] loaded plugin libbx_extfpuirq.so
00000000000i[      ] reading configuration from bochsrc
00000000000i[      ] lt_dlhandle is 0x10c604290
00000000000i[PLUGIN] loaded plugin libbx_textconfig.so
------------------------------
Bochs Configuration: Main Menu
------------------------------

This is the Bochs Configuration Interface, where you can describe the
machine that you want to simulate.  Bochs has already searched for a
configuration file (typically called bochsrc.txt) and loaded it if it
could be found.  When you are satisfied with the configuration, go
ahead and start the simulation.

You can also start bochs with the -q option to skip these menus.

1. Restore factory default configuration
2. Read options from...
3. Edit options
4. Save options to...
5. Restore the Bochs state from...
6. Begin simulation
7. Quit now

Please choose one: [6] 
```
前面好像有两个 `not set`，不用管，它已经选择了默认插件路径和 bochs 安装路径。这里也能看到前面提到的 `$BXSHARE` 的默认选项就是我们想要的 bochs 安装路径。 

随后载入插件…… 然后选择。

这里我们选 `6`，开始模拟。

```
00000000000i[      ] lt_dlhandle is 0x11c6071d0
00000000000i[PLUGIN] loaded plugin libbx_sdl2_gui.so
00000000000i[      ] installing sdl2 module as the Bochs GUI
00000000000i[SDL2  ] maximum host resolution: x=2560 y=1664
00000000000i[      ] using log file bochsout.txt
Bochs internal debugger, type 'help' for help or 'c' to continue
Switching to CPU0
Next at t=0
(0) [0x0000fffffff0] f000:fff0 (unk. ctxt): jmpf 0xf000:e05b          ; ea5be000f0
<bochs:1> c
```

这里启动了调试器，我有这句 `using log file bochsout.txt` 因为没有注释配置文件中 `log` 那句。这里还可以发现我们启动了 sdl2 作为 GUI 库，正合我们指定的 `display_library: sdl2`。这里的调试命令后面再关注，这里我们直接输入 `c` （continue）并回车：

![HelloOS](./pic/chapter01_1.png)

成功辣！

关闭点叉叉就行了……

如果碰到了 `no bootable device` 的错误，有说法是软盘缺失了最后的 `0xaa55` 导致的（要么汇编没写上，要么 `times` 那行写错了数字，要么没把程序载入进软盘……），因为这是一个表示它为启动盘的标志。如果环境相同，一步步走完上面的流程应该是没问题的。





