# LAB - Lazy Binding (GOT/PLT 實驗)

tags: `程式設計師的自我修養` `PLT` `GOT` `PIC`

:::info
- [程式設計師的自我修養 - Notes](https://hackmd.io/@kenjin/coder_cultivation)
- 參考 "程式設計師的自我修養" PIC 章節範例程式
:::

## System Version
    Linux version 4.13.0-41-generic (buildd@lgw01-amd64-028)
    (gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.9))
    #46~16.04.1-Ubuntu SMP Thu May 3 10:06:43 UTC 2018  

## Code Example
main.c
```c=
#include <stdio.h>
#include "lib_a.h"

int main(int argc, char* argv[])
{
    foo();
}
```

lib_a.c
```c=
#include <stdlib.h>

static int a;
extern int b;
extern void ext();

void bar(void)
{
    a = 1;
    b = 2;
}

void foo()
{
    bar();
    ext();
}
```

lib_a.h
```c=
#ifndef LIB_A_H
#define LIB_A_H

void bar(void);
void foo(void);
#endif
```

lib_b.c
```c=
#include <stdio.h>

int b;
void ext (void)
{
    b = 0;
    printf("%s: Call from file %s\n", __func__, __FILE__);
}
```
## Compilation

    $ gcc -g -shared -fPIC lib_a.c  -o liba.so
    $ gcc -g -shared -fPIC lib_b.c  -o libb.so   
    $ gcc -g main.c liba.so libb.so
    $ gcc -g main.c -la -lb -L.
    $ [sudo] cp liba.so libb.so /usr/lib 
    $ ldd a.out
        linux-vdso.so.1 =>  (0x00007ffce28ec000)
        liba.so => /usr/lib/liba.so (0x00007f3859a5d000)
        libb.so => /usr/lib/libb.so (0x00007f385985b000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f3859491000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f3859c5f000)

## ELF Observation
觀察 main(a.out): `readelf -r a.out` 觀察 relocation(-r) section 的各個 Symbol
> '.rel.dyn' 表示 Data 欄位, 而 '.rel.plt' 表示的則是 Function 欄位[color=red]

- line 3: `__gmon_start__`: 查看效能用(編譯時加上 -pg 此 symbol 就有作用)
- line 7: `__libc_start_main`: 這是用來執行 CRT
- line 8: `foo`: 這是 lib_a.o 的 code

```script=
Relocation section '.rela.dyn' at offset 0x4e8 contains 1 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000600ff8  000300000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0

Relocation section '.rela.plt' at offset 0x500 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000601018  000200000007 R_X86_64_JUMP_SLO 0000000000000000 __libc_start_main@GLIBC_2.2.5 + 0
000000601020  000400000007 R_X86_64_JUMP_SLO 0000000000000000 foo + 0
```


觀察 lib_a.so:
- line 15: **bar** 是屬於 "Inner-module data access" type , 居然也會有 PLT 資訊
- line 16: **ext** 是 lib_b 的 code
```script=
Relocation section '.rela.dyn' at offset 0x4a8 contains 9 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000200df8  000000000008 R_X86_64_RELATIVE                    6e0
000000200e00  000000000008 R_X86_64_RELATIVE                    6a0
000000201028  000000000008 R_X86_64_RELATIVE                    201028
000000200fd0  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000200fd8  000300000006 R_X86_64_GLOB_DAT 0000000000000000 b + 0
000000200fe0  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000200fe8  000600000006 R_X86_64_GLOB_DAT 0000000000000000 _Jv_RegisterClasses + 0
000000200ff0  000700000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000200ff8  000800000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x580 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000201018  000a00000007 R_X86_64_JUMP_SLO 0000000000000710 bar + 0
000000201020  000500000007 R_X86_64_JUMP_SLO 0000000000000000 ext + 0
```

觀察 lib_b.so: `readelf -r lib_b.so`
- line 7: **b** 
- line 15: **printf**
```script=
Relocation section '.rela.dyn' at offset 0x488 contains 9 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000200df8  000000000008 R_X86_64_RELATIVE                    6a0
000000200e00  000000000008 R_X86_64_RELATIVE                    660
000000201020  000000000008 R_X86_64_RELATIVE                    201020
000000200fd0  000200000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_deregisterTMClone + 0
000000200fd8  000c00000006 R_X86_64_GLOB_DAT 000000000020102c b + 0
000000200fe0  000400000006 R_X86_64_GLOB_DAT 0000000000000000 __gmon_start__ + 0
000000200fe8  000500000006 R_X86_64_GLOB_DAT 0000000000000000 _Jv_RegisterClasses + 0
000000200ff0  000600000006 R_X86_64_GLOB_DAT 0000000000000000 _ITM_registerTMCloneTa + 0
000000200ff8  000700000006 R_X86_64_GLOB_DAT 0000000000000000 __cxa_finalize@GLIBC_2.2.5 + 0

Relocation section '.rela.plt' at offset 0x560 contains 1 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000201018  000300000007 R_X86_64_JUMP_SLO 0000000000000000 printf@GLIBC_2.2.5 + 0
```

預先觀察 a.out (GOT section): `objdump -SsdD a.out`
- 留意 Endian
- 留意 `0x601010`, `0x601020` 位址的值變化

```script=
Contents of section .got:
 600ff8 00000000 00000000                    ........        
Contents of section .got.plt:
 601000 080e6000 00000000 00000000 00000000  ..`.............
 601010 00000000 00000000 66054000 00000000  ........f.@.....
 601020 76054000 00000000                    v.@.....       
...
...
```

預先觀察 `foo` Symbol, 表示位於 .plt (lazy binding)

    (gdb) info symbol foo
    foo@plt in section .plt


## GDB Observation - "foo" Symbol
### Step 1: 執行 `gdb a.out`, 開始解析程式流程
![](https://i.imgur.com/kJB3iCm.jpg)

開始執行到 Line 7: call `foo()` 會直接跳至 `0x400570` 

```script=
Dump of assembler code for function main:
   0x0000000000400686 <+0>:     push   %rbp
   0x0000000000400687 <+1>:     mov    %rsp,%rbp
   0x000000000040068a <+4>:     sub    $0x10,%rsp
   0x000000000040068e <+8>:     mov    %edi,-0x4(%rbp)
   0x0000000000400691 <+11>:    mov    %rsi,-0x10(%rbp)
   0x0000000000400695 <+15>:    callq  0x400570 <foo@plt>
   0x000000000040069a <+20>:    mov    $0x0,%eax
   0x000000000040069f <+25>:    leaveq
   0x00000000004006a0 <+26>:    retq

```
![](https://i.imgur.com/rCatNN8.jpg)

觀察 `0x400570`(也就是 `<foo@plt>` 位址) 的 machine instruction(x/8w==i== `0x400570`), 會要求跳至 `0x601020` 這個處於 GOT 的位址
> 這也是 PLT 目的, 執行期從 PLT 間接透過 GOT 索引

    (gdb) x/8wi 0x400570
       0x400570 <foo@plt>:  jmpq   *0x200aaa(%rip)        # 0x601020
       0x400576 <foo@plt+6>:        pushq  $0x1
       0x40057b <foo@plt+11>:       jmpq   0x400550
       0x400580:    jmpq   *0x200a72(%rip)        # 0x600ff8
       0x400586:    xchg   %ax,%ax
       0x400588:    Cannot access memory at address 0x400588

觀察 `0x601020` 位址的值, 而知道是要執行 `0x00400576` 位址的指令

    (gdb) x/8wx 0x601020
    0x601020:       0x00400576      0x00000000      0x00000000      0x00000000
    0x601030:       0x00000000      0x00000000      0x00000000      0x00000000

#### "0x400576 pushq $0xXX"
再看 `0x00400576` 是執行怎樣的 machine instruction. 原來意思是因為第一次 call `foo()` 還沒完整經過 PLT -> GOT 進行 Symbol Resolution, 因此**要求我們 `jmpq 0x400550` 跳至 `.plt` 的 codes 並執行查找**

#### 複習 **.plt**
這個 section 包含的 cods 用來:
- Call ld(linker) 解析某個外部函數的地址並填入 `.got.plt` 中, 然後跳轉到該函數, 或者	
- 直接在 `.got.plt` 中查找並跳轉到對應外部函數(如果已填過更新 GOT)

>
    (gdb) x/8wi 0x00400576
       0x400576 <foo@plt+6>:        pushq  $0x1          <-----
       0x40057b <foo@plt+11>:       jmpq   0x400550      <-----
       0x400580:    jmpq   *0x200a72(%rip)        # 0x600ff8
       0x400586:    xchg   %ax,%ax
       0x400588:    Cannot access memory at address 0x400588

#### "0x400576" 補充
這個 `.plt`(屬於 .text section) 的 push 動作並非無關緊要的! 如 `pushq $0x0` 或 `pushq $0x1` 其實就是 push `__libc_start_main` / `foo` 兩個的 Symbol ID 做為後續 ld(dynamic linker) 對 GOT 定位用
- 請回憶前述 `readelf -r a.out` 會有這兩個 symbol
>
    0000000000400560 <__libc_start_main@plt>:
      400560:	ff 25 b2 0a 20 00    	jmpq   *0x200ab2(%rip)        # 601018 <_GLOBAL_OFFSET_TABLE_+0x18>
      400566:	68 00 00 00 00       	pushq  $0x0
      40056b:	e9 e0 ff ff ff       	jmpq   400550 <_init+0x20>

    0000000000400570 <foo@plt>:
      400570:	ff 25 aa 0a 20 00    	jmpq   *0x200aaa(%rip)        # 601020 <_GLOBAL_OFFSET_TABLE_+0x20>
      400576:	68 01 00 00 00       	pushq  $0x1
      40057b:	e9 d0 ff ff ff       	jmpq   400550 <_init+0x20>

### Step 2: 設定 break points, 執行 PLT codes
![](https://i.imgur.com/g47erdC.jpg)

觀察 `0x400550` 

    (gdb) x/4wi 0x400550
       0x400550:    pushq  0x200ab2(%rip)        # 0x601008
       0x400556:    jmpq   *0x200ab4(%rip)        # 0x601010
       0x40055c:    nopl   0x0(%rax)
       0x400560 <__libc_start_main@plt>:    jmpq   *0x200ab2(%rip)        # 0x601018


    Breakpoint 3 at 0x400550
    (gdb) r
    Starting program: /home/moxaiw/test/linking/Shared_Library_GOT_PLT_example/a.out

    Breakpoint 3, 0x0000000000400550 in ?? ()

    (gdb) x/12i 0x400550
       0x400550:    pushq  0x200ab2(%rip)        # 0x601008    <-----
       0x400556:    jmpq   *0x200ab4(%rip)        # 0x601010   <-----
       0x40055c:    nopl   0x0(%rax)
       0x400560 <__libc_start_main@plt>:    jmpq   *0x200ab2(%rip)        # 0x601018
       0x400566 <__libc_start_main@plt+6>:  pushq  $0x0
       0x40056b <__libc_start_main@plt+11>: jmpq   0x400550
       0x400570 <foo@plt>:  jmpq   *0x200aaa(%rip)        # 0x601020
       0x400576 <foo@plt+6>:        pushq  $0x1
       0x40057b <foo@plt+11>:       jmpq   0x400550
       0x400580:    jmpq   *0x200a72(%rip)        # 0x600ff8
       0x400586:    xchg   %ax,%ax
       0x400588:    add    %al,(%rax)
    
    (gdb) x/12wx 0x601000
    0x601000:       0x00600e08      0x00000000      0xf7ffe168      0x00007fff
    0x601010:       0xf7dee870      0x00007fff      0x00400566      0x00000000
    0x601020:       0x00400576      0x00000000      0x00000000      0x00000000

### Step 3: ld 進行 Symbol Resolution
![](https://i.imgur.com/6Wlv357.jpg)

#### 忽略的點 (0x601010)
- 有注意一開始看 GOT 變化的話, 會發現此位址的值從 **0x0** 變為 **0xf7dee870**
- 這是指向 ld 的位址, 表示此位址的值再開始 run a.out 時會更新(待證明: 處理的時間點!?)

#### 最後      
Linker 做完 Symbol Resolution 後, 觀察 foo GOT `0x601020` => ==`0xf7bd572e`== 
> 有設定 hardware watchpoint 應該會立即跳出訊息

    (gdb) x/12wx 0x601000
    0x601000:       0x00600e08      0x00000000      0xf7ffe168      0x00007fff
    0x601010:       0xf7dee870      0x00007fff      0xf7629740      0x00007fff
    0x601020:       0xf7bd572e      0x00007fff      0x00000000      0x00000000

再觀察 `foo` Symbol (已變為 **.text of /usr/lib/liba.so** )

    ext: Call from file lib_b.c
    [Inferior 1 (process 20396) exited normally]
    (gdb) info symbol foo
    foo in section .text of /usr/lib/liba.so
        
若之後有再 call `foo()` 就會從 PLT -> GOT 而直接往 ==`0xf7bd572e`== 走囉