# 01 — 线程基础接口：创建、终止、等待、分离、TLS

> 关联篇目：[第四章 02 线程概念](../04-operating-systems/02-threads.md) | [02 互斥锁](./02-mutex-locks.md)

---

## 1. 线程创建与终止

### 1.1 POSIX 线程（`pthread`）

```c
#include <pthread.h>

void* worker(void* arg) {
    int id = *(int*)arg;
    printf("Thread %d\n", id);
    return (void*)(intptr_t)(id * 2);  // 返回值
}

pthread_t tid;
int arg = 42;
pthread_create(&tid, NULL, worker, &arg);

// 终止方式：
// 1. return —— 正常返回
// 2. pthread_exit(NULL) —— 显式退出（不终止整个进程）
// 3. pthread_cancel(tid) —— 被其他线程取消
// 4. exit() —— ⚠️ 退出整个进程！
```

### 1.2 C++11 `std::thread`

```cpp
#include <thread>

void worker(int id) {
    std::cout << "Thread " << id << "\n";
}

std::thread t1(worker, 42);          // 函数指针 + 参数
std::thread t2([] {                  // lambda
    std::cout << "Lambda thread\n";
});

struct Task { void operator()() {} };
std::thread t3(Task{});              // 可调用对象
```

---

## 2. `join` vs `detach`

```cpp
std::thread t(worker, 1);

// join: 阻塞等待线程完成
t.join();   // 调用后 t 不再代表任何线程

// detach: 分离线程，后台运行
t.detach(); // 调用后 t 不再代表任何线程，线程结束时自动清理

// ⚠️ 必须在 join 和 detach 之间选一个
// 如果 std::thread 析构时仍 joinable → std::terminate!
if (t.joinable()) {
    t.join();  // 或 t.detach()
}
```

| | `join` | `detach` |
|------|------|------|
| 行为 | 阻塞等待线程结束 | 分离，线程独立运行 |
| 资源回收 | 调用后线程资源被回收 | 线程结束时自动回收 |
| 风险 | 可能死锁（互相等待） | 访问已销毁的局部变量 → UB |

---

## 3. 线程局部存储（TLS）

每个线程拥有独立的变量副本。

```cpp
// C++11
thread_local int counter = 0;

void increment() {
    counter++;  // 每个线程各自累加自己的 counter
}

// C / GCC
__thread int counter = 0;

// pthread
pthread_key_t key;
pthread_key_create(&key, destructor);  // destructor 在线程退出时清理
pthread_setspecific(key, value);
void* val = pthread_getspecific(key);
```

### 3.1 TLS 的实现原理

```
TLS 变量通过 fs/gs 段寄存器 + 偏移量访问:
mov eax, [fs:offset_of_counter]

每个线程创建时，TLS 数据块随线程栈一起分配
fs/gs 基址指向当前线程的 TLS 数据块
```

---

## 4. 线程参数传递注意事项

```cpp
// ❌ 危险：传递局部变量的引用/指针
void dangerous() {
    int local = 42;
    std::thread t([](int& x) { x = 100; }, std::ref(local));
    t.detach();  // 线程还在跑，local 已销毁 → 悬空引用！
}

// ✅ 安全：按值传递
void safe() {
    int local = 42;
    std::thread t([](int x) { /* x 是拷贝，安全 */ }, local);
    t.join();
}

// ✅ 传递引用：确保生命周期
int shared = 42;
std::thread t([](int& x) { x = 100; }, std::ref(shared));
t.join();  // 确保线程结束前 shared 有效
// 或使用 shared_ptr 延长生命周期
```

---

## 5. 线程 ID 与 CPU 亲和性

```cpp
#include <thread>
#include <pthread.h>

std::thread::id tid = std::this_thread::get_id();

// 绑定到指定 CPU
cpu_set_t cpuset;
CPU_ZERO(&cpuset);
CPU_SET(2, &cpuset);  // 绑定到 CPU 2
pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);

// 设置线程名称（调试用）
pthread_setname_np(pthread_self(), "worker-0");
```

---

## 小结

| 概念 | C++ 接口 | 关键 |
|------|----------|------|
| 创建 | `std::thread t(f, args...)` | 参数默认按值拷贝 |
| 等待 | `t.join()` | 阻塞到线程结束 |
| 分离 | `t.detach()` | 线程独立运行，自动回收 |
| 检查 | `t.joinable()` | 析构前必须处理 |
| TLS | `thread_local` | 每线程独立副本 |

下一篇 [02 互斥锁与 RAII 锁包装](./02-mutex-locks.md)。
