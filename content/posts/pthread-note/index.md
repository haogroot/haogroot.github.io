---
title: "pthread - Multi-thread 程式設計"
date: "2020-12-20"
categories: 
  - "程式設計"
---

pthread 是 POSIX 標準下的 thread 規範，定義了一系列用於建立與操作 thread 的 API。在 Linux、macOS 等 Unix-like 系統中，pthread 是常見的 multi-thread 程式設計介面；在 Windows 環境下，也有第三方透過 Windows API 實作的 [pthreads-win32](http://sourceware.org/pthreads-win32/#download) 支援。因此，若程式主要面向 POSIX 環境，pthread 是實作 multi-thread 時很值得理解的一組 API。

# 建立 thread

透過 `pthread_create()` 可以建立 child thread，並指定該 thread 要執行的函數。建立成功後，main thread 與 child thread 會平行執行。

在 main thread 中使用 `pthread_join()` 來等待 child thread 結束。呼叫 `pthread_join()` 時，main thread 會阻塞在那裡，直到指定的 child thread 結束後才繼續往下執行。這個動作相當重要，若 main thread 沒有等待 child thread 就直接結束，整個 process 可能會跟著結束，child thread 也就無法成功執行完。

```cpp
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>

void* child (void* data)
{
    char *str = (char *) data;
    for (int i=0; i<3; i++)
    {
        printf ("%s\n", str);
        sleep (1);
    }
    pthread_exit (NULL);
}

int main () {
    pthread_t t;
    pthread_create (&t, NULL, child, "Child is running...");

    for (int i=0; i<3; i++)
    {
        printf ("Mom is running...\n");
        sleep (1);
    }

    pthread_join (t, NULL);
    printf ("child thread done.");
    return 0;
}
```

# 傳遞資料到 thread

如果要將資料傳遞給 child thread 處理，child thread 處理完後，可以透過呼叫 `pthread_exit()` 結束 child thread，並把結果回傳給 main thread。

不過這種寫法容易造成 bug。以下範例是在 child thread 裡配置記憶體，再把結果指標回傳給 main thread。當 child thread 結束時，這塊記憶體不會被自動釋放；如果忘記在 main thread 裡釋放，就會形成 memory leak。

```cpp
void* child (void* data)
{
    int *input = (int *) data;
    int *result = malloc (sizeof(int));

    *result = input[0] + 1;
    pthread_exit (result);
}

int main ()
{
    ...
    void *ret;
    pthread_join (t, &ret);
    int *result = (int *) ret;
    ...
    free (result);
}
```

通常由 main thread 統一分配與釋放記憶體會是比較好的做法。下面範例將輸入與輸出資料放在同一個 struct 裡，讓 main thread 負責管理資料生命週期。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

typedef struct my_data {
    int val;
    int result;
} my_data;

void* child (void* data)
{
    my_data* input = (my_data *) data;
    input->result = input->val * input->val;
    pthread_exit (NULL);
}

int main () {
    pthread_t t;
    my_data data;
    data.val = 3;
    pthread_create (&t, NULL, child, (void *) &data);

    pthread_join (t, NULL);

    printf ("child thread return %d.\n", data.result);
    return 0;
}
```

# Mutex 互斥鎖

如果同時有多個 thread 修改同一個變數，會發生什麼事呢？

觀察以下程式碼及結果：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

typedef struct my_data {
    int val;
} my_data;

void* child (void* data)
{
    my_data* input = (my_data *) data;
    for (int i=0; i<3; i++)
    {
        int tmp = input->val;
        sleep (1);
        input->val = tmp+1;
        printf ("input->val = %d\n", input->val);
    }
    pthread_exit (NULL);
}

int main () {
    pthread_t t1, t2;
    my_data data;
    data.val = 0;
    pthread_create (&t1, NULL, child, (void *) &data);
    pthread_create (&t2, NULL, child, (void *) &data);

    pthread_join (t1, NULL);
    pthread_join (t2, NULL);

    printf ("child thread return %d.\n", data.val);
    return 0;
}
```

程式碼中建立兩個 thread，分別對變數 `val` 進行加 1 的動作。兩個 thread 加起來總共執行 6 次加 1，預期最後結果應該是 6，但以下結果卻顯示為 3。

```bash
❯ ./a.out
input->val = 1
input->val = 1
input->val = 2
input->val = 2
input->val = 3
input->val = 3
child thread return 3.
```

原因在於多個 thread 同時存取並修改同一份資料時，可能發生彼此覆蓋更新結果的情況。這類需要避免並行存取的區塊稱為 critical section。開發者必須確保 critical section 不會被多個 thread 同時操作；在 pthread 的使用場景中，可以透過 mutex 來保護 critical section。

保護 critical section 時，一個重要觀念是：真正需要保護的是資料或資源，而不是程式碼本身。

在操作 critical section 之前，thread 需要先取得 mutex。如果 mutex 尚未被上鎖，thread 就能成功上鎖並進入 critical section；其他 thread 因為無法取得同一把 mutex，必須等待目前持有 mutex 的 thread 解鎖後，才有機會進入 critical section。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

typedef struct my_data {
    int val;
} my_data;

pthread_mutex_t mutex;

void* child (void* data)
{
    my_data* input = (my_data *) data;
    for (int i=0; i<3; i++)
    {
        pthread_mutex_lock (&mutex);
        int tmp = input->val;
        sleep (1);
        input->val = tmp+1;
        printf ("input->val = %d\n", input->val);
        pthread_mutex_unlock (&mutex);
    }
    pthread_exit (NULL);
}

int main () {
    pthread_t t1, t2;
    my_data data;
    data.val = 0;

    pthread_mutex_init (&mutex, 0);
    pthread_create (&t1, NULL, child, (void *) &data);
    pthread_create (&t2, NULL, child, (void *) &data);

    pthread_join (t1, NULL);
    pthread_join (t2, NULL);

    printf ("child thread return %d.\n", data.val);
    return 0;
}
```

# Condition Variable

當希望一個 thread 暫時 sleep，等待某個條件成立後再醒來，就可以透過 condition variable 來協助。經典的 producer-consumer problem 就很適合用 condition variable 實作。

假設便當店只有一位老闆，老闆不是在做便當，就是在幫客人結帳。老闆每次可以做 6 個便當；為了確保便當新鮮，如果便當還剩下 2 個以上，老闆就先去幫客人結帳。

客人會不斷來買便當。當便當只剩 2 個時，客人買完後，老闆就會知道該去做新的便當了。當老闆在做便當時，客人無法結帳，只能等老闆有空。

這樣的情境可以透過 condition variable 來模擬。當 producer thread 發現便當還有剩、應該先去前台結帳時，可以透過 `pthread_cond_wait()` 暫時等待。`pthread_cond_wait()` 會在進入等待狀態時釋放 mutex，讓其他 thread 可以繼續修改 shared data；當便當賣完後，其他 thread 再透過 `pthread_cond_signal()` 將 producer thread 喚醒。producer thread 雖然被喚醒，但仍然必須等到 mutex 可以由它重新上鎖，才會從 `pthread_cond_wait()` 返回並繼續往下執行。

`pthread_cond_wait()` 的一個優勢是等待中的 thread 不會 busy waiting 消耗 CPU 時間。實務上，`pthread_cond_wait()` 應該放在 `while` 迴圈中反覆檢查條件，因為 thread 被喚醒不代表條件一定成立，也可能發生 spurious wakeup。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

int counter;

pthread_mutex_t mutex;
pthread_cond_t cond;

void* producer (void* data)
{
    while (1) {
        pthread_mutex_lock (&mutex);
        while (counter > 0)
        {
            pthread_cond_wait (&cond, &mutex);
        }
        counter +=6;
        printf ("Boss makes 6 bento. Left %d bento\n", counter);
        printf ("Boss can settle bill.\n\n");
        pthread_mutex_unlock (&mutex);
    }
}

void *consumer (void* data)
{
    while (1) {
        sleep (2);
        pthread_mutex_lock (&mutex);
        if (counter == 2)
        {
            counter -= 2;
            printf ("Consumer buys 2 bento. Left %d bento.\n", counter);
            printf ("Call boss to make bento\n\n");
            pthread_cond_signal (&cond);
        }
        else
        {
            counter -= 2;
            printf ("Consumer buys 2 bento. Left %d bento.\n", counter);
        }
        pthread_mutex_unlock (&mutex);
    }
}

int main () {
    pthread_t t1, t2;

    counter = 2;
    pthread_mutex_init (&mutex, 0);
    pthread_cond_init (&cond, 0);
    pthread_create (&t2, NULL, consumer, NULL);
    pthread_create (&t1, NULL, producer, NULL);

    sleep (30);
    return 0;
}
```

執行結果：

```bash
❯ ./a.out
Consumer buys 2 bento. Left 0 bento.
Call boss to make bento

Boss makes 6 bento. Left 6 bento
Boss can settle bill.

Consumer buys 2 bento. Left 4 bento.
Consumer buys 2 bento. Left 2 bento.
Consumer buys 2 bento. Left 0 bento.
Call boss to make bento

Boss makes 6 bento. Left 6 bento
Boss can settle bill.

Consumer buys 2 bento. Left 4 bento.

...
```

## condition variable 一定要搭配 mutex

condition variable 本身不保護 shared data，它必須搭配 mutex 與一個明確的條件一起使用。原因是 `pthread_cond_signal()` 不會把通知「存起來」；如果 signal 發生時沒有任何 thread 正在等待，這次 signal 就不會留下後續效果。

因此，檢查條件與進入等待狀態必須放在同一把 mutex 保護之下。如果 producer thread 在 consumer thread 發出 signal 後才開始 wait，就可能錯過這次通知，造成 producer thread 永遠睡著。下面是刻意拿掉 mutex 的錯誤示範，用來說明 lost wakeup 的問題；實務上不要這樣呼叫 `pthread_cond_wait()`，因為沒有先 lock mutex 就呼叫它屬於 undefined behavior。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

int counter;

pthread_mutex_t mutex;
pthread_cond_t cond;

void* producer (void* data)
{
    while (1) {
        // 錯誤示範：沒有先 lock mutex。
        //pthread_mutex_lock (&mutex);
        while (counter > 0)
        {
            sleep (4);
            // 錯誤示範：pthread_cond_wait() 必須搭配已上鎖的 mutex。
            pthread_cond_wait (&cond, &mutex);
        }
        counter +=6;
        printf ("Boss makes 6 bento. Left %d bento\n", counter);
        printf ("Boss can settle bill.\n\n");
        //pthread_mutex_unlock (&mutex);
    }
}

void *consumer (void* data)
{
    while (1) {
        sleep (2);
        //pthread_mutex_lock (&mutex);
        if (counter == 2)
        {
            counter -= 2;
            printf ("Consumer buys 2 bento. Left %d bento.\n", counter);
            printf ("Call boss to make bento\n\n");
            pthread_cond_signal (&cond);
        }
        else
        {
            counter -= 2;
            printf ("Consumer buys 2 bento. Left %d bento.\n", counter);
        }
        //pthread_mutex_unlock (&mutex);
    }
}

int main () {
    pthread_t t1, t2;

    counter = 2;
    pthread_mutex_init (&mutex, 0);
    pthread_cond_init (&cond, 0);
    pthread_create (&t1, NULL, producer, NULL);
    pthread_create (&t2, NULL, consumer, NULL);

    sleep (30);
    return 0;
}
```

```bash
❯ ./a.out
Consumer buys 2 bento. Left 0 bento.
Call boss to make bento

Consumer buys 2 bento. Left -2 bento.
Consumer buys 2 bento. Left -4 bento.
Consumer buys 2 bento. Left -6 bento.
Consumer buys 2 bento. Left -8 bento.
Consumer buys 2 bento. Left -10 bento.
```

# semaphore 與 pthread

semaphore 如同一個計數器，可以透過 `sem_post()` 增加計數器，透過 `sem_wait()` 檢查計數器是否大於 0。若計數器大於 0，thread 可以消耗一次計數並繼續往下執行；若計數器值為 0，thread 會被阻塞，直到計數器再次大於 0。

以下範例中，main thread 透過 `sem_post()` 指派工作；原本被 `sem_wait()` 阻塞的 child thread 取得 semaphore 後，就能繼續往下執行。semaphore 本身只負責同步，不負責傳遞資料；如果 child thread 需要額外資料，通常會搭配 shared data、queue 或其他資料結構。

```cpp
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

sem_t semaphore;
int counter = 0;

void *child (void *data)
{
    for (int i=0; i<5; i++)
    {
        sem_wait (&semaphore);
        printf ("Counter = %d\n", ++counter);
    }
    pthread_exit (NULL);
}

int main ()
{
    counter = 0;
    sem_init (&semaphore, 0, 0);

    pthread_t t;
    pthread_create (&t, NULL, child, NULL);

    printf ("Post 2 jobs\n");
    sem_post (&semaphore);
    sem_post (&semaphore);
    sleep (3);


    printf ("Post 3 jobs\n");
    sem_post (&semaphore);
    sem_post (&semaphore);
    sem_post (&semaphore);

    pthread_join (t, NULL);
}
```

```bash
❯ ./a.out
Post 2 jobs
Counter = 1
Counter = 2
Post 3 jobs
Counter = 3
Counter = 4
Counter = 5
```

# Semaphore vs. Condition Variable

看完 condition variable 和 semaphore 章節，可能會產生一個疑問：這兩種工具看起來都可以讓 thread 等待某個事件發生，該如何區分它們的使用情境？

condition variable 通常搭配 mutex 使用，用來等待某個 shared state 滿足特定條件。它本身不保護 shared resource，真正負責保護 shared data 的是 mutex。condition variable 透過 `pthread_cond_broadcast()` 可以一次叫醒所有 waiting thread，而你不需要知道有多少 thread 正在等待，這是 semaphore 不適合直接表達的情境。

透過 `pthread_cond_signal()` 喚醒 waiting thread 時，若當時沒有任何 thread 正在等待，這個呼叫不會留下後續效果。然而 semaphore 比較像是一個 counter，當沒有等待的 thread 時，仍然能夠透過 `sem_post()` 增加 semaphore 的值；一旦有新的 thread 呼叫 `sem_wait()`，就可以消耗這個計數並繼續執行。

簡單來說，如果你想表達的是「某個狀態改變了，請重新檢查條件」，condition variable 會比較自然；如果你想表達的是「目前有 N 個資源或 N 次事件可以被消耗」，semaphore 會比較自然。與其硬背什麼狀況一定用 semaphore 或 condition variable，不如清楚了解各自的特性，再依據自己的需要來決定該使用何種技術。

# Reference

1. [POSIX thread (pthread) libraries - YoLinux](http://www.yolinux.com/TUTORIALS/LinuxTutorialPosixThreads.html)
2. [understanding of pthread_cond_wait() and pthread_cond_signal() - stackoverflow](https://stackoverflow.com/questions/16522858/understanding-of-pthread-cond-wait-and-pthread-cond-signal)
3. [C 語言 pthread 多執行緒平行化程式設計入門教學與範例](https://blog.gtwang.org/programming/pthread-multithreading-programming-in-c-tutorial/)
4. [CS 365: Lecture 10: Condition Variables](https://ycpcs.github.io/cs365-spring2017/lectures/lecture10.html)
5. [pthread_cond_wait versus semaphore - stackoverflow](https://stackoverflow.com/a/108918/4545634)
6. [Conditional Variable vs Semaphore - stackoverflow](https://stackoverflow.com/questions/3513045/conditional-variable-vs-semaphore)
7. Linux man page
