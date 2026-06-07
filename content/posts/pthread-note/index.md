---
title: "pthread - Multi-thread 程式設計"
date: "2020-12-20"
categories: 
  - "程式設計"
---

pthread 是 POSIX 標準下的 Thread 規範，定義了一系列用於建立與操作 Thread 的 API。在 Windows 環境下，也有第三方（3rd-party）透過 Windows API 實作的 [pthreads-win32](http://sourceware.org/pthreads-win32/#download) 支援。因此，對於需要開發跨平台（Cross-platform）軟體的開發者來說，pthread 無疑是實作 Multi-thread 的首選。

# 建立 thread

透過 `pthread_create ()` 建立 child thread 並指定其所要執行的函數，main thread 與 child thread 將會平行執行，

在 main thread 中使用 `pthread_join ()` 來等待 child thread 結束，否則 main thread 將會阻塞在那，這個動作相當重要，若你不等待 child thread 結束， main thread 就直接結束的話， child thread 將無法成功執行完。

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

如果要將資料傳遞給 child thread 處理，child thread 處理完後，可以透過呼叫 `pthread_exit()` 來終止 child thread 並回傳資料給 main thread。

但這樣寫法容易造成 bug ，記憶體是在 child thread 裏面去分配來的，當 child thread 結束時，記憶體是不會被自動釋放的 （如果真的釋放了，你在 main thread 拿到的資料就會是垃圾...) ，必須要在 main thread 裏面去釋放記憶體，往往忘記釋放，就會形成 memory leak 的麻煩...

```cpp
void* child (void* data)
{
    int *input = (int *) data;
    int *result = (int*) malloc (sizeof (int)*1);
        result = input[0]++;
    pthread_exit ((void*) result);
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

通常由 main thread 來分配記憶體與釋放記憶體會是比較好的作法，如下面範例，將資料塞進一個 struct 裡，統一由 main thread 管理記憶體。

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

如果同時有多個 thread 會修改同一個變數，會發生什麼事呢？

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

程式碼中建立兩個 thread ，分別對變數 val 進行加 1 的動作，兩個 thread 加起來總共執行 6 次加 1 ，預期 main thread 中應該得到回傳值為 6 ，但以下結果卻顯示結果為 3 。

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

原因在於多個 thread 同時存取和操作資料，就有可能發生相互覆蓋共用資料的情況，在作業系統中會稱這樣的區塊為 critical section ，開發者必須保證 critical section 不會被多個並行來源同時操作，在 pthread 的使用場景中，即可以透過 mutex 來保護 critical section 。

保護 critical section 很重要觀念是，該保護的是資料或資源，而不是程式碼。

在操作 critical section 之前，如果 mutex 是沒有被上鎖的， thread 才能將 mutex 上鎖並順利進入 critical section ，其他 thread 因為無法上鎖 mutex 而無法進入 ，直到將他上鎖的 thread 解鎖後， 下一個 thread 才能夠進入 critical section 。

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

當希望一個 thread 能夠暫時 sleep ，等待某個條件完成後再醒來，就可以透過 condition variable 來協助，很經典的 producer-consumer problem 就可以透過 condition variable 協助。

假設便當店只有一位老闆，老闆不是再做便當就是再幫客人結帳，老闆每次可以做 6 個便當，為了確保便當新鮮，如果便當還剩下 2 個以上，就開始去幫客人結帳。

而客人會不斷來買便當，當便當只剩 2 個的時候，客人買完後，老闆就會知道該去做便當了 ，當老闆去做便當時候，客人就無法結帳了，只能等老闆有空。

這樣的情境就可以透過 condition variable 來模擬，老闆發現便當還有剩下，他應該到前台結帳時，透過 `pthread_cond_wait ()` 就能夠讓 producer thread 解鎖 mutex 並等待，當便當沒有了，其他 thread 透過 `pthread_cond_signal ()` 將 producer thread 喚醒， producer thread 雖然被喚醒，但他還是必須等到 mutex 可以由他上鎖，才會往下執行。

`pthread_cond_wait ()` 的一個優勢是該 thread 不會使用 CPU 時間，他會等待直到有人透過 `pthread_cond_signal ()` 喚醒他，對應到我的前述情境，老闆一旦不再做便當，就可以空出來去幫客人結帳。

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

如果沒有 mutex ，可能發生 race condition ，producer thread 在 consumer thread 對 condition variable 發出 signal 後，才去 wait 此 condition variable ，這種狀況下 producer thread 就會永遠 sleep ，我們以上面的例子拿掉 mutex ，並且在 wait 前刻意 sleep 來製造前述狀況，即可以發現 producer thread 沒有被喚醒。

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
        //pthread_mutex_lock (&mutex);
        while (counter > 0)
        {
            sleep (4);
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

semaphore 如同一個計數器，可以透過 `sem_post ()` 來增加這個計數器，透過 `sem_wait ()` 來確認計數器是否大於 0 ，若是，則可以消耗計數器並繼續往下執行，若計數器值為 0 ，則 thread 會被卡住直到計數器大於 0 。

看以下範例程式， main thread 負責指派工作，則被卡住的 thread 就能夠順利往下執行，semaphore 無法傳遞資料在參數中，通常會透過其他 global variable 來讓 child 能取得資料。

```cpp
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

sem_t semaphore;
int counter = 0;

void *child ()
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
Execute 0 times
Execute 1 times
Post 3 jobs
Execute 2 times
Execute 3 times
Execute 4 times
```

# Semaphore v.s. Condition Variable

看完 condition variable 和 semaphore 章節，不知道大家心中是否有疑惑，這兩種用法都能夠保護 shared resource ，同時間只能被單一 thread 操作 ，這兩種用法非常相似，該如何區分這兩種的使用狀況。

condition variable 透過 `pthread_cond_broadcast ()` 可以一次叫醒所有 sleep thread ，而你不需要知道有多少 thread ，這是 semaphore 做不到的。

透過 `pthread_cond_signal ()` 喚醒 sleep thread 時，若當時沒有任何 sleep thread ，那這個呼叫也不會產生任何效果。然而 semaphore 比較像是一個 counter ，當沒有等待的 thread 時，仍然能夠增加 semaphore ，一旦有新的 thread 出現 ，將不會被阻塞可以直接開始執行。

在大部份情況下，這兩種工具都可以適用，與其總結出什麼狀況下用 semaphore 或什麼狀況下用 condition variable ，不如清楚了解各自的特性，再依據自己的需要來決定該使用何種技術。

# Reference

1. [POSIX thread (pthread) libraries - YoLinux](http://www.yolinux.com/TUTORIALS/LinuxTutorialPosixThreads.html)
2. [understanding of pthread_cond_wait() and pthread_cond_signal() - stackoverflow](https://stackoverflow.com/questions/16522858/understanding-of-pthread-cond-wait-and-pthread-cond-signal)
3. [C 語言 pthread 多執行緒平行化程式設計入門教學與範例](https://blog.gtwang.org/programming/pthread-multithreading-programming-in-c-tutorial/)
4. [CS 365: Lecture 10: Condition Variables](https://ycpcs.github.io/cs365-spring2017/lectures/lecture10.html)
5. [pthread_cond_wait versus semaphore - stackoverflow](https://stackoverflow.com/a/108918/4545634)
6. [Conditional Variable vs Semaphore - stackoverflow](https://stackoverflow.com/questions/3513045/conditional-variable-vs-semaphore)
7. Linux man page
