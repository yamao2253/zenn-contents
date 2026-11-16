1. objdumpとは何をするツール？
objdump は、オブジェクトファイル（.o）や実行ファイル（.elf）の内部構造を調べるツールです。
正式には GNU Binutils に含まれるツールで、今回の arm-none-eabi-objdump はArm向けのものです。

```
trykernel.elf:     file format elf32-littlearm

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
```

- elf32-littlearmは 32ビットArm用のリトルエンディアン形式のELFであることを示す


```
PS C:\home\yamao\repos\src\pico\interface_trykernel\part_2\sect_3> arm-none-eabi-objdump -h trykernel.elf

trykernel.elf:     file format elf32-littlearm  

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 boot2         00000100  10000000  10000000  00001000  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  1 .text         00000ae4  10000100  10000100  00001100  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  2 .ARM.exidx    00000008  10000be4  10000be4  00001be4  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .data         00000000  20000000  10000bec  00002000  2**0
                  CONTENTS, ALLOC, LOAD, DATA
  4 .bss          00000000  20000000  10000bec  00002000  2**0
                  ALLOC
  5 .debug_info   00000ae4  00000000  00000000  00002000  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  6 .debug_abbrev 00000505  00000000  00000000  00002ae4  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  7 .debug_aranges 00000138  00000000  00000000  00002ff0  2**3
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  8 .debug_line   00000901  00000000  00000000  00003128  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
  9 .debug_str    00000a78  00000000  00000000  00003a29  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 10 .comment      00000039  00000000  00000000  000044a1  2**0
                  CONTENTS, READONLY
 11 .ARM.attributes 0000002c  00000000  00000000  000044da  2**0
                  CONTENTS, READONLY
 12 .debug_frame  000002c0  00000000  00000000  00004508  2**2
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 13 .debug_line_str 00000138  00000000  00000000  000047c8  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 14 .debug_loclists 000002dc  00000000  00000000  00004900  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
 15 .debug_rnglists 00000014  00000000  00000000  00004bdc  2**0
                  CONTENTS, READONLY, DEBUGGING, OCTETS
```
