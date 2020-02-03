---
title: 'Linux Kernel: BUILD_BUG_ON_ZERO() / BUILD_BUG_ON_NULL()'
date: 2012-10-14 22:50-02
tags:
- C
- C++
- Linux Kernel
categories:
- [Linux Kernel, Tricks]
---

之前在 trace Linux Kernel source codes 時發現了兩個很特別的 macros：**BUILD_BUG_ON_ZERO()** 和 **BUILD_BUG_ON_NULL()**
(定義在：*include/linux/kernel.h*)

它們的定義如下：

```c
/* Force a compilation error if condition is true, but also produce a
   result (of value 0 and type size_t), so the expression can be used
   e.g. in a structure initializer (or where-ever else comma expressions
   aren't permitted). */
#define BUILD_BUG_ON_ZERO(e) (sizeof(struct { int:-!!(e); }))
#define BUILD_BUG_ON_NULL(e) ((void *)sizeof(struct { int:-!!(e); }))
```

<!-- more -->

其中 *e* 是我們所傳入的判斷式，若判斷式為 *true*，則會造成 compile error。如此我們便可透過這個 macro 來判斷是否某些錯誤/不應發生的情況 (判斷式) 是否會發生，若會發生則可在 compile-time 的時候就顯示錯誤訊息。

一開始看不太懂這個 macro 的意義，上網查了資料後發現在 Stackoverflow 上有人做了很詳細的解釋：[What is 「:-!!」 in C code?](http://stackoverflow.com/questions/9229601/what-is-in-c-code)

```c
sizeof(struct { int:-!!(e); })
```

如同 Stackoverflow 上所解釋，這段 codes 可以拆成下面幾個片段來分析：

- **!!(e)**

    將所傳入的 *e* 做兩次 negative，如此可以確保只要 *e* 不為 0 結果一定為 1，*e* 為 0 結果仍為 0。

- **-!!(e)**

    將剛剛的結果乘上 (-1)，因此只要 *e* 不為 0 結果就會是 -1，*e* 為 0 結果仍為 0。

- **struct { int:-1!!(e) }**

    宣告一個 structure，包含一個 int，這邊用到了 C 語言 bit-fields的技巧。
    根據[維基百科 bit-fields](http://en.wikipedia.org/wiki/Bit_field) 的定義：

    > A bit field is a common idiom used in computer programming to compactly store multiple logical values as a short series of bits where each of the single bits can be addressed separately.

    也就是說我們可以將資料以 bit 的形式儲存在某一個資料型態中，舉例來說：

    ```c
    struct {
            unsigned char a : 1;
            unsigned char b : 3;
            unsigned char c : 2;
    } flags;
    ```

    就宣告了 3 個 bit-fields：*a*, *b*, *c*。這 3 個 bit-fields 會包在同一個 *unsigned char* 資料型態 (8-bits) 中，其中 *a* 佔了 1 個bit，*b* 佔了 3 個 bits，*c* 佔了 2 個 bits，但整個 flags structure 還是會佔 8 個 bits (1 byte)，即使 bit-fields 並沒有佔滿整個 8 bits 空間。

    此外，bit-fields 是無法使用 **sizeof()** 取得其 size 的，因此以下的 codes 將會產生：**error: 'sizeof' applied to a bit-field** 的錯誤訊息：

    ```c
    printf("%d\n", sizeof(flags.a));
    ```

    ---

    回到原本的 **struct { int:-1!!(e) }**，我們可以得知：

    若 *e* 不為 0，則 **struct { int:-!!(e); }** 會展開成：**struct { int:-1; }**。也就是會宣告一個 structure，包含 1 個佔 int (32 bits) 中，**-1** 個 bits 的 anonymous bit-field。當然，絕對不會有佔 **-1** 個 bits 的 bit-field 存在，因此這樣會導致在編譯時產生：**error: negative width in bit-field '<anonymous>'** 的錯誤訊息。

    若 *e* 為 0，則 **struct { int:-1!!(e); }** 會展開成：**struct { int:0; }**。也就是宣告一個 structure，包含 1 個佔 int (32 bits) 中，**0** 個 bits 的 anonymous bit-field。**0** 個 bits 的 bit-field 並不會造成編譯出錯，事實上，宣告成 **0** 個 bits 的 bit-field 通常是用來將資料強制對齊至下一個word邊界 (force alignment at the next word boundary)，而且**不會佔任何的空間!!**。

透過這樣的方式，只要傳入 ***BUILD_BUG_ON_ZERO(e)*** 的 *e* 不為 0，其就會造成 **編譯出錯**。

若傳入 **BUILD_BUG_ON_ZERO(e)** 的 *e* 為 0，則只會宣告一個 不佔任何空間的 structure。經過 **sizeof()** 計算後回傳 **0** 的值。

同樣的 **BUILD_BUG_ON_NULL()** 則是將上述的結果轉成 **void**，因此只要傳入 **BUILD_BUG_ON_NULL(e)** 的 *e* **不為NULL**，其就會造成 **編譯出錯**。

若傳入 **BUILD_BUG_ON_NULL(e)** 的 ***e 為 NULL***，則只會宣告一個 **不佔任何空間的 structure**。經過 **sizeof()** 計算後仍為 **0**，再轉成一個**指向位址 0 的 void**。

而 Stackoverflow 上的回答也有提到，為何不直接使用 **assert()** 就好了？其答案也很清楚：

> **These macros implement a compile-time test, while assert() is a run-time test.**

也就是說這樣的機制是可以在 compile-time 的時候就發現問題，而 **assert()** 則必須等到 run-time 的時候才能發現問題。不過也就是因為 **BUILD_BUG_ON_ZERO()** 和 **BUILD_BUG_ON_NULL()** 只能使用在 compile-time 就可以找到 bug 的情況下，因此若是判斷式中有任何必須等到 run-time 才能得知的結果，就會造成錯誤：

```c
#include &lt;stdio.h&gt;
 
#define BUILD_BUG_ON_ZERO(e)    (sizeof(struct { int:-!!(e); }))
 
int main(void)
{
        int a = 2;
 
        BUILD_BUG_ON_ZERO(a == 2);
        BUILD_BUG_ON_ZERO(2 == 2);
 
        return 0;
}
```

第一次呼叫 **BUILD_BUG_ON_ZERO()** 由於傳入的判斷式中包含 run-time 才會得知其值的 *a*，因此會造成 compiler 錯誤的判斷，產生錯誤訊息：**error: bit-field '<anonymous>' width not an integer constant**。但第二次呼叫 **BUILD_BUG_ON_ZERO()** 由於判斷式展開後皆可在 compile-time 的時候得知其結果，因此就不會產生錯誤訊息。

所以在使用 **BUILD_BUG_ON_ZERO()** 或是 **BUILD_BUG_ON_NULL()** 的時候還是要注意其使用時機.....

---

不得不說Linux內用了許多非常漂亮的技巧... 不但可以在 compile-time 的時候就將錯誤顯示出來，若判斷式 (錯誤/不應發生的情況) 不成立亦不會造成任何空間的浪費!!
(**BUILD_BUG_ON_ZERO()** 的實際用法可以參考 {% post_link Linux-Kernel-ARRAY-SIZE %} 一文)
