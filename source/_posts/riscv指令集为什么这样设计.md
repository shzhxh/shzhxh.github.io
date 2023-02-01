---
layout: post
title:  "riscv指令集为什么这样设计"
date:   2021-11-21 22:02:00 +0800
categories: COD
---

机器语言是一个折中，一方面是人类想要的自然语言，一方面是机器能提供的位运算。

R类型的指令说的是操作对象只有寄存器。

I类型的指令说的是操作对象包含12位的立即数。

S类型的指令说的是这个指令会改变内存地址(Store)。

U类型的指令说的是操作对象包含高位(Upper)立即数。(高20位)

J类型的指令说的是要跳转(jump)到一个地址，它只是把U类型的imm重新组合了一遍，所以是U类型的变体。

B类型的指令说的是要执行分支(branch)操作，它只是把S类型的imm重新组合了一遍，所以是S类型的变体。

RISCV的操作码应该是扩展操作码。opcode[+func3[+func7]]。opcode和func7是7位，提供128种可能。func3是3位，提供8种可能。

通用寄存器有32个，所以单个寄存器在指令里占5位。

opcode排在最前，因为它是最重要的，说明了指令的作用。rd排在opcode后面，因为它也特别重要，一个指令起作用，它要么修改寄存器，要么修改内存，否则CPU就是运行了个寂寞。func3的重要性类似于opcode，所以它排在rd之后。关于剩下的rs1,rs2,func7的排列，rs1作为次常用寄存器要放在一堆，rs2和func7有可能作为imm出现所以要放在另一堆。

#### I指令集

##### 整数计算指令

I类型的指令9个，opcode为OP-IMM，这9个指令主要通过func3区分。

R类型的指令10个，opcode为OP，这10个指令主要通过func3区分，辅助以func7。

U类型的指令2个，opcode为LUI和AUIPC。

##### 控制转移指令

J类型的指令1个，opcode为JAL。

I类型的指令1个，opcode为JALR，func3为0。

B类型的指令6个，opcode为BRANCH，这6个指令通过func3区分。

##### LOAD与STORE指令

I类型的指令5个，opcode为LOAD，这5个指令通过func3里指定的宽度区分。

S类型的指令3个，opcode为STORE，这3个指令通过func3里指定的宽度区分。

##### 内存次序指令

I类型的指令1个，opcode为MISC-MEM，func3为FENCE。

##### 环境调用与断点指令

I类型的指令2个，opcode为SYSTEM，func3为PRIV，func12为ECALL或EBREAK。
