---
title: QEMU Decodetree 語法介紹 (Part 1.)
date: 2020-01-31 21:53:40
tags:
- QEMU
- RISC-V
categories:
- [QEMU, RISC-V]
---

> ⚠️ 本文所使用的 QEMU 版本為：`v4.2.0`

QEMU 在 decode 指令的時候，需要呼叫各平台所定義的 instruction decoders 來解析指令。如在 ARM 平台下，就定義了：`disas_arm_insn()`、`disas_thumb_insn()` 及 `disas_thumb2_insn()` 等來分別負責 ARM 32-bits 指令、ARM Thumb 指令及 ARM Thumb2 指令的解析。

而 `Decodetree` 則是由 `Bastian Koppelmann` 於 2017 年在 porting RISC-V QEMU 的時候所提出來的機制 (詳見：[討論串 1](https://lists.gnu.org/archive/html/qemu-devel/2017-07/msg07735.html)、[討論串 2](https://lists.gnu.org/archive/html/qemu-devel/2017-10/msg05046.html))。主因是過往的 instruction decoders (如：ARM) 都是採用一大包的 `switch-case` 來做判斷。不僅難閱讀，也難以維護。

因此 `Bastian Koppelmann` 就提出了 `Decodetree` 的機制，開發者只需要透過 `Decodetree` 的語法定義各個指令的格式，便可交由 `Decodetree` 來動態生成對應包含 `switch-case` 的 instruction decoder `.c` 檔。

<!-- more -->

`Decodetree` 特別適合像 RISC-V 這種具有**固定指令格式**的 ISA[^1]。

- 因為各欄位都在固定的位置，(如 RISC-V 的 `opcode` 都是固定在 `bits[6..0]` 的位置)，各指令可重複使用的定義相較於其他的 ISA 來得多。

{% asset_img riscv_insn_format.png RISC-V instruction formats %}

`Decodetree` 其實是由 Python script (`./scripts/decodetree.py`) 所撰寫的。其規格說明文件可以參考：`./docs/devel/decodetree.rst`，裡面有詳細定義了其語法的格式。QEMU 在編譯時，會呼叫 `Decodetree`，根據各平台所定義的 decode 檔，動態生成對應的 decoder。

- 如 RISC-V 的 instruction decoders 就是被定義在：`./target/riscv/*.decode` 中。其 `Makefile.obj` 就有如下的宣告：

    ```highlight:false
    ...
    
    DECODETREE = $(SRC_PATH)/scripts/decodetree.py
    
    decode32-y = $(SRC_PATH)/target/riscv/insn32.decode
    decode32-$(TARGET_RISCV64) += $(SRC_PATH)/target/riscv/insn32-64.decode
    
    ...
    
    target/riscv/decode_insn32.inc.c: $(decode32-y) $(DECODETREE)
    	$(call quiet-command, \
    	  $(PYTHON) $(DECODETREE) -o $@ --static-decode decode_insn32 \
              $(decode32-y), "GEN", $(TARGET_DIR)$@)
    ```

    (實際參數說明請見 [decodetree 參數](#decodetree-參數))

`Decodetree` 的語法共分為：[Fields](#Fields)、[Argument Sets](#Argument-Sets)、[Formats](#Formats)、[Patterns](/QEMU-Decodetree-語法介紹-Part-2/#Patterns)、[Pattern Groups](/QEMU-Decodetree-語法介紹-Part-2/#Pattern-Groups) 五部分。本文將介紹如何透過 `Decodetree` 的語法，來動態生成一個指令的 decoder。

---

# Fields

`Field` 定義如何取出一指令中，各**欄位** (eg: `rd`, `rs1`, `rs2`, `imm`) 的值。

```highlight:false
field_def     := '%' identifier ( unnamed_field )* ( !function=identifier )?
unnamed_field := number ':' ( 's' ) number
```

其語法由 `%` 開頭，隨後緊接著一個 `identifier` 及零個或多個 `unamed_field`，並可再加上可選的 `!function`。

- `identifier` 可由開發者自訂，如：`rd`、`imm`... 等。
- `unamed_field` 定義了該欄位的所在位元。第一個數字定義了該欄位的 `least-significant bit position`，第二個數字則定義了該欄位的`位元長度`。另外可加上可選的 `s` 字元來標明在取出該欄位後，是否需要做 `sign-extended`。
    - Eg：`%rd  7:5` 代表 `rd` 佔了指令中 bits 7 ~ bits 11 的位置 (insn[11:7])，共 5 bits。
- `!function` 定義在擷取出該欄位的值後，所會再呼叫的 function。

`Field` (32-bits 指令) 最後會生成對應的 `extract32()` 及 `sextract32()` 程式碼[^2]，以用來取得指令中各欄位的值：


### Examples

| Input | Generated code |
|-------|----------------|
| %disp 0:s16 | sextract(i, 0, 16)  |
| %imm9 16:6 10:3 | extract(i, 16, 6) << 3 \| extract(i, 10, 3) |
| %disp12 0:s1 1:1 2:10 | sextract(i, 0, 1) << 11 \| extract(i, 1, 1) << 10 \| extract(i, 2, 10) |
| %shimm8 5:s8 13:1 !function=expand_shimm8 | expand_shimm8(sextract(i, 5, 8) << 1 \| extract(i, 13, 1)) |

以 RISC-V 的 `U-type` 指令為例：

{% asset_img riscv_u_type_insn.png RISC-V U-type instruction %}

其中，`imm` 佔 `insn[31:12]`，`rd` 佔 `insn[11:7]`，且 `imm` 需要做 `sign-extended`  後 `左移 12 位` (`20-bit immediate is shifted left by 12 bits to form U immediates`)。因此，如果我們要定義 RISC-V 的 `U-type` 指令，則可以宣告成：

```highlight:false
%rd       7:5
%imm_u    12:s20                 !function=ex_shift_12
```

最後會生成如下的程式碼：

```c
static void decode_insn32_extract_u(DisasContext *ctx, arg_u *a, uint32_t insn)
{
    a->imm = ex_shift_12(ctx, sextract32(insn, 12, 20));
    a->rd = extract32(insn, 7, 5);
}
```

(P.S. `static void decode_insn32_extract_u()` 是由 [Format](#Formats) 定義所生成的，而 `arg_u *a` 則是由 [Argument Set](#Argument-Sets) 定義所生成的，將會在後面的部分再做說明)

可以看到：

```c
a->imm = ex_shift_12(ctx, sextract32(insn, 12, 20));
a->rd = extract32(insn, 7, 5);
```

- `a->imm` 是由 `insn[31:12]` 所取得並做 `sign-extended`，且會再呼叫 `ex_shift_12()` 來 `左移 12 個 bits`。
    - P.S. RISC-V 的 `ex_shift_12()` 是透過定義在`./target/riscv/translate.c` 中 `EX_SH` 這個 macro 所展開的：

    ```c
    #define EX_SH(amount) \
        static int ex_shift_##amount(DisasContext *ctx, int imm) \
        {                                         \
            return imm << amount;                 \
        }
    EX_SH(1)
    EX_SH(2)
    EX_SH(3)
    EX_SH(4)
    EX_SH(12)
    ```

- `a->rd` 是由 `insn[11:7]` 所取得。

此外，在 `Decodetree` 的 spec. 中也有提到，我們可以透過只定義 `!function` 來直接呼叫該 function。在這種情況下，只有 `DisasContext` 會被傳入該 function。

One may use `!function` with zero `unnamed_fields`.  This case is called
a **parameter**, and the named function is only passed the `DisasContext`
and returns an integral value extracted from there.

如 ARM Thumb `./target/arm/t16.decode` 就有定義：

```highlight:false
# Set S if the instruction is outside of an IT block.
%s               !function=t16_setflags
```

```c
static void disas_t16_extract_addsub_2i(DisasContext *ctx, arg_s_rri_rot *a, uint16_t insn)
{
    a->imm = extract32(insn, 6, 3);
    a->rn = extract32(insn, 3, 3);
    a->rd = extract32(insn, 0, 3);
    a->s = t16_setflags(ctx); // 呼叫 t16_setflags()，並傳入 DisasContext
    a->rot = 0;
}
```

未包含任何 `unnamed_fields` 或 `!function` 的 `Field` 會被視為錯誤。

A field with no `unnamed_fields` and no `!function` is in error.

---

# Argument Sets

`Argument Set` 定義用來保存從指令中所擷取出來各欄位的值。

```highlight:false
args_def    := '&' identifier ( args_elt )+ ( !extern )?
args_elt    := identifier
```

其語法由 `&` 開頭，隨後緊接著一個或多個的 `identifier` ，並可再加上可選的 `!extern` 。

- `identifier` 可由開發者自訂，如：`regs`、`loadstore`... 等。
- `!extern` 則表示是否在其他地方已經由其他的 decoder 定義過。如果加上的話，就**不會**再次生成對應的 `argument set struct`。

### Examples

---

```highlight:false
&ampreg3 ra rb rc
```

會生成以下的 `argument set struct`：

```c
typedef struct {
    int ra;
    int rb;
    int rc;
} arg_reg3;
```

---

```highlight:false
&loadstore reg base offset
```

則會生成以下的 `argument set struct`：

```c
typedef struct {
    int base;
    int offset;
    int reg;
} arg_loadstore;
```

---

因此，以剛剛的 RISC-V `U-type` 指令為例，我們需要從指令中擷取 `imm` 及 `rd` 欄位的值，可以宣告其 `argument set` 如下：

```highlight:false
&u    imm rd
```

最後會生成以下的 `argument set struct`：

```c
typedef struct {
    int imm;
    int rd;
} arg_u;
```

此 `argument set struct` 會被傳入由 `Format` 定義所生成的 extract function：

```c
static void decode_insn32_extract_u(DisasContext *ctx, arg_u *a, uint32_t insn)
{
    a->imm = ex_shift_12(ctx, sextract32(insn, 12, 20));
    a->rd = extract32(insn, 7, 5);
}
```

所傳入的`arg_u` 會保存從指令中擷取出的 `imm` 及 `rd` 欄位的值，待後續使用。

---

# Formats

`Format` 定義了指令的格式 (如 RISC-V 中的 `R`、`I`、`S`、`B`、`U`、`J-type`)，並會生成對應的 decode function。

```highlight:false
fmt_def      := '@' identifier ( fmt_elt )+
fmt_elt      := fixedbit_elt | field_elt | field_ref | args_ref
fixedbit_elt := [01.-]+
field_elt    := identifier ':' 's'? number
field_ref    := '%' identifier | identifier '=' '%' identifier
args_ref     := '&' identifier
```

其語法由 `@` 開頭，隨後緊接著一個 `identifier` 及一個以上的 `fmt_elt`。

- `identifier` 可由開發者自訂，如：`opr`、`opi`... 等。
- `fmt_elt` 則可以採用以下不同的語法：
    - `fixedbit_elt` 包含一個或多個  `0`、`1`、`.`、`-`，每一個代表指令中的 1 個 bit。
        - `.` 代表該 bit 可以用 `0` 或是 `1` 來表示。
        - `-` 代表該 bit 完全被忽略。
    - `field_elt` 可以用 [Field](#Fields) 的語法來宣告。
        - Eg：`ra:5`、`rb:5`、`lit:8`
    - `field_ref` 有下列兩種格式 (以下範例參考上文所定義之 [Field](#Fields))：
        - `'%' identifier`：直接參考一個被定義過的 `Field`。
            - 如：`%rd`，會生成：

                ```c
                a->rd = extract32(insn, 7, 5);
                ```

        - `identifier '=' '%' identifier`：直接參考一個被定義過的 `Field`，但透過第一個 `identifier` 來重新命名其所對應的 `argument` 名稱。此方式可以用來指定不同的 `argument` 名稱來參考至同一個 `Field`。
            - 如：`my_rd=%rd`，會生成：

                ```c
                a->my_rd = extract32(insn, 7, 5); // rd 被重新命名為 my_rd
                ```

    - `args_ref` 指定所傳入 decode function 的 `Argument Set`。若沒有指定 `args_ref` 的話，`Decodetree` 會根據 `field_elt` 或 `field_ref` 自動生成一個 `Argument Set`。此外，一個 `Format` 最多只能包含一個 `args_ref`。

當 `fixedbit_elt` 或 `field_ref` 被定義時，該 `Foramt` 的所有的 bits 都必須被定義 (可透過 `fixedbit_elt` 或 `.` 來定義各個 bits，`空格`會被忽略)。

### Examples

```highlight:false
@opi    ...... ra:5 lit:8    1 ....... rc:5
```

定義了 `op1` 這個 `Format`，其中：

- insn[31:26] 可為 `0` 或 `1`。
- insn[25:21] 為 `ra`。
- insn[20:13] 為 `lit`。
- insn[12] 固定為 `1`。
- insn[11:5] 可為 `0` 或 `1`。
- insn[4:0] 為 `rc`。

此 `Format` 會生成以下的 decode function：

```c
typedef struct {
    int lit;
    int ra;
    int rc;
} arg_decode_insn320;


static void decode_insn32_extract_opi(DisasContext *ctx, arg_decode_insn320 *a, uint32_t insn)
{
    a->ra = extract32(insn, 21, 5);
    a->lit = extract32(insn, 13, 8);
    a->rc = extract32(insn, 0, 5);
}
```

由於我們沒有指定 `args_ref`，因此 `Decodetree` 根據了 `field_elt` 的定義，自動生成了 `arg_decode_insn320` 這個 `Argument Set`。

---

以 RISC-V `I-type` 指令為例：

```highlight:false
# Fields:
%rs1       15:5
%rd        7:5

# immediates:
%imm_i    20:s12

# Argment sets:
&i    imm rs1 rd

@i       ........ ........ ........ ........ &i      imm=%imm_i     %rs1 %rd
```

定義了 `i` 這個 `Format`，其中：

- insn[31:20] 為 `imm`，且為 `sign-extended`。
- insn[19:5] 為 `rs1`。
- insn[11:7] 為 `rd`。

此外，我們可以看到：

- 此 `Format` 指定了 `Argument Set`：`&i`。 `&i` 中必須包含所有有用到的 `arguments` (也就是：`imm`、`rs1` 及 `rd`)
- `imm` 是透過重新命名的方式來參考 `%imm_i` 這個 `Field`。

此範例會生成以下的 decode function：

```c
typedef struct {
    int imm;
    int rd;
    int rs1;
} arg_i;


static void decode_insn32extract_i(DisasContext *ctx, arg_i *a, uint32_t insn)
{
    a->imm = sextract32(insn, 20, 12); // imm_i 被重新命名為 imm
    a->rs1 = extract32(insn, 15, 5);
    a->rd = extract32(insn, 7, 5);
}
```

相比於第一個範例，由於這次我們有指定 `args_ref`：`&i`，因此對應的 `arg_i` 會被傳入 decode function。

---

回到先前的 RISC-V `U-type` 指令，我們可以如同 `I-type` 指令定義其格式：

```highlight:false
# Fields:
%rd        7:5

# immediates:
%imm_u    12:s20                 !function=ex_shift_12

# Argument sets:
&u    imm rd

@u       ....................      ..... ....... &u      imm=%imm_u          %rd
```

定義了 `u` 這個 `Format`，其中：

- insn[31:12] 為 `imm`，且為 `sign-extended`。
- insn[11:7] 為 `rd`。

會生成以下的 decode function：

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
```

我們可以看到：

- 此 `Format` 指定了 `Argument Set`：`&u`。 `&u` 中必須包含所有有用到的 `arguments` (也就是：`imm`、`rd`)
- `imm` 是透過重新命名的方式來參考 `%imm_u` 這個 `Field`。

---

以上就是 `Decodetree` 的 [Fields](#Fields)、[Argument Sets](#Argument-Sets) 及 [Formats](#Formats) 語法的簡介。剩下的 [Patterns](/QEMU-Decodetree-語法介紹-Part-2/#Patterns) 及 [Pattern Groups](/QEMU-Decodetree-語法介紹-Part-2/#Pattern-Groups) 就留到 {% post_link QEMU-Decodetree-語法介紹-Part-2 "Part 2." %} 再做介紹。

---

# decodetree 參數

- `--translate`：translator function 的 prefix，預設為 `trans`。一旦指定後，translator function 的 scope 就不會再是 `static`。
- `--decode`：decode function 的 prefix，預設為 `decode`，且 scope 為 `static`。一旦指定後，decode function 的 scope 就不會再是 `static`。
- `--static-decode`：如同 `--decode`，不過 decode function 的 scope 仍維持為 `static`。
- `-o` / `--output`：指定生成的 decoder `.c` 檔路徑。
- `-w` / `--insnwidth`：指令長度，eg：`32` or `16`，預設為 `32`。
- `--varinsnwidth`：指令為不定長度。
- `最後一個參數`為輸入的 decode 檔路徑。

執行範例：

```highlight:false
./decodetree.py -o target/riscv/decode_insn16.inc.c --static-decode decode_insn16 \
    -w 16 ./insn16.decode
```

[^1]: ARM 其實在 `Decodetree` 引進後，也有部分的 instructions 改採用 `Decodetree` 來動態生成對應的 instruction decoders，如 Thumb 指令：`./target/arm/t32.decode` 及 `./target/arm/t16.decode`。

[^2]: `extract32()` 及 `sextract32()` 被定義在 `include/qemu/bitops.h`：

    ```c
    /**
     * extract32:
     * @value: the value to extract the bit field from
     * @start: the lowest bit in the bit field (numbered from 0)
     * @length: the length of the bit field
     *
     * Extract from the 32 bit input @value the bit field specified by the
     * @start and @length parameters, and return it. The bit field must
     * lie entirely within the 32 bit word. It is valid to request that
     * all 32 bits are returned (ie @length 32 and @start 0).
     *
     * Returns: the value of the bit field extracted from the input value.
     */
    static inline uint32_t extract32(uint32_t value, int start, int length)
    {
        assert(start >= 0 && length > 0 && length <= 32 - start);
        return (value >> start) & (~0U >> (32 - length));
    }
    ```

    ```c
    /**
     * sextract32:
     * @value: the value to extract the bit field from
     * @start: the lowest bit in the bit field (numbered from 0)
     * @length: the length of the bit field
     *
     * Extract from the 32 bit input @value the bit field specified by the
     * @start and @length parameters, and return it, sign extended to
     * an int32_t (ie with the most significant bit of the field propagated
     * to all the upper bits of the return value). The bit field must lie
     * entirely within the 32 bit word. It is valid to request that
     * all 32 bits are returned (ie @length 32 and @start 0).
     *
     * Returns: the sign extended value of the bit field extracted from the
     * input value.
     */
    static inline int32_t sextract32(uint32_t value, int start, int length)
    {
        assert(start >= 0 && length > 0 && length <= 32 - start);
        /* Note that this implementation relies on right shift of signed
         * integers being an arithmetic shift.
         */
        return ((int32_t)(value << (32 - length - start))) >> (32 - length);
    }
    ```
