# flag

Download: [Link to binary](http://pwnable.kr/bin/flag)

First of all lets download the binary, `chmod +x flag`, and check its architecture type and run it.

```bash
$ file flag
flag: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, no section header
$ ./flag
I will malloc() and strcpy the flag there. take it.
```

So as we can see the binary itself doesn’t do much, accepts no input, just ouput exactly what will it be doing.

Lets hop into GDB and see how is it working, you can use Ghidra/IDA as well but lets just stick to GDB as of now, you’ll see in a while why!?!

```bash
gef➤  disas main
No symbol table is loaded.  Use the "file" command.
gef➤  file flag
Reading symbols from flag...
(No debugging symbols found in flag)
gef➤  info functions
All defined functions:
gef➤  info variables
All defined variables:
```

There is nothing in the file, this is very odd! No main function, no _entry function, no variables. **STRANGE**

The only possible explanation of this behavior is, that this binary has been packed with some packer to strip out all the information. We can check which packer has been used using a tool called [DetectItEasy](http://ntinfo.biz/index.html)

```bash
$ bash diec.sh ./flag
ELF64: packer: UPX(3.08)[NRV,brute]
```

As suspected, this binary has been packed using [UPX](https://upx.github.io/). We can use the UPX tools to unpack this binary and then see exactly what does it do.

```bash
$ ./upx -d ./flag -o ./flag-unpacked-upx
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2018
UPX 3.95        Markus Oberhumer, Laszlo Molnar & John Reiser   Aug 26th 2018

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
    883745 <-    335288   37.94%   linux/amd64   flag-unpacked-upx

Unpacked 1 file.

$ file ./flag-unpacked-upx
flag-unpacked-upx: ELF 64-bit LSB executable, x86-64, version 1 (GNU/Linux), statically linked, for GNU/Linux 2.6.24, BuildID[sha1]=96ec4cc272aeb383bd9ed26c0d4ac0eb5db41b16, not stripped
```

As you can see, now we have more information about the binary and it is not stripped and we can not use GDB to analyze it’s behavior.

```assembly
gef➤  disas main
Dump of assembler code for function main:
   0x0000000000401164 <+0>:	push   rbp
   0x0000000000401165 <+1>:	mov    rbp,rsp
   0x0000000000401168 <+4>:	sub    rsp,0x10
   0x000000000040116c <+8>:	mov    edi,0x496658
   0x0000000000401171 <+13>:	call   0x402080 <puts>
   0x0000000000401176 <+18>:	mov    edi,0x64
   0x000000000040117b <+23>:	call   0x4099d0 <malloc>
   0x0000000000401180 <+28>:	mov    QWORD PTR [rbp-0x8],rax
   0x0000000000401184 <+32>:	mov    rdx,QWORD PTR [rip+0x2c0ee5] # 0x6c2070 <flag>
   0x000000000040118b <+39>:	mov    rax,QWORD PTR [rbp-0x8]
   0x000000000040118f <+43>:	mov    rsi,rdx
   0x0000000000401192 <+46>:	mov    rdi,rax
   0x0000000000401195 <+49>:	call   0x400320
   0x000000000040119a <+54>:	mov    eax,0x0
   0x000000000040119f <+59>:	leave  
   0x00000000004011a0 <+60>:	ret    

```

As we can see in the disassembly, at `0x0000000000401184`, some value is being moved into the `$rdx register` and is marked with a comment of flag as well. Lets put a breakpoint at `*main+39` so that we can inspect `$rdx register` and get the value of the flag.

```assembly
gef➤  b *main+39
Breakpoint 1 at 0x40118b
...
gef➤  x/s $rdx
0x496628:	"UPX...? sounds like a delivery service :)" # FLAG
```

We got our flag. However, we used few tools like GDB and DetectItEasy to understand the binary better, these could have been easily avoided and simple `strings` could have been used to solve this challenge.

```bash
$ strings flag | grep UPX
UPX!
$Info: This file is packed with the UPX executable packer http://upx.sf.net $
$Id: UPX 3.08 Copyright (C) 1996-2011 the UPX Team. All Rights Reserved. $
UPX!
UPX!
... Unpack the binary ...
$ string flag-unpacked-upx | grep UPX
UPX...? sounds like a delivery service :)
```

So, there were multiple ways to solve this binary, however, for the `strings` method you’d have to go through the strings output manually and search for interesting strings to understand that it has been packed using UPX and then again go through the strings output of the unpacked binary to get the value of the flag.