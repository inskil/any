# 生产者消费者代码注解

## 生产者

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <semaphore.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <sys/shm.h>

#define TYPE int     //根据MAXNUM敲定类型
#define MAXNUM 500     // < intmax
#define PRODUCTERNUM 3 //线程数量
#define TYPESIZE sizeof(TYPE)
#define MAXSPACES 10 //缓冲区大小
#define BUFSIZE MAXSPACES *TYPESIZE + 66
#define KEY (key_t)123456

sem_t *mlock, *products, *spaces;
int buff, outfile, pids[PRODUCTERNUM];
TYPE number[1] = {0}; //产品标号
int *queue; //产品队列

// 生产者
void Producer()
{
    int i;
    while ((int)number[0] < MAXNUM)
    {
        // 生产一个产品 item;
        number[0]++;
        // 空闲缓存资源
        sem_wait(spaces);
        // 互斥信号量，上锁
        sem_wait(mlock);
        // 将item放到空闲缓存中;
        printf("Producer put %d \n", number[0]);
        // 找到第一个空闲区并置入产品
        for (i = 0; i < MAXSPACES; i++)
        {
            if (queue[i] == 0)
            {
                queue[i] = number[0];
                break;
            }
        }
        // 释放锁
        sem_post(mlock);
        // 产品资源
        sem_post(products);
    }
}

int main()
{
    int i,shm_id;

    // 产品标号初始化为0
    *(number) = 0;
    
    // 创建共享内存（生产者和消费者共享内存的KEY值相同）
    shm_id = shmget(KEY,BUFSIZE,IPC_CREAT|IPC_EXCL|0600);
    // 错误处理
    if (shm_id == -1)
    {
        perror("shmget error");
        return -1;
    }

    // 产品队列，生产者可以每次向队列尾部添加产品
    queue = (int *)shmat(shm_id, NULL, 0);
    // 出错处理
    if (queue < 0)
    {
        perror("shmat error");
        exit(0);
    }
    // 初始化产品队列为0
    for (i = 0; i < MAXSPACES; i++)
    {
        queue[i] = 0;
    }

    // 清除之前的信号量
    sem_unlink("mlock");
    sem_unlink("products");
    sem_unlink("spaces");

    
   /*信号量初始化
   	*mlock为二值信号量，用作锁
	*products信号量代表产品，初始化为0，如果有产品消费者便可以消费产品
	*sapces信号量代表代表空闲区，初始化为缓冲区大小
	*如果有空闲区生产者便可以生产产品
	*/
    products = sem_open("products", O_CREAT, 0644, 0);
    spaces = sem_open("spaces", O_CREAT, 0644, MAXSPACES);
    mlock = sem_open("mlock", O_CREAT, 0644, 1);

    // 生产者循环
    Producer();
    sleep(2);

    // 释放信号量
    sem_unlink("mlock");
    sem_unlink("products");
    sem_unlink("spaces");

    exit(0);
}

```

## 消费者

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <semaphore.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <sys/shm.h>

#define TYPE int       // 根据MAXNUM敲定类型
#define MAXNUM 500     // < intmax
#define PRODUCTERNUM 3 // 线程数量（应该是消费者，写错了懒得改hhh）
#define TYPESIZE sizeof(TYPE)
#define MAXSPACES 10 // 缓冲区大小
#define BUFSIZE MAXSPACES *TYPESIZE + 66
#define KEY (key_t)123456

sem_t *mlock, *products, *spaces;
int buff, outfile, pids[PRODUCTERNUM];
int *queue;

// 从缓冲区读入并删除已读的部分（模拟获得一个产品）
TYPE mread()
{
    int re = queue[0], i;
    for (i = 1; i < MAXSPACES ; i++)
    {
        queue[i - 1] = queue[i];
        
    }
    queue[MAXSPACES - 1] = 0;
    return re;
}

// 消费者
void Consumer()
{
    // 获取消费者进程号，用于输出
    int mypid = getpid();
    TYPE dig = 0;
    while (dig < MAXNUM)
    {
        // 等产品
        sem_wait(products);
        // 等互斥锁
        sem_wait(mlock);
        // 从缓存区取出一个赋值dig（相当于获取产品）
        TYPE dig = mread();
        printf("Consumer:pid=%d, index=%d\n", mypid, dig);
        // 释放锁
        sem_post(mlock);
        // 消费产品item;
        sem_post(spaces);
    }
}

int main()
{
    int i, shm_id;

    // 创建共享内存（和生产者使用相同的共享内存KEY值相同）
    shm_id = shmget(KEY, BUFSIZE, IPC_CREAT | 0600);
    if (shm_id == -1)
    {
        perror("shmget error");
        return -1;
    }
    queue = (int *)shmat(shm_id, NULL, 0);
    if (queue < 0)
    {
        perror("shmat error");
        exit(0);
    }

    // 信号量初始化（和生产者使用相同信号量）
    products = sem_open("products", O_RDWR);
    spaces = sem_open("spaces", O_RDWR);
    mlock = sem_open("mlock", O_RDWR);

    // 消费者循环
    Consumer();

    sleep(2);

    // 释放信号量
    sem_unlink("mlock");
    sem_unlink("products");
    sem_unlink("spaces");

    // 释放共享空间
    shmdt((void *)queue);
    if (shmctl(shm_id, IPC_RMID, 0) < 0)
    {
        perror("shmctl error");
    }
    exit(0);
}

```



