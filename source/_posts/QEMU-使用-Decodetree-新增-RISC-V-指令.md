---
title: 'QEMU: 使用 Decodetree 新增 RISC-V 指令'
date: 2020-02-16 17:30:00
tags:
- QEMU
- RISC-V
categories:
- [QEMU, RISC-V]
---

> ⚠️ 本文所使用的 QEMU 版本為：`v4.2.0`

在之前的文章中 ([Part 1.](/QEMU-Decodetree-語法介紹-Part-1), [Part 2.](/QEMU-Decodetree-語法介紹-Part-2)) 我們提到了如何使用 `Decodetree` 來定義指令的 decoder。本篇文章就實際使用 `Decodetree` 來定義一個 QEMU RISC-V 目前尚未支援的指令 - `B(itmanip) Extension` 中的 `pcnt` 指令，並實做其行為。

<!-- more -->

---

# pcnt 指令

`pcnt` 指令的定義如下：

> This instruction counts the number of 1 bits in a register. This operations is known as population
count, popcount, sideways sum, bit summation, or Hamming weight.

其指令格式為：

{% codeblock highlight:false line_number:false %}
| 1 0 9 8 7 6 5 | 4 3 2 1 0 | 9 8 7 6 5 | 4 3 2 | 1 0 9 8 7 | 6 5 4 3 2 1 0 |
|===========================================================================|
|    0110000    |   00010   |    rs1    |  001  |     rd    |    0010011    | PCNT
{% endcodeblock %}

---

# 安裝 toolchain

由於 `B Extension` 尚未正式定稿 (Draft)，因此必須至 [riscv-bitmanip repo](https://github.com/riscv/riscv-bitmanip/tree/master/tools) 下載 toolchain，並依照該 repo 的指示安裝：

{% codeblock highlight:false line_number:false %}
sudo mkdir /opt/riscv64b
sudo chown $USER: /opt/riscv64b
bash build-all.sh
{% endcodeblock %}

此安裝除了 toolchain 外，還會安裝支援 `B Extension` 的 `Spike (riscv-isa-sim)` 及 `riscv-pk` (P.S. 目前的 script 是寫死安裝路徑為：`/opt/riscv64b`)。

---

# 範例程式

在安裝好後，我們可以寫一個範例程式，並搭配 Spike 來做測試：

```c pcnt_example.c
#include &lt;stdio.h&gt;

int count_set_bits(int num)
{
    int count;
    __asm__("pcnt %0, %1\n"
            : "=r"(count)
            : "r"(num)
            :);
    return count;
}

int main(void)
{
    int num = 187;
    int result = count_set_bits(num);

    printf("num = %d\n", num);
    printf("# of set bits = %d\n", result);

    return 0;
}
```

此範例程式做的事情很簡單，透過 inline assembly：`pcnt` 指令，將 `int num = 187` 的 `1 bits` 個數給計算出來。

透過剛剛的 toolchain 編譯此程式：

{% codeblock highlight:false line_number:false %}
/opt/riscv64b/bin/riscv64-unknown-elf-gcc -Wall -march=rv64gb -Os -o pcnt_example pcnt_example.c
{% endcodeblock %}

- -march=rv64gb：指定 target ISA 為 `RISC-V 64-bit` + `g (IMAFD base)` + `b (B Extension)`。

透過 objdump 觀看其反組譯碼：

{% codeblock highlight:false line_number:false %}
/opt/riscv64b/bin/riscv64-unknown-elf-objdump -S pcnt_example > pcnt_example.s
{% endcodeblock %}


```lang:assembly pcnt_example.s
pcnt_example:     file format elf64-littleriscv

Disassembly of section .text:

00000000000100b0 &lt; main &gt;:
   100b0:       00017537                lui     a0,0x17
   100b4:       ff010113                addi    sp,sp,-16
   100b8:       0bb00593                li      a1,187
   100bc:       8b850513                addi    a0,a0,-1864 # 168b8 <__trunctfdf2+0x2ce>
   100c0:       00113423                sd      ra,8(sp)
   100c4:       00813023                sd      s0,0(sp)
   100c8:       2ac000ef                jal     ra,10374 <printf>
   100cc:       0bb00413                li      s0,187
   100d0:       00017537                lui     a0,0x17
   100d4:       60241413                pcnt    s0,s0
   100d8:       0004041b                sext.w  s0,s0
   100dc:       00040593                mv      a1,s0
   100e0:       8c850513                addi    a0,a0,-1848 # 168c8 <__trunctfdf2+0x2de>
   100e4:       290000ef                jal     ra,10374 <printf>
   100e8:       00813083                ld      ra,8(sp)
   100ec:       00013403                ld      s0,0(sp)
   100f0:       00000513                li      a0,0
   100f4:       01010113                addi    sp,sp,16
   100f8:       00008067                ret
```

可以看到 `Line:15` 呼叫了 `pcnt` 指令：`pcnt s0, s0`。 

透過 Spike 執行程式：

{% codeblock highlight:false line_number:false %}
/opt/riscv64b/bin/spike --isa=RV64GCB pk pcnt_example
{% endcodeblock %}

{% codeblock highlight:false line_number:false %}
bbl loader
num = 187
# of set bits = 6
{% endcodeblock %}

`187` 的二進位為 `10111011`，`1 bits` 個數為 `6`，與程式輸出的結果一致。

同樣的程式，我們使用 QEMU 來執行：

{% codeblock highlight:false line_number:false %}
./qemu/riscv64-linux-user/qemu-riscv64 pcnt_example
{% endcodeblock %}

{% codeblock highlight:false line_number:false %}
num = 187
Illegal instruction
{% endcodeblock %}

可以看到，目前 QEMU 尚未支援 `pcnt` 指令，因此當執行到 `pcnt` 指令時，便會噴 `Illegal instruction` 的錯誤訊息。

---

# 在 QEMU 中新增 pcnt 指令

根據前述 `B Extension` spec. 所列的 `pcnt` 指令格式，參考目前 QEMU RISC-V 現有的 `Decodetree`：`target/riscv/insn32.decode`，`pcnt` 的 `Pattern` 可以搭配 `@r2` 的 `Format` (只有 `rs1` 及 `rd` 這兩個 `Fields`)，其完整定義如下：

### Field

{% codeblock lang:plain first_line:22 ./target/riscv/insn32.decode %}
%rs1       15:5
%rd        7:5
{% endcodeblock %}

### Format

{% codeblock lang:plain first_line:64 ./target/riscv/insn32.decode %}
@r2      .......   ..... ..... ... ..... ....... %rs1 %rd
{% endcodeblock %}

### Pattern

我們可以定義 `pcnt` 的 `Pattern` 如下：

{% codeblock lang:plain first_line:207 ./target/riscv/insn32.decode %}
# *** RV32B Standard Extension ***
pcnt       0110000  00010 ..... 001 ..... 0010011 @r2
{% endcodeblock %}

所會產生的 decoder 如下：

{% codeblock lang:c first_line:38 ./riscv64-linux-user/target/riscv/decode_insn32.inc.c %}
typedef struct {
    int rd;
    int rs1;
} arg_decode_insn3213;
{% endcodeblock %}

{% codeblock lang:c first_line:491 ./riscv64-linux-user/target/riscv/decode_insn32.inc.c %}
static void decode_insn32_extract_r2(DisasContext *ctx, arg_decode_insn3213 *a, uint32_t insn)
{
    a->rs1 = extract32(insn, 15, 5);
    a->rd = extract32(insn, 7, 5);
}
{% endcodeblock %}

{% codeblock lang:c first_line:673 ./riscv64-linux-user/target/riscv/decode_insn32.inc.c %}
<pre>
            case 0x1:
                /* 01...... ........ .001.... .0010011 */
                decode_insn32_extract_r2(ctx, &u.f_decode_insn3213, insn);
                switch ((insn >> 20) & 0x3ff) {
                case 0x202:
                    /* 01100000 0010.... .001.... .0010011 */
                    /* /home/frankchang/qemu/target/riscv/insn32.decode:208 */
                    if (trans_pcnt(ctx, &u.f_decode_insn3213)) return true;
                    return false;
                }
                return false;
            }
</pre>
{% endcodeblock %}

由於 `@r2 Format` 並沒有參考任何的 `Argument Set`，因此 `Decodetree` 會自動根據 `Format` 所參考到的 `Fields` (`rs1`、`rd`) 動態產生 `argument set struct`: `arg_decode_insn3213`。

此外，由 `pcnt Pattern` 所產生的 decode function 會呼叫 `decode_insn32_extract_r2()` 這個 extract function 來解析指令中 `rs2` 及 `rd` 欄位的值，並更新所傳入 `arg_decode_insn3213` 對應的欄位，而後再呼叫 `trans_pcnt()` 來執行 `pcnt` 指令 (產生對應的 `TCG ops`)。因此，我們還必須定義 `trans_pcnt()` 來實作 `pcnt` 的指令行為。

---

參考 `B Extension` spec. 中，`pcnt` 指令的實作：

```c
uint_xlen_t pcnt(uint_xlen_t rs1)
{
    int count = 0;
    for (int index = 0; index < XLEN; index++)
    count += (rs1 >> index) & 1;
    return count;
}
```

及 Spike 中，`pcnt` 指令的實作：

```c riscv-isa-sim/riscv/insns/pcnt.h 
require_extension('B');
reg_t x = 0;
for (int i = 0; i < xlen; i++)
  if (1 & (RS1 >> i)) x++;
WRITE_RD(sext_xlen(x));
```

實作很簡單，每次迴圈 right shift `rs1` `i` 個 bits 並與 `1` 做 `AND`，若為 `true` 就將 `count` 加 `1`，最後回傳的 `count` 就是 `1 bits` 個數。

---

`trans_pcnt()` 實作了 `pcnt` 指令對應的 `TCG ops`。QEMU 在執行時，會將 `target instructions` (e.g. RISC-V instructions) 轉譯成 `TCG ops`，而 `TCG ops` 則會再轉譯為 `host instructions` (e.g. x86 instruction)。

{% codeblock lang:plain line_number:false QEMU dynamic instructions translation %}
+---------------------+      +---------+      +-------------------+
| Target Instructions | ---> | TCG ops | ---> | Host instructions |
+---------------------+      +---------+      +-------------------+
     (e.g. RISC-V)                                  (e.g. x86)
{% endcodeblock %}

關於 `TCG` 的說明，可以參考 QEMU 的 documentations：[Translator Internals](https://github.com/qemu/qemu/blob/master/docs/devel/tcg.rst)、[TCG README](https://github.com/qemu/qemu/blob/master/tcg/README)。

新增一檔案：`./target/riscv/insn_trans/trans_rvb.inc.c` 來定義 `B Extension` 指令的實作 (當然，目前只有 `pcnt` 指令)：

```c ./target/riscv/insn_trans/trans_rvb.inc.c
/*
 * RISC-V translation routines for the RVB Standard Extension.
 */

static bool trans_pcnt(DisasContext *ctx, arg_pcnt *a) {
    if (a->rd != 0) {
        TCGv t0 = tcg_temp_new();
        gen_get_gpr(t0, a->rs1);
        gen_helper_pcnt(cpu_gpr[a->rd], t0);
        tcg_temp_free(t0);
    }
    return true;
}
```

由於對 `x0` (`zero register`) 的寫入都會被忽略，因此首先判斷 `rd` 是否為 `0`，若為 `0` 則不做任何的事情。

再來宣告一 TCG variable：`t0`，並透過 `gen_get_gpr()` 將 `rs1` 暫存器的值 (如 `pcnt_example` 中 `pcnt s0, s0` 指令，`rs1` 即為 `s0`，也就是 `x8`)，載入到 `t0`。

這邊還呼叫了我們所定義幫我們處理 `pcnt` 計算 `1 bits` 個數的 `pcnt` helper function：`gen_helper_pcnt()`。該 helper function 會在計算完後，將最後的結果存至 `rd` (i.e. `cpu_gpr[a->rd]`) 暫存器中。

最後別忘了要釋放之前所宣告的 TCG variable：`t0`。

P.S. 其實這邊可以更簡單的直接將 `cpu_gpr[a->rs1]` 傳入，省略 TCG variable：`t0` 的宣告：

```c ./target/riscv/insn_trans/trans_rvb.inc.c
/*
 * RISC-V translation routines for the RVB Standard Extension.
 */

static bool trans_pcnt(DisasContext *ctx, arg_pcnt *a) {
    if (a->rd != 0) {
        gen_helper_pcnt(cpu_gpr[a->rd], cpu_gpr[a->rs1]);
    }
    return true;
}
```

`pcnt` 的 helper function 定義如下：

```c ./target/riscv/helper.h
/* Bitmanip Extension */
DEF_HELPER_1(pcnt, tl, tl)
```

```c ./target/riscv/bitmanip_helper.c
/*
 * RISC-V Bitmanip Extension Helpers for QEMU.
 */
#include "qemu/osdep.h"
#include "cpu.h"
#include "exec/exec-all.h"
#include "exec/helper-proto.h"

target_ulong HELPER(pcnt)(target_ulong rs1)
{
    target_ulong count = 0;
    for (int i = 0; i < TARGET_LONG_BITS; i++) {
        count += (rs1 >> i) & 1;;
    }
    return count;
}
```

基本上就是實作先前在 `B Extension` spec. 及 Spike 中所看到的 `1 bits` 個數計算方式。由於 `pcnt` helper function 只需接收 `rs1` 暫存器的值，並回傳最後 `1 bits` 個數的結果，因此，我們定義 `pcnt` 的 helper function 為接收一 `target_ulong` 型態的 `rs1` 並回傳 `target_ulong` 型態的 `1 bits` 個數結果。

最後別忘了將我們新增的 `bitmanip_helper.o` 加入 compile objects 列表：

```lang:makefile ./target/riscv/Makefile.objs
obj-y += translate.o op_helper.o cpu_helper.o cpu.o csr.o fpu_helper.o bitmanip_helper.o gdbstub.o
```

---

重新編譯 QEMU，再次執行 `pcnt_example`：

{% codeblock highlight:false line_number:false %}
./qemu/riscv64-linux-user/qemu-riscv64 pcnt_example
{% endcodeblock %}

{% codeblock highlight:false line_number:false %}
num = 187
# of set bits = 6
{% endcodeblock %}

這次 QEMU 就可以正確的 decode 並執行 `pcnt` 指令了。

---

在 QEMU 中新增指令的流程大致就如同本文所介紹，不過由於 `pcnt` 指令只是單純的 bit operation 指令，沒有像 `csr` 相關指令會涉及 `CPURISCVState` 的更新，以及像 `jal` 指令會涉及 `DisasContext` 的判斷，因此實作起來相對簡單。若欲讓 QEMU 支援不論是 `B Extension` 或是 `V Extension` 的其他指令，就是得好好 K spec. 並一個一個新增了。

另外最近剛好 C-Sky Microsystems 的 `LIU Zhiwei &lt;zhiwei_liu@c-sky.com&gt;` 在實作 `V Extension` 的 configure instructions：`vsetvl` 及 `vsetvli`，比起本文所介紹之 `B Extension` 的 `pcnt` 指令要來得複雜得多，[patches](https://github.com/patchew-project/qemu/commit/04f8d9c87bf88fabd1221f8758187ba1fa5628f0) 仍在被 reviewed 中，也可以做為參考。

本文所對 QEMU 做的修正，可以參考此 [commit](https://github.com/frankchang0125/qemu/commit/3ca50fb830ab3f2b9a1b806e2240ea5bff9dfabf)。
