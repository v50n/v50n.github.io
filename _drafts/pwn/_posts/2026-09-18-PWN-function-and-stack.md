---
title: "Pwn - stack and function"
categories:
  - Pwn
tags:
  - Pwn
layout: post
---

## Function and stack

Khi học tới ROP stack pivot, bố mày thấy mông lung, đéo hiểu stack hoạt động thế nào khi có 2 function gọi nhau nên bố mày tự tìm hiểu rồi viết ra blog này. 

Giả sử  main() --> test() thì stack sẽ hoạt động như thế nào: 

```md
Main()
  | 
  |
  v
Test()  
```

Viết thử 1 bài bằng C rồi debug với gdb 
```c
#include<stdio.h>

void test(){
    int a;
    a = 5;
    printf("here is a test");
}

int main(){
    char a;
    a = 'h';
    printf("hello. Here we go");    
    test();
    return 0;
}
```

chỉ cần quan tâm khi function main call test, và sau khi xong test, quay lại main. 

## Main function before call test

Main function
```
   0x000055555555515f <+0>:	push   rbp
   0x0000555555555160 <+1>:	mov    rbp,rsp
   0x0000555555555163 <+4>:	sub    rsp,0x10
   0x0000555555555167 <+8>:	mov    BYTE PTR [rbp-0x1],0x68
   0x000055555555516b <+12>:	lea    rax,[rip+0xea1]        # 0x555555556013
   0x0000555555555172 <+19>:	mov    rdi,rax
   0x0000555555555175 <+22>:	mov    eax,0x0
   0x000055555555517a <+27>:	call   0x555555555030 <printf@plt>
   0x000055555555517f <+32>:	call   0x555555555139 <test>
   0x0000555555555184 <+37>:	mov    eax,0x0
   0x0000555555555189 <+42>:	leave
   0x000055555555518a <+43>:	ret
```

```
Stack : 
0x00007fffffffdb40│+0x0000: 0x0000000000000000	 ← $rsp
0x00007fffffffdb48│+0x0008: 0x6800000000000000
0x00007fffffffdb50│+0x0010: 0x00007fffffffdc78   ← $rbp
```

## Call test()


Call làm 2 việc quan trọng 
- rsp = rsp - 0x8
- [rsp] = rip (main+37) 

call test() --> rsp - 0x8 

```diff
+ 0x00007fffffffdb38│+0x0000: 0x0000555555555184   ← $rsp
0x00007fffffffdb40│+0x0008: 0x0000000000000000
0x00007fffffffdb48│+0x0010: 0x6800000000000000
0x00007fffffffdb50│+0x0018: 0x00007fffffffdc78   ← $rbp
```

## 3 instructions đầu của test() 

```assembly
   0x0000555555555139 <+0>:	push   rbp
   0x000055555555513a <+1>:	mov    rbp,rsp
   0x000055555555513d <+4>:	sub    rsp,0x10
```
push rbp = push value rbp into a stack (0x00007fffffffdb50)
```diff
+ 0x00007fffffffdb30│+0x0000: 0x00007fffffffdb50   ← $rsp
0x00007fffffffdb38│+0x0000: 0x0000555555555184   
0x00007fffffffdb40│+0x0008: 0x0000000000000000
0x00007fffffffdb48│+0x0010: 0x6800000000000000
0x00007fffffffdb50│+0x0018: 0x00007fffffffdc78   ← $rbp
```
mov rbp,rsp
Stack k thay đổi.
Register rbp thay đổi
```diff
0x00007fffffffdb30│+0x0000: 0x00007fffffffdb50   ← $rsp, $rbp
0x00007fffffffdb38│+0x0000: 0x0000555555555184   
0x00007fffffffdb40│+0x0008: 0x0000000000000000
0x00007fffffffdb48│+0x0010: 0x6800000000000000
0x00007fffffffdb50│+0x0018: 0x00007fffffffdc78 
```

sub rsp,0x10 
```diff
+ 0x00007fffffffdb20│+0x0000: 0x0000000000000000	 ← $rsp
+ 0x00007fffffffdb28│+0x0008: 0x00007ffff7ffe2e0  
0x00007fffffffdb30│+0x0010: 0x00007fffffffdb50   ← $rbp
0x00007fffffffdb38│+0x0000: 0x0000555555555184   
0x00007fffffffdb40│+0x0008: 0x0000000000000000
0x00007fffffffdb48│+0x0010: 0x6800000000000000
0x00007fffffffdb50│+0x0018: 0x00007fffffffdc78 
```

Sau khi function test() chạy xong, 2 instruction cuối sẽ vào việc
```assembly
leave
ret
```
## leave
leave làm 2 việc. 
- Việc 1 : mov rsp,rbp 
- Việc 2 : pop rbp 

Việc 1 : mov rsp,rbp : 

```diff
- 0x00007fffffffdb20│+0x0000: 0x0000000000000000	 ← y hệt addr trên
- 0x00007fffffffdb28│+0x0008: 0x00007ffff7ffe2e0   ← Đéo liên quan tới stack vì rsp của test() đã được mov. 
0x00007fffffffdb30│+0x0010: 0x00007fffffffdb50   ← $rsp, $rbp
0x00007fffffffdb38│+0x0000: 0x0000555555555184   
0x00007fffffffdb40│+0x0008: 0x0000000000000000
0x00007fffffffdb48│+0x0010: 0x6800000000000000
0x00007fffffffdb50│+0x0018: 0x00007fffffffdc78 
```
Việc 2 : pop rbp 
```diff
Lấy ra lưu vào rbp. vậy giờ rbp = 0x00007fffffffdb50. rsp sẽ = 0x00007fffffffdb38 vì đã pop giá trị dưới cùng

- 0x00007fffffffdb30│+0x0010: 0x00007fffffffdb50   ← $rsp, $rbp
0x00007fffffffdb38│+0x0000: 0x0000555555555184   ← $rsp
0x00007fffffffdb40│+0x0008: 0x0000000000000000
0x00007fffffffdb48│+0x0010: 0x6800000000000000
0x00007fffffffdb50│+0x0018: 0x00007fffffffdc78   ← $rbp
```


## Ret instruction


Ret làm 2 việc quan trọng : 
```
Set rip = [rsp] (main+37)
rsp =  rsp - 0x8 . Vậy là sẽ tìm được đến stack cũ
```

## After ret 



Stack quay trở lại như cũ
```diff
Pops address from stack and jumps back --> why we need to overwrite RIP

- 0x00007fffffffdb38│+0x0000: 0x0000555555555184   ← $rsp
0x00007fffffffdb40│+0x0008: 0x0000000000000000   ← $rsp
0x00007fffffffdb48│+0x0010: 0x6800000000000000
0x00007fffffffdb50│+0x0018: 0x00007fffffffdc78   ← $rbp
```

Quan trọng nhất là function quay trở lại với addresse 0x0000555555555184 (main+37)

```
gef➤  disas
Dump of assembler code for function main:
   0x000055555555515f <+0>:	push   rbp
   0x0000555555555160 <+1>:	mov    rbp,rsp
   0x0000555555555163 <+4>:	sub    rsp,0x10
   0x0000555555555167 <+8>:	mov    BYTE PTR [rbp-0x1],0x68
   0x000055555555516b <+12>:	lea    rax,[rip+0xea1]        # 0x555555556013
   0x0000555555555172 <+19>:	mov    rdi,rax
   0x0000555555555175 <+22>:	mov    eax,0x0
   0x000055555555517a <+27>:	call   0x555555555030 <printf@plt>
   0x000055555555517f <+32>:	call   0x555555555139 <test>
=> 0x0000555555555184 <+37>:	mov    eax,0x0
   0x0000555555555189 <+42>:	leave
   0x000055555555518a <+43>:	ret
End of assembler dump.

```

Vậy thôi, sau khi rời khỏi function test, tất cả lại quay trở lại với stack ban đầu :) 

Instruction ret sau này sẽ rất cần cho ROP pivot :) Mỗi lần nó ret, nhét vào mồm nó 1 addr của 1 gadget. Nó sẽ tự nhảy vào, làm dăm ba thứ, khi nó nhảy ra, lại nhét vào mồm nó 1 gadget khác ! 