**Project Address**

https://github.com/jkriege2/TinyTIFF

**Security Issue Report**

A global-buffer-overflow issue was discovered in TinyTIFF in tinytiffreader.c file. The flow allows an attacker to cause a denial of service (abort) via a crafted file.

**OS information**

```
ubuntu@ubuntu:~/Documents/TinyTIFF/src$ uname -a
Linux ubuntu 5.15.0-58-generic #64~20.04.1-Ubuntu SMP Fri Jan 6 16:42:31 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```

**Summary**

AddressSanitizer: global-buffer-overflow (/home/ubuntu/Desktop/TinyTIFF/src/asan_tinytiffreader+0x4969c6) in __asan_memcpy

**Problem Code location**

```
    #0 0x4969c6 in __asan_memcpy (/home/ubuntu/Desktop/TinyTIFF/src/asan_tinytiffreader+0x4969c6)
    #1 0x4cdff4 in TinyTIFFReader_readNextFrame /home/ubuntu/Desktop/TinyTIFF/src/tinytiffreader.c
    #2 0x4cb3e9 in TinyTIFFReader_open /home/ubuntu/Desktop/TinyTIFF/src/tinytiffreader.c:921:9
    #3 0x4d10ea in main /home/ubuntu/Desktop/TinyTIFF/src/tinytiffreader.c:1058:10
    #4 0x7ffff7c4a082 in __libc_start_main /build/glibc-SzIz7B/glibc-2.31/csu/../csu/libc-start.c:308:16
    #5 0x41c3bd in _start (/home/ubuntu/Desktop/TinyTIFF/src/asan_tinytiffreader+0x41c3bd)
```

[Poc file: id8](https://github.com/10cksYiqiyinHangzhouTechnology/Security-Issue-Report-of-TinyTIFF/blob/main/id8)

[asan_tinytiffreader](https://github.com/10cksYiqiyinHangzhouTechnology/Security-Issue-Report-of-TinyTIFF/blob/main/asan_tinytiffreader)

**compile the test case in the source**

```bash
cd TinyTIFF/
cmake .
make -j4
cd src/
```
modified the tinytiffreader.c file, add harness

```c
int main(int argc, char *argv[]){
       TinyTIFFReaderFile* tiffr=NULL;
   tiffr=TinyTIFFReader_open(argv[1]); 
   if (!tiffr) { 
	   printf("ERROR reading (not existent, not accessible or no TIFF file)\n");
   } else { 
        const uint32_t width=TinyTIFFReader_getWidth(tiffr); 
        const uint32_t height=TinyTIFFReader_getHeight(tiffr);
        const uint16_t bitspersample=TinyTIFFReader_getBitsPerSample(tiffr, 0);		
        uint8_t* image=(uint8_t*)calloc(width*height, bitspersample/8);  
        TinyTIFFReader_getSampleData(tiffr, image, 0); 

	printf("%ld\n",width);
        printf("%ld\n",height);
        printf("%u\n",bitspersample);

							///////////////////////////////////////////////////////////////////
							// HERE WE CAN DO SOMETHING WITH THE SAMPLE FROM THE IMAGE 
							// IN image (ROW-MAJOR!)
							// Note: That you may have to typecast the array image to the
							// datatype used in the TIFF-file. You can get the size of each
							// sample in bits by calling TinyTIFFReader_getBitsPerSample() and
							// the datatype by calling TinyTIFFReader_getSampleFormat().
							///////////////////////////////////////////////////////////////////
                
        free(image); 
    } 
    TinyTIFFReader_close(tiffr); 

}
```
then,complie the tinytiffreader.c

```bash
gcc -o tinytiffreader tinytiffreader.c
```

test with poc

```bash
./tinytiffreader id8
```
this program will trigger  global-buffer-overflow crash. The asan complie program is asan_tinytiffreader.

**ASAN Report:**

```bash
ubuntu@ubuntu:~/Desktop/TinyTIFF/src$ ./asan_tinytiffreader out/default/crashes/id\:000008\,sig\:11\,src\:000016+000007\,time\:306221\,execs\:34950\,op\:splice\,rep\:16 
=================================================================
==3817392==ERROR: AddressSanitizer: global-buffer-overflow on address 0x0000004e82c3 at pc 0x0000004969c7 bp 0x7fffffffd890 sp 0x7fffffffd058
READ of size 2082441480 at 0x0000004e82c3 thread T0
    #0 0x4969c6 in __asan_memcpy (/home/ubuntu/Desktop/TinyTIFF/src/asan_tinytiffreader+0x4969c6)
    #1 0x4cdff4 in TinyTIFFReader_readNextFrame /home/ubuntu/Desktop/TinyTIFF/src/tinytiffreader.c
    #2 0x4cb3e9 in TinyTIFFReader_open /home/ubuntu/Desktop/TinyTIFF/src/tinytiffreader.c:921:9
    #3 0x4d10ea in main /home/ubuntu/Desktop/TinyTIFF/src/tinytiffreader.c:1058:10
    #4 0x7ffff7c4a082 in __libc_start_main /build/glibc-SzIz7B/glibc-2.31/csu/../csu/libc-start.c:308:16
    #5 0x41c3bd in _start (/home/ubuntu/Desktop/TinyTIFF/src/asan_tinytiffreader+0x41c3bd)

0x0000004e82c3 is located 61 bytes to the left of global variable '<string literal>' defined in 'tinytiffreader.c:653:13' (0x4e8300) of size 62
0x0000004e82c3 is located 0 bytes to the right of global variable '<string literal>' defined in 'tinytiffreader.c:214:32' (0x4e82c0) of size 3
  '<string literal>' is ascii string 'rb'
SUMMARY: AddressSanitizer: global-buffer-overflow (/home/ubuntu/Desktop/TinyTIFF/src/asan_tinytiffreader+0x4969c6) in __asan_memcpy
Shadow bytes around the buggy address:
  0x000080095000: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x000080095010: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x000080095020: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x000080095030: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x000080095040: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x000080095050: 00 00 00 00 00 00 00 00[03]f9 f9 f9 f9 f9 f9 f9
  0x000080095060: 00 00 00 00 00 00 00 06 f9 f9 f9 f9 00 00 00 00
  0x000080095070: 00 00 f9 f9 f9 f9 f9 f9 00 00 00 00 00 00 00 07
  0x000080095080: f9 f9 f9 f9 00 00 00 00 00 00 00 03 f9 f9 f9 f9
  0x000080095090: 00 00 00 00 00 03 f9 f9 f9 f9 f9 f9 00 00 00 00
  0x0000800950a0: 00 00 00 02 f9 f9 f9 f9 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
  Shadow gap:              cc
==3817392==ABORTING
```
**use gdb debug this crah**

```bash
ubuntu@ubuntu:~/Documents/TinyTIFF/src$ gdb --args ./tinytiffreader id8
GNU gdb (Ubuntu 9.2-0ubuntu1~20.04.1) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from ./tinytiffreader...
(No debugging symbols found in ./tinytiffreader)
gdb-peda$ r id8
Starting program: /home/ubuntu/Documents/TinyTIFF/src/tinytiffreader id8

Program received signal SIGSEGV, Segmentation fault.
[----------------------------------registers-----------------------------------]
RAX: 0x7ffdfdd09010 --> 0x0 
RBX: 0x55555555b2a0 --> 0x55555555b720 --> 0xfbad2488 
RCX: 0x0 
RDX: 0x1fa0b8878 
RSI: 0x0 
RDI: 0x7ffdfdd09010 --> 0x0 
RBP: 0x7fffffffdcd0 --> 0x7fffffffdda0 --> 0x7fffffffdef0 --> 0x7fffffffdf30 --> 0x0 
RSP: 0x7fffffffdca8 --> 0x555555555663 (<TinyTIFFReader_memcpy_s+51>:	mov    eax,0x0)
RIP: 0x7ffff7f4d8f5 (<__memmove_avx_unaligned_erms+533>:	vmovdqu ymm4,YMMWORD PTR [rsi])
R8 : 0x7ffdfdd09010 --> 0x0 
R9 : 0x0 
R10: 0x22 ('"')
R11: 0x246 
R12: 0x555555555240 (<_start>:	endbr64)
R13: 0x7fffffffe020 --> 0x2 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x7ffff7f4d8ec <__memmove_avx_unaligned_erms+524>:	vmovdqu YMMWORD PTR [r11],ymm4
   0x7ffff7f4d8f1 <__memmove_avx_unaligned_erms+529>:	vzeroupper 
   0x7ffff7f4d8f4 <__memmove_avx_unaligned_erms+532>:	ret    
=> 0x7ffff7f4d8f5 <__memmove_avx_unaligned_erms+533>:	vmovdqu ymm4,YMMWORD PTR [rsi]
   0x7ffff7f4d8f9 <__memmove_avx_unaligned_erms+537>:	vmovdqu ymm5,YMMWORD PTR [rsi+0x20]
   0x7ffff7f4d8fe <__memmove_avx_unaligned_erms+542>:	vmovdqu ymm6,YMMWORD PTR [rsi+0x40]
   0x7ffff7f4d903 <__memmove_avx_unaligned_erms+547>:	vmovdqu ymm7,YMMWORD PTR [rsi+0x60]
   0x7ffff7f4d908 <__memmove_avx_unaligned_erms+552>:	vmovdqu ymm8,YMMWORD PTR [rsi+rdx*1-0x20]
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffdca8 --> 0x555555555663 (<TinyTIFFReader_memcpy_s+51>:	mov    eax,0x0)
0008| 0x7fffffffdcb0 --> 0x1fa0b8878 
0016| 0x7fffffffdcb8 --> 0x0 
0024| 0x7fffffffdcc0 --> 0x1fa0b8878 
0032| 0x7fffffffdcc8 --> 0x7ffdfdd09010 --> 0x0 
0040| 0x7fffffffdcd0 --> 0x7fffffffdda0 --> 0x7fffffffdef0 --> 0x7fffffffdf30 --> 0x0 
0048| 0x7fffffffdcd8 --> 0x5555555563fb (<TinyTIFFReader_readNextFrame+886>:	jmp    0x555555556683 <TinyTIFFReader_readNextFrame+1534>)
0056| 0x7fffffffdce0 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
__memmove_avx_unaligned_erms () at ../sysdeps/x86_64/multiarch/memmove-vec-unaligned-erms.S:436
436	../sysdeps/x86_64/multiarch/memmove-vec-unaligned-erms.S: No such file or directory.
```
you can see "../sysdeps/x86_64/multiarch/memmove-vec-unaligned-erms.S: No such file or directory.", I think this problem is caused by abnormal references to Pointers, See this [link](https://stackoverflow.com/questions/69875165/c-segmentation-failed-read-in-csv-file-memmove-vec-unaligned-erms-s-no-such).This one bug I tested on multiple systems crashed.
