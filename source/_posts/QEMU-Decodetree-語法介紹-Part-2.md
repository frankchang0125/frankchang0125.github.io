---
title: QEMU Decodetree 語法介紹 (Part 2.)
date: 2020-02-01 14:57:43
tags:
- QEMU
- RISC-V
categories:
- [QEMU, RISC-V]
---

> ⚠️ 本文所使用的 QEMU 版本為：`v4.2.0`

延續 {% post_link QEMU-Decodetree-語法介紹-Part-1 "Part 1." %} 一文，本文將繼續介紹 `Decodetree` 中的 `Patterns` 及 `Pattern Groups` 語法。

<!-- more -->

---

# Patterns

`Pattern` 實際定義了一個指令的 decode 方式。`Decodetree` 會根據 `Patterns` 的定義，來動態產生出對應的 `switch-case` decode 判斷式。

```highlight:false
pat_def      := identifier ( pat_elt )+
pat_elt      := fixedbit_elt | field_elt | field_ref | args_ref | fmt_ref | const_elt
fmt_ref      := '@' identifier
const_elt    := identifier '=' number
```

其語法由使用者所定義的 `identifier`，隨後緊接著一個以上的 `pat_elt`。

- `identifier` 可由開發者自訂，如：`addl_r`、`addli` ... 等。
- `pat_elt` 則可以採用以下不同的語法：
    - `fixedbit_elt` 與在 `Format` 中 `fixedbit_elt` 的定義相同。
    - `field_elt` 與在 `Format` 中 `field_elt` 的定義相同。
    - `field_ref` 與在 `Format` 中 `field_ref` 的定義相同。
    - `args_ref` 與在 `Format` 中 `args_ref` 的定義相同。
    - `fmt_ref` 直接參考一個被定義過的 [Format](/QEMU-Decodetree-語法介紹-Part-1/#Formats)。
    - `const_elt` 可以直接指定某一個 `argument` 的值。

由於 `Pattern` 實際定義了一個指令的 decode 方式，因此**所有的 bits** 及 **arguments (如果有參考 args_ref 的話)**  都必須明確的被定義，如果在搭配了所有的 `pat_elt` 後還有未定義的 bits 或是 arguments 的話，`Decodetree` 便會報錯。

此外，`Pattern` 所產生出來的 decoder，最後還會呼叫對應的 `translator function`。

- `translator function` 需開發者自行定義。

### Examples

```highlight:false
addl_i   010000 ..... ..... .... 0000000 ..... @opi
```

定義了 `addl_i` 這個指令的 `Pattern`，其中：

- insn[31:26] 為 `010000`。
- insn[11:5] 為 `0000000`。
- 參考了 [Part 1. Examples](/QEMU-Decodetree-語法介紹-Part-1/#Examples-3) 定義的 `@opi` `Format`。
- 由於 `Pattern` 的**所有 bits** 都必須明確的被定義，因此 `@opi` 必須包含其餘 `insn[25:12]` 及 `insn[4:0]` 的格式定義，否則 `Decodetree` 便會報錯。

最後 `addl_i` 的 decoder 還會呼叫 `trans_addl_i()` 這個 `translator function`。

搭配之前介紹的 [Fields](/QEMU-Decodetree-語法介紹-Part-1/#Fields)、[Argument Sets](/QEMU-Decodetree-語法介紹-Part-1/#Argument-Sets) 及 [Formats](/QEMU-Decodetree-語法介紹-Part-1/#Formats)，讓我們再看幾個完整的例子應該會更清楚 `Decodetree` 是怎產生一個指令的 decoder 的。

---

首先是 RISC-V 的 `lui` 及 `auipc` 指令：

{% asset_img lui_auipc.png LUI & AUIPC instruction formats %}

```highlight:false
# Fields:
%rd        7:5

# immediates:
%imm_u    12:s20                 !function=ex_shift_12


# Argument sets:
&u    imm rd


# Formats:
@u       ....................      ..... ....... &u      imm=%imm_u          %rd


# Patterns
lui      ....................       ..... 0110111 @u
auipc    ....................       ..... 0010111 @u
```

會產生以下 `lui` 及 `auipc` 的 decoder：

```c
typedef struct {
    int imm;
    int rd;
} arg_u;


static void decode_insn32_extract_u(DisasContext *ctx, arg_u *a, uint32_t insn)
{
    a->imm = ex_shift_12(ctx, sextract32(insn, 12, 20));
    a->rd = extract32(insn, 7, 5);
}

static bool decode_insn32(DisasContext *ctx, uint32_t insn)
{
    union {
        arg_u f_u;
    } u;

    decode_insn32_extract_u(ctx, &u.f_u, insn);
    switch (insn & 0x0000007f) {
    case 0x00000017:
        /* ........ ........ ........ .0010111 */
        /* ./insn32.decode:18 */
        if (trans_auipc(ctx, &u.f_u)) return true;
        return false;
    case 0x00000037:
        /* ........ ........ ........ .0110111 */
        /* ./insn32.decode:17 */
        if (trans_lui(ctx, &u.f_u)) return true;
        return false;
    }
    return false;
}
```

回顧到目前為止所介紹的：

- `Argument Sets`：`&u` 這個 `argument set` 包含了 `imm` 及 `rd` 這兩個 `arguments`。

    ```c
    typedef struct {
        int imm;
        int rd;
    } arg_u;
    ```

- `Fields`： `imm` 及 `rd`  分別位在 insn[31:12] 及 insn[11:7]，且 `imm` 為 `sign-extended`。最後在擷取出 `imm` 的值後，還會呼叫 `ex_shift_12()`。

    ```c
    a->imm = ex_shift_12(ctx, sextract32(insn, 12, 20));
    a->rd = extract32(insn, 7, 5);
    ```

- `Formats`：`@u` 定義了 RISC-V `U-type` 指令的格式
    - 參考了 `&u` 這個 `Argument Set`，因此 decode function 會傳入 `arg_u` 作為參數。
    - insn[31:12] 參考了 `imm_u` 這個 `Field` (並重新命名為 `imm`)
    - insn[11:7] 參考了 `rd` 這個 `Field`。

    ```c
    static void decode_insn32_extract_u(DisasContext *ctx, arg_u *a, uint32_t insn)
    {
        a->imm = ex_shift_12(ctx, sextract32(insn, 12, 20));
        a->rd = extract32(insn, 7, 5);
    }
    ```

- `Patterns`：
    - `lui` 的 `opcode` (insn[6:0]) 為 `0010111`，也就是 `0x17`，在產生出來的 `switch-case` 中可以看到其對應的 `case`。
    - `lui` 的 decoder 最後呼叫了 `trans_lui()`，並傳入 `DisasContext` 及經由 `decode_insn32_extract_u()` 所解析出來的 `arg_u`。
    - `auipc` 的 `opcode` (insn[6:0]) 為 `0110111`，也就是 `0x37`，在產生出來的 `switch-case` 中可以看到其對應的 `case`。
    - `auipc` 的 decoder 最後呼叫了 `trans_auipc()`，並傳入 `DisasContext` 及經由 `decode_insn32_extract_u()` 所解析出來的 `arg_u`。
    - P.S. 這邊由於 `Decodetree` 發現 `lui` 及 `auipc` 可以共用 `decode_insn32_extract_u()`，因此將其提到了 `switch-case` 之外。

    ```c
    static bool decode_insn32(DisasContext *ctx, uint32_t insn)
    {
        union {
            arg_u f_u;
        } u;
    
        decode_insn32_extract_u(ctx, &u.f_u, insn);
        switch (insn & 0x0000007f) {
        case 0x00000017:
            /* ........ ........ ........ .0010111 */
            /* ./insn32.decode:18 */
            if (trans_auipc(ctx, &u.f_u)) return true;
            return false;
        case 0x00000037:
            /* ........ ........ ........ .0110111 */
            /* ./insn32.decode:17 */
            if (trans_lui(ctx, &u.f_u)) return true;
            return false;
        }
        return false;
    }
    ```

    我們另外可以發現，`Pattern` + `Format` 把所有的 32-bits 都給了明確的定義：

    - `Pattern` 定義了 `opcode` (insn[6:0])。
    - `Format` 參考了 `imm` (insn[31:12]) 及 `rd` (insn[11:7])。

    如果有任何未明確定義的 bits 的話，`Decodetree` 便會報錯，例如如果我們將 `lui` 的 `opcode` 最高 2 個 bits (insn[6:5]) 由 `01` 改成 `..`：

    ```highlight:false
    lui      ....................       ..... ..10111 @u
    ```

    `Decodetree` 在解析時，便會報錯：

    > ./insn32.decode:17: error: ('bits left unspecified (0x00000060)',)

    `Decodetree` 提醒我們，insn[6:5] (`0x00000060`) 尚未給出明確定義，並會顯示出其錯誤的行數。

    `trans_lui()` 和 `trans_auipc()` 被定義在 `target/riscv/insn_trans/trans_rvi.inc.c`：

    ```c
    static bool trans_lui(DisasContext *ctx, arg_lui *a)
    {
        if (a->rd != 0) {
            tcg_gen_movi_tl(cpu_gpr[a->rd], a->imm);
        }
        return true;
    }

    static bool trans_auipc(DisasContext *ctx, arg_auipc *a)
    {
        if (a->rd != 0) {
            tcg_gen_movi_tl(cpu_gpr[a->rd], a->imm + ctx->base.pc_next);
        }
        return true;
    }
    ```

    可以看到 `trans_*()` 負責實際指令的 business logics 及產生對應的 `TCG codes`。

---

如同先前所介紹，`Patterns` 的 `pat_elt` 也可以採用 `field_elt` 語法，如 RISC-V 的 `fence` 指令：

```highlight:false
fence    ---- pred:4 succ:4 ----- 000 ----- 0001111
```

- insn[27:24] 為 `pred`。
- insn[23:20] 為 `succ`。
- insn[14:12] 固定為 `000`。
- insn[6:0] 為 `opcode` (`0001111`)。
- 沒有參考任何的 `Format`。
- 剩下的 insn[31:28]、insn[19:15]、insn[11:7] 被宣告為 `-`，因此就算沒有被明確定義也沒有關係。

所生成 `fence` 的 decoder 如下：

```c
typedef struct {
    int pred;
    int succ;
} arg_decode_insn320;


static void decode_insn32_extract_decode_insn32_Fmt_0(DisasContext *ctx, arg_decode_insn320 *a, uint32_t insn)
{
    a->pred = extract32(insn, 24, 4);
    a->succ = extract32(insn, 20, 4);
}

static bool decode_insn32(DisasContext *ctx, uint32_t insn)
{
    union {
        arg_decode_insn320 f_decode_insn320;
    } u;

    decode_insn32_extract_decode_insn32_Fmt_0(ctx, &u.f_decode_insn320, insn);
    switch (insn & 0x0000707f) {
    case 0x0000000f:
        /* ........ ........ .000.... .0001111 */
        /* ./insn32.decode:2 */
        if (trans_fence(ctx, &u.f_decode_insn320)) return true;
        return false;
    }
    return false;
}
```

值得注意的是，雖然這次我們沒有參考任何的 `Argument Set`，但 `Decodetree` 還是替我們生成了一個包含 `pred` 和 `succ` 的 `arg_decode_insn320` 。

`trans_fence()` 同樣是被定義在 `./target/riscv/insn_trans/trans_rvi.inc.c`：


```c
static bool trans_fence(DisasContext *ctx, arg_fence *a)
{
    /* FENCE is a full memory barrier. */
    tcg_gen_mb(TCG_MO_ALL | TCG_BAR_SC);
    return true;
}
```

---

# Pattern Groups

`Pattern Groups` 由一個以上的 `Patterns` 所組成，其主要差別是不同 `Patterns` 之間的 bits 可以 overlap。當同組中有多個 `Patterns` 時，會依據該組中各 `Pattern` 的宣告順序依序判斷目前的指令是否符合其定義。除此之外，當符合的 `Pattern` 其 `trans_*()` 回傳值為 `false` 時，也會被視為**不相符**，而繼續判斷該組中的下一個 `Pattern`。因此 `Pattern Groups` 非常適合將多個相似格式的指令給組成同一個 `Pattern Group`。

原文說明如下：

> Unlike ungrouped patterns, grouped patterns are allowed to overlap. Conflicts are resolved by selecting the patterns in order.  If all of the `fixedbits` for a pattern match, its translate function will be called.  If the translate function returns `false`, then subsequent patterns within the group will be matched.

```highlight:false
group    := '{' ( pat_def | group )+ '}'
```

各 `Pattern Group` 以 `{` 開頭，並以 `}` 結尾，且允許 `nested pattern groups` 的存在，其他語法皆與 `Pattern` 相同。

### Examples

```highlight:false
{
  {
    nop   000010 ----- ----- 0000 001001 0 00000
    copy  000010 00000 r1:5  0000 001001 0 rt:5
  }
  or      000010 rt2:5 r1:5  cf:4 001001 0 rt:5
}
```

會產生以下的 decoder：

```c
switch (insn & 0xfc000fe0) {
case 0x08000240:
    /* 000010.. ........ ....0010 010..... */
    if ((insn & 0x0000f000) == 0x00000000) {
        /* 000010.. ........ 00000010 010..... */
        if ((insn & 0x0000001f) == 0x00000000) {
            /* 000010.. ........ 00000010 01000000 */
            extract_decode_Fmt_0(&u.f_decode0, insn);
            if (trans_nop(ctx, &u.f_decode0)) return true;
						// 這邊沒有直接回傳 false，讓 switch-case 繼續往下執行
      }
      if ((insn & 0x03e00000) == 0x00000000) {
          /* 00001000 000..... 00000010 010..... */
          extract_decode_Fmt_1(&u.f_decode1, insn);
          if (trans_copy(ctx, &u.f_decode1)) return true;
				　// 這邊沒有直接回傳 false，讓 switch-case 繼續往下執行
      }
  }
  extract_decode_Fmt_2(&u.f_decode2, insn);
  if (trans_or(ctx, &u.f_decode2)) return true;
  return false;
}
```

當指令的值符合 `nop` 及 `copy` 這個內層 `Pattern Group` 時，會先判斷該指令是否符合 `nop` 指令的定義，且 `trans_nop()` 的回傳值為 `true`。否則的話，就會繼續判斷是否符合同組中的 `copy` 指令。若都不符，就會再判斷是否符合外層 `Pattern Group` 的 `or` 指令。若仍不符，才會回傳 `false` 表示 decode 失敗。

與單純使用 `Pattern` 最大不同的是，當一 `Pattern` 的 `trans_*()` 回傳值為 `false` 時，不會直接回傳 `false` (代表 decode 失敗)，而是會接續著判斷後續的 `Patterns` 是否相符。

---

RISC-V Compressed-Extension 中的 `c.ebreak`、`c.jalr`、及 `c.add` 指令，由於這三個指令的格式非常相似，因此非常適合使用 `Pattern Group` 來定義：

{% asset_img risc_v_c_insn.png RISC-V Compressed-Extension instruction formats %}

RISC-V spec. 中定義：

`C.EBREAK` shares the `opcode` with the `C.ADD` instruction, but with `rd` and `rs2` both `zero`, thus can also use the `CR` format.

`C.JALR` is only valid when `rs1≠x0`; the code point with `rs1=x0` corresponds to the `C.EBREAK` instruction.

`C.ADD` is only valid when `rs2≠x0`; the code points with `rs2=x0` correspond to the `C.JALR` and `C.EBREAK` instructions. The code points with `rs2̸=x0` and `rd=x0` are `HINTs`.

`c.ebreak`、`c.jalr`、`c.add` 三個指令：

- insn[15:13]、insn[12]、insn[1:0] 的值皆相同。
- 當 insn[11:7] 且 insn[6:2] 的值皆為 `0` (`rs1=0` 且 `rs2=0`) 時為 `c.ebreak` 指令。
- 當只有 insn[11:7] 的值為 `0` (`rs1=0` 且 `rs2≠0`) 時為 `c.jalr` 指令。
- 否則為 `c.add` 指令 (`rs1≠x0` 且 `rs2≠0`)。

```highlight:false
# Fields
%rd        7:5
%rs2_5     2:5


# Argument Sets
&r         rd rs1 rs2   !extern
&i         imm rs1 rd   !extern


# Formats
@cr        ....  ..... .....  .. &r      rs2=%rs2_5       rs1=%rd     %rd
@c_jalr    ... . .....  ..... .. &i      imm=0 rs1=%rd


# Pattern Groups
{
  ebreak          100 1  00000  00000 10
  jalr            100 1  .....  00000 10 @c_jalr rd=1  # C.JALR
  add             100 1  .....  ..... 10 @cr
}
```

所生成的 decoder 如下：

```c
static void decode_insn16_extract_c_jalr(DisasContext *ctx, arg_i *a, uint16_t insn)
{
    a->imm = 0; // 在 c_jalr 的 Format 中指定 imm 的值為 0
    a->rs1 = extract32(insn, 7, 5);
}

static void decode_insn16_extract_cr(DisasContext *ctx, arg_r *a, uint16_t insn)
{
    a->rs2 = extract32(insn, 2, 5);
    a->rs1 = extract32(insn, 7, 5);
    a->rd = extract32(insn, 7, 5);
}

static void decode_insn16_extract_decode_insn16_Fmt_2(DisasContext *ctx, arg_decode_insn162 *a, uint16_t insn)
{


static bool decode_insn16(DisasContext *ctx, uint16_t insn)
{
    union {
        arg_decode_insn162 f_decode_insn162;
        arg_i f_i;
        arg_r f_r;
    } u;

    switch (insn & 0x0000f003) {
    case 0x00009002:
        /* 1001.... ......10 */
        if ((insn & 0x00000ffc) == 0x00000000) {
            /* 10010000 00000010 */
            /* ./insn16.decode:20 */
            decode_insn16_extract_decode_insn16_Fmt_2(ctx, &u.f_decode_insn162, insn);
            if (trans_ebreak(ctx, &u.f_decode_insn162)) return true;
            // 這邊沒有直接回傳 false，讓 switch-case 繼續往下執行
        }
        if ((insn & 0x0000007c) == 0x00000000) {
            /* 1001.... .0000010 */
            /* ./insn16.decode:21 */
            decode_insn16_extract_c_jalr(ctx, &u.f_i, insn);
            u.f_i.rd = 1; // 在 jalr 的 Pattern 中指定 rd 的值為 0。
            if (trans_jalr(ctx, &u.f_i)) return true;
            // 這邊沒有直接回傳 false，讓 switch-case 繼續往下執行
        }
        /* ./insn16.decode:22 */
        decode_insn16_extract_cr(ctx, &u.f_r, insn);
        if (trans_add(ctx, &u.f_r)) return true;
        return false;
    }
    return false;
}
```

當指令格式符合 `c.ebreak`、`c.jalr`、`c.add` 的 `Pattern Group` 時，會依序判斷該指令是否符合 `c.ebreak`、`c.jalr`、`c.add` 的定義以及其對應的 `trans_*()`。

另外值得一提的是，在 `c_jalr` `Format` 和 `jalr` `Pattern` 中有分別指定其 `imm` 及 `rd` 的值為 `0`，所生成的 codes 也會分別在對應的地方將該值設為 `0` (見 codes 註解說明)。

---

以上就是 `Decodetree` 的語法說明。透過 `Decodetree`，我們不用再像以前以樣寫一大包的 `switch-case` 來 decode 指令。將不同類型的指令寫至不同的 decode 檔，不僅方便維護，閱讀起來也更為容易。
