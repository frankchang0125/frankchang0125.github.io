---
title: 'Linux Kernel: ARRAY_SIZE()'
date: 2012-10-15 11:37:40
tags:
- C
- C++
- Linux Kernel
categories:
- [Linux Kernel, Tricks]
---

通常我們在 C 語言中取得陣列的元數個數可以透過下列的方式來計算：

```c
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]))

int arr[10];
int arr_size = ARRAY_SIZE(arr);
```

但如同 Jserv 大大在 [這篇文章](http://blog.linux.org.tw/~jserv/archives/001888.html) 中所提到：**ARRAY_SIZE()** 這樣的 macro 其實是陷阱重重...

<!-- more -->

因為 macro 本身沒辦法做型態檢查，只是單純的將值帶入並展開，而在 C 中我們常常會將指標和陣列混著使用。因此若是我們將指向該陣列的指標傳入，就會得到錯誤的計算結果。

如下面的程式：

```c
#include &lt;stdio.h&gt;

#define ARRAY_SIZE(arr)     (sizeof(arr) / sizeof(arr[0]))

int main(void)
{
        int a[10];
        int *a_ptr = a;

        printf("%d\n", ARRAY_SIZE(a));
        printf("%d\n", ARRAY_SIZE(a_ptr));

        return 0;
}
```

若傳入陣列 *a*，則結果會正確顯示 size 大小為 **10**，但若傳入的是指向陣列 *a* 的指標 *a_ptr*，則因為指標的在 32 位元作業系統上大小為 **4 bytes** (4 / 4) 的結果則會變成 **1**，而並不是我們所要的答案 **10**。

當然只要我們小心使用，這樣的問題其實是可以避免的。但我們常常有可能會將陣列透過指標的方式傳入某個 function 中，這樣的情況下我們就有可能會將指標誤用成陣列傳入 **ARRAY_SIZE()** 而得到錯誤的結果。當程式成長到一定的複雜度後，類似的bug就很有可能被忽略。

因此 Linux 在定義 **ARRAY_SIZE()** 時除了透過上述的方式來取得陣列元數個數外，還另外加上了**型態檢查**，以確保使用者所傳入的參數必須為陣列而非指標 (Linux中的 **ARRAY_SIZE()** 是被定義在：*include/linux/kernel.h*)，其定義如下：

```c
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]) + __must_be_array(arr))
```

其中在最尾端額外加了 **__must_be_array()** 的回傳值。**__must_be_array()** 這個 macro 是用來判斷所傳入的參數是否為一陣列 (定義在：*include/linux/compiler-gcc.h*)，其定義如下：

```c
/* &a[0] degrades to a pointer: a different type from an array */
#define __must_be_array(a) BUILD_BUG_ON_ZERO(__same_type((a), &(a)[0]))
```

在這邊 **__must_be_array()** 對所傳入的參數 **a** 做了一次**降級 (degrade)**，並將其當作第二個參數傳入 **__same_type()** 這個 macro。在這邊做降級的目的就是為了讓**陣列**轉成一個**指標**，但若所傳入的參數 *a* 是個**不應傳入的指標**，則這樣的降級轉換後的結果仍會是**相同的原指標**。

**__same_type()** 的回傳值則會傳入 **BUILD_BUG_ON_ZERO()**
(**BUILD_BUG_ON_ZERO()** 的說明可以參考：{% post_link Linux-Kernel-BUILD-BUG-ON-ZERO-BUILD-BUG-ON-NULL 'BUILD_BUG_ON_ZERO() / BUILD_BUG_ON_NULL()' %})。

**__same_type()** 的定義則如下 (定義在：*include/linux/compiler.h*)：

```c
/* Are two types/vars the same type (ignoring qualifiers)? */
#ifndef __same_type
# define __same_type(a, b) __builtin_types_compatible_p(typeof(a), typeof(b))
#endif
```

在這邊我們可以看到 **__same_type()** 呼叫了 GCC 的 built-in function：**__builtin_types_compatible_p()**：
(**__builtin_types_compatible_p()** 的定義可以參考 [GCC manual](http://gcc.gnu.org/onlinedocs/gcc-4.7.2/gcc/Other-Builtins.html#Other-Builtins) 的說明)；**__builtin_types_compatible_p()** 的定義可以參考 [GCC manual](http://gcc.gnu.org/onlinedocs/gcc-4.7.2/gcc/Other-Builtins.html#Other-Builtins) 的說明

> You can use the built-in function __builtin_types_compatible_p to determine whether two types are the same.
This built-in function returns 1 if the unqualified versions of the types type1 and type2 (which are types, not expressions) are compatible, 0 otherwise. The result of this built-in function can be used in integer constant expressions.
This built-in function ignores top level qualifiers (e.g., const, volatile). For example, int is equivalent to const int.
The type int[] and int[5] are compatible. On the other hand, int and char * are not compatible, even if the size of their types, on the particular architecture are the same. Also, the amount of pointer indirection is taken into account when determining similarity. Consequently, short * is not similar to short **. Furthermore, two types that are typedefed are considered compatible if their underlying types are compatible.
An enum type is not considered to be compatible with another enum type even if both are compatible with the same integer type; this is what the C standard specifies. For example, enum {foo, bar} is not similar to enum {hot, dog}.
You would typically use this function in code whose execution varies depending on the arguments' types. For example:

```c
    #define foo(x)                                                  \
      ({                                                           \
        typeof (x) tmp = (x);                                       \
        if (__builtin_types_compatible_p (typeof (x), long double)) \
          tmp = foo_long_double (tmp);                              \
        else if (__builtin_types_compatible_p (typeof (x), double)) \
          tmp = foo_double (tmp);                                   \
        else if (__builtin_types_compatible_p (typeof (x), float))  \
          tmp = foo_float (tmp);                                    \
        else                                                        \
          abort ();                                                 \
        tmp;                                                        \
      })
```

> Note: This construct is only available for C.

也就是說 **__builtin_types_compatible_p()** 會檢查所傳入的型態：*type_1* 和 *type_2* 是否相同：
  - 若 *type_1* 和 *type_2* 的型態相同，則會回傳 **1**。
  - 若 *type_1* 和 *type_2* 的型態不同，則會回傳 **0**。

此外，為了取得參數的型態，這邊還另外用到了另一個 GCC 的 extension：**typeof()**。**typeof()** 可以取得所傳入參數的型態，因此我們可以透過 **typeof()** 來宣告一個與所傳入參數一模一樣的新變數：
(**typeof()** 的定義同樣可以參考 [GCC manual](http://gcc.gnu.org/onlinedocs/gcc-4.7.2/gcc/Typeof.html#Typeof) 的說明)


```c
#include &lt;stdio.h&gt;
 
#define ARRAY_SIZE(arr)     (sizeof(arr) / sizeof(arr[0]))
 
int main(void)
{
        int a[10];
        typeof(a) b;
 
        printf("%d\n", ARRAY_SIZE(a));
        printf("%d\n", ARRAY_SIZE(b));
       
        return 0;
}
```

在此我們宣告一個與陣列 *a* 型態一模一樣的陣列 *b*，因此透過 **ARRAY_SIZE()** 計算陣列元數個數的結果都會是 **10**。

---

回到 **__same_type()**：

透過 **__builtin_types_compatible_p()** 和 **typeof()**，我們就可以知道所傳入的兩個參數型態是否相同：

  - 如果相同 (傳入 **ARRAY_SIZE()** 的參數為一不應傳入的指標)，則 **__builtin_types_compatible_p()** 就會回傳 **1**，再傳入 **BUILD_BUG_ON_ZERO()** 後就會造成編譯錯誤。

  - 但如果不同 (傳入 **ARRAY_SIZE()** 的參數為一正確的陣列)，則 **__builtin_types_compatible_p()** 就會回傳 **0**，再傳入 **BUILD_BUG_ON_ZERO()** 後得到的結果為 **0**，加回 **ARRAY_SIZE()** 後並不會影響其原先結果。

透過這樣的方式，我們便可在 compile-time 的時候就發現所傳入 **ARRAY_SIZE()** 的參數是否為一錯誤的指標，並可在編譯時期加以修正...

此外 Jserv 大大 [那篇文章](http://blog.linux.org.tw/~jserv/archives/001888.html) 下面的回應也有人提出了其他不同的 [作法](http://heaven.branda.to/~thinker/GinGin_CGI.py/show_id_doc/236)。雖然其原意是為了要避免使用 GCC extension 的，但最後發現原來 **typeof()** 也是一個 GCC extension，不過作法同樣可以作為參考。

---

Extra References:

- [sizeof 在語言層面的陷阱](http://blog.linux.org.tw/~jserv/archives/001876.html)
