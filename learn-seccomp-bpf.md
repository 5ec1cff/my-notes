# bpf 和 seccomp

最近看了[空大师的博客](https://nullptr.icu/)，发现了一篇[讲解 seccomp 拦截系统调用的文章](https://nullptr.icu/index.php/archives/62/)

[&#x5b;原创&#x5d;分享一个Android通用svc跟踪以及hook方案——Frida-Seccomp-Android安全-看雪论坛-安全社区|安全招聘|bbs.pediy.com](https://bbs.pediy.com/thread-271815.htm#msg_header_h1_2)

[seccomp沙箱机制 & 2019ByteCTF VIP · pollux's Dairy](http://pollux.cc/2019/09/22/seccomp%E6%B2%99%E7%AE%B1%E6%9C%BA%E5%88%B6%20&%202019ByteCTF%20VIP/#0x00-seccomp%E6%B2%99%E7%AE%B1%E6%9C%BA%E5%88%B6)

## 什么是 bpf

[bpf(4)](https://www.freebsd.org/cgi/man.cgi?query=bpf&sektion=4&manpath=FreeBSD+4.7-RELEASE)

过滤器程序是一个指令数组

```
	 A filter program is an array of instructions, with	all branches forwardly
     directed, terminated by a return instruction.  Each instruction performs
     some action on the	pseudo-machine state, which consists of	an accumula-
     tor, index	register, scratch memory store,	and implicit program counter.

     The following structure defines the instruction format:
```

## 指令格式

一个 bpf 指令的结构体：

```
     struct bpf_insn {
	     u_short code;
	     u_char  jt;
	     u_char  jf;
	     u_long k;
     };
```

BPF 指令的格式：操作码(code)+参数(ulong k)，jt 和 jf 为判断值为真/假后跳转的指令相对当前指令的位置（偏移）

```
     The k field is used in different ways by different	instructions, and the
     jt	and jf fields are used as offsets by the branch	instructions.  The op-
     codes are encoded in a semi-hierarchical fashion.	There are eight
     classes of	instructions: BPF_LD, BPF_LDX, BPF_ST, BPF_STX,	BPF_ALU,
     BPF_JMP, BPF_RET, and BPF_MISC.  Various other mode and operator bits are
     or'd into the class to give the actual instructions.  The classes and
     modes are defined in <net/bpf.h>.

     Below are the semantics for each defined bpf instruction.	We use the
     convention	that A is the accumulator, X is	the index register, P[]	packet
     data, and M[] scratch memory store.  P[i:n] gives the data	at byte	offset
     "i" in the	packet,	interpreted as a word (n=4), unsigned halfword (n=2),
     or	unsigned byte (n=1).  M[i] gives the i'th word in the scratch memory
     store, which is only addressed in word units.  The	memory store is	in-
     dexed from	0 to BPF_MEMWORDS - 1.	k, jt, and jf are the corresponding
     fields in the instruction definition.  "len" refers to the	length of the
     packet.
```

### 寄存器

A: 累加器 (4字节)
X: 索引寄存器(index) (4字节)
P[]: 包，P[i:n]  P 从 i 位置开始的 n 个字节(1, 2, 4)
M[]: 暂存内存（？） 以 word (4字节)单位访问 M[i] 第 i 字节
暂存内存(scratch memory): [Scratchpad memory - Wikipedia](https://en.wikipedia.org/wiki/Scratchpad_memory)

pc: 程序寄存器（不可访问）

## 指令一览

### LD: 修改累加器 A
W, H, B: 4, 2, 1 （字，半字，字节）
ABS: 取绝对位置 (k)
IND: 取相对 X 寄存器的位置 (X+k)
LEN: P 的长度
IMM: 直接赋值 k
MEM: 取 M[k]

```
     BPF_LD    These instructions copy a value into the	accumulator.  The type
	       of the source operand is	specified by an	"addressing mode" and
	       can be a	constant (BPF_IMM), packet data	at a fixed offset
	       (BPF_ABS), packet data at a variable offset (BPF_IND), the
	       packet length (BPF_LEN),	or a word in the scratch memory	store
	       (BPF_MEM).  For BPF_IND and BPF_ABS, the	data size must be
	       specified as a word (BPF_W), halfword (BPF_H), or byte (BPF_B).
	       The semantics of	all the	recognized BPF_LD instructions follow.

	       BPF_LD+BPF_W+BPF_ABS  A <- P[k:4]
	       BPF_LD+BPF_H+BPF_ABS  A <- P[k:2]
	       BPF_LD+BPF_B+BPF_ABS  A <- P[k:1]
	       BPF_LD+BPF_W+BPF_IND  A <- P[X+k:4]
	       BPF_LD+BPF_H+BPF_IND  A <- P[X+k:2]
	       BPF_LD+BPF_B+BPF_IND  A <- P[X+k:1]
	       BPF_LD+BPF_W+BPF_LEN  A <- len
	       BPF_LD+BPF_IMM	     A <- k
	       BPF_LD+BPF_MEM	     A <- M[k]
```

### LDX: 修改索引寄存器 X
只有 W (4字节访问)
IMM: 直接赋值
MEM: M[k]
LEN: P 的长度
MSH: 4*(P[k:1] & 0xf) （用于获取 ip 头长度）

```
     BPF_LDX   These instructions load a value into the	index register.	 Note
	       that the	addressing modes are more restrictive than those of
	       the accumulator loads, but they include BPF_MSH,	a hack for ef-
	       ficiently loading the IP	header length.

	       BPF_LDX+BPF_W+BPF_IMM  X	<- k
	       BPF_LDX+BPF_W+BPF_MEM  X	<- M[k]
	       BPF_LDX+BPF_W+BPF_LEN  X	<- len
	       BPF_LDX+BPF_B+BPF_MSH  X	<- 4*(P[k:1]&0xf)
```

### ST: 把累加器 A 存储到暂存内存的第 k 字节

```
     BPF_ST    This instruction	stores the accumulator into the	scratch	mem-
	       ory.  We	do not need an addressing mode since there is only one
	       possibility for the destination.

	       BPF_ST  M[k] <- A
```

### STX: 把索引寄存器 X 存储到暂存内存的第 k 字节

```
     BPF_STX   This instruction	stores the index register in the scratch mem-
	       ory store.

	       BPF_STX	M[k] <-	X
```

### ALU: 运算单元

进行一系列二元运算，结果保存到 A 累加器，且 A 是运算的第一个元。

运算的第二个元：

K: 使用操作数
X: 使用 X 寄存器

支持 ADD, SUB, MUL, DIV, AND, OR, LSH, RSH ，即加减乘除且或左移右移

此外还有取反（单元运算）

```
     BPF_ALU   The alu instructions perform operations between the accumulator
	       and index register or constant, and store the result back in
	       the accumulator.	 For binary operations,	a source mode is re-
	       quired (BPF_K or	BPF_X).

	       BPF_ALU+BPF_ADD+BPF_K  A	<- A + k
	       BPF_ALU+BPF_SUB+BPF_K  A	<- A - k
	       BPF_ALU+BPF_MUL+BPF_K  A	<- A * k
	       BPF_ALU+BPF_DIV+BPF_K  A	<- A / k
	       BPF_ALU+BPF_AND+BPF_K  A	<- A & k
	       BPF_ALU+BPF_OR+BPF_K   A	<- A | k
	       BPF_ALU+BPF_LSH+BPF_K  A	<- A <<	k
	       BPF_ALU+BPF_RSH+BPF_K  A	<- A >>	k
	       BPF_ALU+BPF_ADD+BPF_X  A	<- A + X
	       BPF_ALU+BPF_SUB+BPF_X  A	<- A - X
	       BPF_ALU+BPF_MUL+BPF_X  A	<- A * X
	       BPF_ALU+BPF_DIV+BPF_X  A	<- A / X
	       BPF_ALU+BPF_AND+BPF_X  A	<- A & X
	       BPF_ALU+BPF_OR+BPF_X   A	<- A | X
	       BPF_ALU+BPF_LSH+BPF_X  A	<- A <<	X
	       BPF_ALU+BPF_RSH+BPF_X  A	<- A >>	X
	       BPF_ALU+BPF_NEG	      A	<- -A
```

### JMP：跳转（操作 pc 寄存器）

JA: pc 直接加操作数 k （允许跳转 32 位长）

GT, GE, EQ, SET ：（无符号比较）大于，大于等于，等于，且；结果为 true （非零）就 + jt ，否则 + jf （只能跳转 8 位长）

```
     BPF_JMP   The jump	instructions alter flow	of control.  Conditional jumps
	       compare the accumulator against a constant (BPF_K) or the index
	       register	(BPF_X).  If the result	is true	(or non-zero), the
	       true branch is taken, otherwise the false branch	is taken.
	       Jump offsets are	encoded	in 8 bits so the longest jump is 256
	       instructions.  However, the jump	always (BPF_JA)	opcode uses
	       the 32 bit k field as the offset, allowing arbitrarily distant
	       destinations.  All conditionals use unsigned comparison conven-
	       tions.

	       BPF_JMP+BPF_JA	       pc += k
	       BPF_JMP+BPF_JGT+BPF_K   pc += (A	> k) ? jt : jf
	       BPF_JMP+BPF_JGE+BPF_K   pc += (A	>= k) ?	jt : jf
	       BPF_JMP+BPF_JEQ+BPF_K   pc += (A	== k) ?	jt : jf
	       BPF_JMP+BPF_JSET+BPF_K  pc += (A	& k) ? jt : jf
	       BPF_JMP+BPF_JGT+BPF_X   pc += (A	> X) ? jt : jf
	       BPF_JMP+BPF_JGE+BPF_X   pc += (A	>= X) ?	jt : jf
	       BPF_JMP+BPF_JEQ+BPF_X   pc += (A	== X) ?	jt : jf
	       BPF_JMP+BPF_JSET+BPF_X  pc += (A	& X) ? jt : jf
```

### RET: 终止程序，返回寄存器 A 或操作数 k

```
     BPF_RET   The return instructions terminate the filter program and	spec-
	       ify the amount of packet	to accept (i.e., they return the trun-
	       cation amount).	A return value of zero indicates that the
	       packet should be	ignored.  The return value is either a con-
	       stant (BPF_K) or	the accumulator	(BPF_A).

	       BPF_RET+BPF_A  accept A bytes
	       BPF_RET+BPF_K  accept k bytes
```

### MISC: 杂项

目前可以把一个寄存器的值复制到另一个（A 到 X 或 X 到 A）

     BPF_MISC  The miscellaneous category was created for anything that
	       doesn't fit into	the above classes, and for any new instruc-
	       tions that might	need to	be added.  Currently, these are	the
	       register	transfer instructions that copy	the index register to
	       the accumulator or vice versa.

	       BPF_MISC+BPF_TAX	 X <- A
	       BPF_MISC+BPF_TXA	 A <- X

## 初始化指令的宏

```
     The bpf interface provides	the following macros to	facilitate array ini-
     tializers:	BPF_STMT(opcode, operand) and BPF_JUMP(opcode, operand,
     true_offset, false_offset).
```

# 理解空大师的 bpf 

先看看 32 位部分：

```cpp
// 跳板的地址
auto trampoline = (uintptr_t) Trampoline;
struct sock_filter filter[] = {
    // 读取 packet ——此处是 seccomp_data 结构体——的 nr 字段，也就是系统调用号
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, nr)),
#if defined(__i386__) || defined(__arm__)
    // 如果是，则继续前进，否则跳过两个指令
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, nr, 0, 2),
    // 读 instruction_pointer ，也就是产生系统调用的指令地址
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS, offsetof(struct seccomp_data, instruction_pointer)),
    // 如果是跳板代码，则放行，否则 TRAP 进入信号处理器
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, trampoline, 0, 1),
    // 允许执行（从跳板执行的 syscall 会走到这里）
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    // trap （从非跳板处执行的 syscall 会走到这里）
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_TRAP)
};
```