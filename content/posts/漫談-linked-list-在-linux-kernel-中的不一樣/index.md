---
title: "漫談 linked list 在 linux kernel 中的不一樣"
date: "2019-12-11"
categories: 
  - "linux-kernel"
---

linked list 是大家在學資料結構一定會學到的，通常會將資料跟指標寫在同一個 struct，但在 linux kernel 內卻不是這樣使用的。

## Linux kernel 內的定義

一般來說 linked list 在 linux kernel 都以 `struct list_head` 來描述。

在 [tools/include/linux/types.h](https://github.com/torvalds/linux/blob/master/tools/include/linux/types.h) 可以見到其定義:

```c
struct list_head {
        struct list_head *next, *prev;
    };
```

這個 struct 沒有帶任何的 data，Kernel 用法會將它直接放到其他 struct 內，使得其他 struct 也可以擁有 linked list 的操作。

我們來看一個例子，在 [linux/net/netlabel/netlabel\_addrlist.h](https://github.com/torvalds/linux/blob/9c7db5004280767566e91a33445bf93aa479ef02/net/netlabel/netlabel_addrlist.h) 可以見到這樣用法

```c
    /**
     * struct netlbl_af6list - NetLabel IPv6 address list
     * @addr: IPv6 address
     * @mask: IPv6 address mask
     * @valid: valid flag
     * @list: list structure, used internally
     */
    struct netlbl_af6list {
        struct in6_addr addr;
        struct in6_addr mask;
        u32 valid;
        struct list_head list;
    };
```

> 疑~ 等等 ! 那這樣就算我們走訪整個 linked list，我們也存取不到 data，這樣我們就失去使用 linked list 的意義了!

一般在走訪到每個 list node ，透過指標就可以直接存取 data ，如以下作法:

```c
    ListNode *current = head;
    while(current != 0) {
        printf("%d", current->data);
        current = current->next;
    }
```

由於 kernel 並沒有把 data 跟 pointer 放在同樣的 struct，造成無法直接存取 data ，以下就來探討 kernel 是如何解決的問題。

## 如何走訪 linked list 並存取資料

kernel 提供 containerof 這個巨集來解決上述問題，以下我們就來拆解

```c
    #define container_of(ptr, type, member)                            \\
        __extension__({                                                \\
            const __typeof__(((type *) 0)->member) *__pmember = (ptr); \\
            (type *) ((char *) __pmember - offsetof(type, member));    \\
        })
```

- `offsetof(type, member)` 這個 macro 定義在標準函式庫裡，通過把 0 位址轉換為 type 型態的 pointer，然後去獲取該 struct 中的 member 的 pointer，也就是獲取了 member 與該 struct 起始點的 offset。
- `typeof` 可以用在 expression 或 type 上，此 macro 會回傳 argument 的 type，值得留意的是，使用 expression 作為 argument 上並不會真的執行這個 expression。

```c
    typeof (int *) m
    typeof (typeof (char *)[4]) n;
```

上述等同於宣告 int _m 和 char_ n\[4\] 一樣。

- （type \*）0 -> 這個 type 形態的 struct，他裡面有個成員叫做 member，我們利用 `typeof` 取得 member 的型態。
- 再用 `__pmember` 減去 offset，也就等於得到了該 type struct 的真實位址。
- 因此 `container_of` 我們可以理解為我們利用其成員來反求其母結構的位址
- `__ extension __` GCC uses the **extension** attribute when using the -ansi flag to avoid warnings in headers with GCC extensions.

透過 `container_of` 讓我們能夠在走訪 linked list 時候隨時都可以得到其所在之 struct 的位址，一但有了位址，就能夠自由的存取他的所有成員變數了。

Updated on _2022-07-27 08:04:21 星期三_
