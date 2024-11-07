---
layout: post
title:  "简析Cocos2d-x中的纹理异步加载"
date:   2018-07-30 16:25:18 +800
category: Cocos2d-x
---

- [1. 说明](#1-说明)
- [2. 开始之前先了解一下C/C++一些关于thread的基本知识吧](#2-开始之前先了解一下cc一些关于thread的基本知识吧)
- [3. 下面开始直接开始贴Cocos2d-x的源码（Cocos2d-x 3.17）了](#3-下面开始直接开始贴cocos2d-x的源码cocos2d-x-317了)

## 1. 说明

最近在看项目代码，看到异步加载纹理的时候正好看到Cocos2d-x底层源码那里去了，这里正好写一篇文件来记录加深一下印象吧，照理说都做这么久了现在才真正开始认真看源码也是浪费了太多的时间了啊。

在下面贴上之前在网上看到的一段话，很形象的解释了同步和异步的区别：

![00_difference_between_syn_and_asy](/assets/images/2018/2018-07-30-AddImageAsyncInCocos2dx/00_difference_between_syn_and_asy.jpg)

## 2. 开始之前先了解一下C/C++一些关于thread的基本知识吧

- [std::thread](https://en.cppreference.com/w/cpp/thread/thread)

先看下[cppreference](https://en.cppreference.com/w/)上对`std::thread`的解释：

```en
The class thread represents a single thread of execution. Threads allow multiple functions to execute concurrently.
线程类表示一个单独执行的thread，多个thread允许同时执行多个functions
```

看一个`std::thread`使用的例子：

```cpp
#include <iostream>
#include <thread>
#include <chrono>

void threadFunc1() {
  //阻止当前线程执行2s
  std::this_thread::sleep_for(std::chrono::seconds(2));
  std::cout << "thread function 1" << std::endl;
}

void threadFunc2(int a, int b) {
  std::cout << a << " thread function 2 " << b << std::endl;
}

int main() {
  //创建一个新的线程，回调到threadFunc1里
  std::thread t1(&threadFunc1);
  //创建一个新的线程，回调到threadFunc2里，可以传递参数
  std::thread t2(&threadFunc2, 1, 2);

  t1.join();//等待threadFunc1线程执行完以后，主线程才可以继续执行下去，此时主线程会释放掉执行完后子线程threadFunc1的资源

  t2.detach();//将子线程从主线程里分离，子线程执行完成后会自己释放掉资源; 分离后的线程，主线程将对它没有控制权

  return 0;
}
```

最后输出：

```cpp
1 thread function 2 2
thread function 1
```

- [std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)

```en
The mutex class is a synchronization primitive that can be used to protect shared data from being simultaneously accessed by multiple threads.

mutex类是一个同步原语(synchronization primitive)，可以被用来保护共享数据以免被多个线程同时访问。

1.一个线程自从它成功地lock()或try_lock()起，它就拥有了一个mutex，直到它调用unlock();

2.当一个线程拥有mutex时，所有其他调用了lock()的线程都会阻塞住，而调用了try_lock()的线程都会返回得到try_lock()的返回值为false;

3.一个线程在调用lock()或try_lock()之前不可能拥有mutex
```

下面是一个mutex的例子：

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex mtx;

void printThreadId(int id) {
    mtx.lock();
    std::cout << "thread id # " << id << std::endl;
    mtx.unlock();
}

int main(void) {
    const int length = 10;
    std::thread threads[length];
    for (int i = 0; i < length; ++i) {
        threads[i] = std::thread(printThreadId, i + 1);
    }
    for (auto &th : threads) {
        th.join();
    }
    system("pause");
    return 0;
}
```

输出：

```cpp
thread id # 1
thread id # 2
thread id # 3
thread id # 4
thread id # 5
thread id # 6
thread id # 7
thread id # 8
thread id # 9
thread id # 10
```

但是如果去掉mutex锁的话：

```cpp
void printThreadId(int id) {
    // mtx.lock();
    std::cout << "thread id # " << id << std::endl;
    // mtx.unlock();
}
```

输出会乱掉：

```cpp
thread id # thread id # thread id # thread id # 10thread id #
8
thread id # 7thread id # 96
thread id # 2

thread id # 5

thread id # 3
4
1
```

这是因为没有了mutex锁的话，多个线程会同时调用相同的输出语句，导致最后输出错乱。所以mutex的作用就是：多个线程共享使用相同的数据时，当其中一个线程在使用数据时，防止其他线程使用数据而造成数据错误。lock()或者try_lock()后就代表当前线程拥有了一个mutex，其他线程只有在unlock()之后才能成功调用共享数据。

**Note:**

```en
std::mutex is usually not accessed directly: std::unique_lock, std::lock_guard, or std::scoped_lock (since C++17) manage locking in a more exception-safe manner.

std::mutex一般不会被直接使用，一般使用std::unique_lock，std::lock_guard，或者std::scoped_lock(C++17后)这些更安全的方式来管理锁定。
```

- [std::unique_lock](https://en.cppreference.com/w/cpp/thread/unique_lock)

```en
The class unique_lock is a general-purpose mutex ownership wrapper allowing deferred locking, time-constrained attempts at locking, recursive locking, transfer of lock ownership, and use with condition variables.

类 unique_lock 是通用互斥包装器，允许延迟锁定、锁定的有时限尝试、递归锁定、所有权转移和与条件变量一同使用。
```

构造函数：

```cpp
unique_lock() noexcept; // (1)(since C++11)
unique_lock( unique_lock&& other ) noexcept;// (2)(since C++11)
explicit unique_lock( mutex_type& m );// (3)(since C++11)
unique_lock( mutex_type& m, std::defer_lock_t t ) noexcept;// (4)(since C++11)
unique_lock( mutex_type& m, std::try_to_lock_t t );// (5)(since C++11)
unique_lock( mutex_type& m, std::adopt_lock_t t );// (6)(since C++11)
template< class Rep, class Period >
unique_lock( mutex_type& m,
             const std::chrono::duration<Rep,Period>& timeout_duration );// (7)(since C++11)
template< class Clock, class Duration >
unique_lock( mutex_type& m,
             const std::chrono::time_point<Clock,Duration>& timeout_time );// (8)(since C++11)
```

上述构造函数的描述：

```en
Constructs a unique_lock, optionally locking the supplied mutex.
1) Constructs a unique_lock with no associated mutex.
构造无关联互斥的 unique_lock 。
2) Move constructor. Initializes the unique_lock with the contents of other. Leaves other with no associated mutex.
移动构造函数。以 other 的内容初始化 unique_lock 。令 other 无关联互斥。
3-8) Constructs a unique_lock with m as the associated mutex. Additionally:
    3) Locks the associated mutex by calling m.lock(). The behavior is undefined if the current thread already owns the mutex except when the mutex is recursive.
    通过调用 m.lock() 锁定关联互斥。若当前线程已占有互斥则行为未定义，除非互斥是递归的。
    4) Does not lock the associated mutex.
    不锁定关联互斥。
    5) Tries to lock the associated mutex without blocking by calling m.try_lock(). The behavior is undefined if the current thread already owns the mutex except when the mutex is recursive.
    通过调用 m.try_lock() 尝试锁定关联互斥而不阻塞。若当前线程已占有互斥则行为未定义，除非互斥是递归的。
    6) Assumes the calling thread already owns m.
    假定调用方线程已占有 m 。
    7) Tries to lock the associated mutex by calling m.try_lock_for(timeout_duration). Blocks until specified timeout_duration has elapsed or the lock is acquired, whichever comes first. May block for longer than timeout_duration.
    通过调用 m.try_lock_for(timeout_duration) 尝试锁定关联互斥。阻塞直至经过指定的 timeout_duration 或获得锁，之先到来者。可能阻塞长于 timeout_duration 。
    8) Tries to lock the associated mutex by calling m.try_lock_until(timeout_time). Blocks until specified timeout_time has been reached or the lock is acquired, whichever comes first. May block for longer than until timeout_time has been reached.
    通过调用 m.try_lock_until(timeout_time) 尝试锁定关联互斥。阻塞直至抵达指定的 timeout_time 或获得锁，之先到来者。可能阻塞长于抵达 timeout_time 。
```

参数：

```en
other               —— 用以初始化状态的另一 unique_lock
m                   —— 与锁关联且可选的获得所有权的互斥
t                   —— 用于选择拥有不同锁定策略的构造函数的标签参数
timeout_duration    —— 要阻塞的最大时长
timeout_time        —— 要阻塞到的最大时间点
```

- [std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard)

```en
The class lock_guard is a mutex wrapper that provides a convenient RAII-style mechanism for owning a mutex for the duration of a scoped block.

When a lock_guard object is created, it attempts to take ownership of the mutex it is given. When control leaves the scope in which the lock_guard object was created, the lock_guard is destructed and the mutex is released.

The lock_guard class is non-copyable.

类 lock_guard 是互斥封装器，为在作用域块期间占有互斥提供便利 RAII 风格机制。

创建 lock_guard 对象时，它试图接收给定互斥的所有权。控制离开创建 lock_guard 对象的作用域时，销毁 lock_guard 并释放互斥。

lock_guard 类不可复制。
```

代码示例：

```cpp
#include <iostream>
#include <thread>
#include <mutex>

int g_i = 0;
std::mutex g_mtx;// protects g_i

void safeIncrement() {
    std::lock_guard<std::mutex> lock(g_mtx);
    ++g_i;
    std::cout << std::this_thread::get_id() << ": " << g_i << std::endl;
    // g_i_mutex is automatically released when lock
    // goes out of scope
}

int main(void) {
    const int length = 10;
    std::thread threads[length];
    for (int i = 0; i < length; ++i) {
        threads[i] = std::thread(safeIncrement);
    }
    for (auto &th : threads) {
        th.join();
    }
    std::cout << "main: " << g_i << std::endl;
    system("pause");
    return 0;
}
```

输出：

```cpp
9708: 1
7340: 2
9792: 3
10140: 4
2888: 5
8016: 6
7952: 7
5464: 8
4932: 9
9100: 10
main: 10
```

感觉lock_guard其实是帮我们封装好lock和unlock，全部都是自动完成的，用起来比较方便，但是没有unique_lock灵活。

- [std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)

```en
The condition_variable class is a synchronization primitive that can be used to block a thread, or multiple threads at the same time, until another thread both modifies a shared variable (the condition), and notifies the condition_variable.

condition_variable 类是同步原语，能用于阻塞一个线程，或同时阻塞多个线程，直至另一线程修改共享变量（条件）并通知 condition_variable 。

试图修改共享变量(条件)的线程必须满足以下条件：
    1.拥有互斥mutex(通常是std::lock_guard);
    2.在保有锁时进行修改;
    3.在std::condition_variable上执行notify_one()或notify_all()(不需要为通知保有锁);

即使共享变量是原子的，也必须在互斥下修改它，以正确地发布修改到等待的线程

任何有意在 std::condition_variable 上等待的线程必须：
    1.获得 std::unique_lock<std::mutex> ，在与用于保护共享变量者相同的互斥上;
    2.执行 wait() 、 wait_for() 或 wait_until() ，等待操作自动释放互斥，并悬挂线程的执行;
    3.condition_variable 被通知时，时限消失或虚假唤醒发生，线程被唤醒，且自动重获得互斥。之后线程应检查条件，若唤醒是虚假的，则继续等待;

std::condition_variable 只可与 std::unique_lock<std::mutex> 一同使用,这种限制可以在某些平台上实现最大效率。

condition_variable 允许 wait() 、 wait_for() 、 wait_until() 、 notify_one() 及 notify_all() 成员函数的同时调用。
```

- wait

    当前线程调用wait()后将被阻塞，直到另外某个线程调用notify_one()或者notify_all()唤醒当前线程。该函数会自动调用std::mutex的unlock()释放锁，使得其他被阻塞在锁竞争上的线程得以继续执行。一旦当前线程获得通知，wait函数也会自动调用std::mutex的lock()。wait()分为无条件被阻塞和带条件的被阻塞两种。

    无条件被阻塞：调用该函数前，当前线程应该已经对`unique_lock<mutex>` lck完成了加锁。所有使用同一个条件变量的线程必须在wait函数中使用同一个`unique_lock<mutex>`。该wait函数内部会自动调用lck.unlock()对互斥锁解锁，使得其他被阻塞在互斥锁上的线程恢复执行。使用本函数被阻塞的当前线程在获得通知(notified，通过别的线程调用 notify_*系列的函数)而被唤醒后，wait()函数恢复执行并自动调用lck.lock()对互斥锁加锁。

    带条件的被阻塞：wait函数设置了谓词(Predicate)，只有当pred条件为false时调用该wait函数才会阻塞当前线程，并且在收到其它线程的通知后只有当pred为true时才会被解除阻塞。因此，等效于:

    ```cpp
    while (!pred()) {
        wait(lock);
    }
    ```

- wait_for

    与wait()类似，只是wait_for可以指定一个时间段，在当前线程收到通知或者指定的时间超时之前，该线程都会处于阻塞状态。而一旦超时或者收到了其它线程的通知，wait_for返回，剩下的步骤和wait类似。

- wait_until

    与wait_for类似，只是wait_until可以指定一个时间点，在当前线程收到通知或者指定的时间点超时之前，该线程都会处于阻塞状态。而一旦超时或者收到了其它线程的通知，wait_until返回，剩下的处理步骤和wait类似。

- notify_all

    唤醒所有的wait线程，如果当前没有等待线程，则该函数什么也不做。

- notify_one

    唤醒某个wait线程，如果当前没有等待线程，则该函数什么也不做；如果同时存在多个等待线程，则唤醒某个线程是不确定的(unspecified)。

代码例子：

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::mutex m_mtx;
std::condition_variable m_cv;
int i = 0;

void wait() {
    std::unique_lock<std::mutex> lk(m_mtx);
    std::cout << "Waiting...\n";
    // 等待其他线程的通知，并且后面function的返回值为true才会关闭阻塞
    m_cv.wait(lk, [] { return i == 1; });
    std::cout << "finished waiting, i: " << i << "\n";
}

void signals() {
    // 线程sleep 2s
    std::this_thread::sleep_for(std::chrono::seconds(2));
    // 加锁
    std::lock_guard<std::mutex> lk(m_mtx);
    i = 1;
    std::cout << "Notifying...\n";
    // 通知wait的线程
    m_cv.notify_all();
}

int main(void) {
    std::thread t1(wait);
    std::thread t2(signals);
    t1.join();
    t2.join();
    return 0;
}
```

输出：

```cpp
Waiting...
// 2s后
Notifying...
finished waiting, i: 1
```

## 3. 下面开始直接开始贴Cocos2d-x的源码（Cocos2d-x 3.17）了

```cpp
auto textureCache = Director::getInstance()->getTextureCache();
textureCache->addImageAsync("Images/background1.jpg", CC_CALLBACK_1(TextureAsync::imageLoaded, this));
```

通过单例`Director`类获取`TextureCache`类，然后调用`addImageAsync`方法来异步加载纹理，

```cpp
void TextureCache::addImageAsync(const std::string &path, const std::function<void(Texture2D*)>& callback) {
    addImageAsync( path, callback, path );
}
```

可以看到`addImageAsync`方法需要2个参数，1个是文件路径，另1个是加载成功后的回调函数，下面看`addImageAsync`函数：

```cpp
void TextureCache::addImageAsync(const std::string &path, const std::function<void(Texture2D*)>& callback, const std::string& callbackKey) {
    Texture2D *texture = nullptr;

    std::string fullpath = FileUtils::getInstance()->fullPathForFilename(path);

    auto it = _textures.find(fullpath);
    if (it != _textures.end())
        texture = it->second;

    if (texture != nullptr)
    {
        if (callback) callback(texture);
        return;
    }

    // check if file exists
    if (fullpath.empty() || !FileUtils::getInstance()->isFileExist(fullpath)) {
        if (callback) callback(nullptr);
        return;
    }

    // lazy init
    if (_loadingThread == nullptr)
    {
        // create a new thread to load images
        _needQuit = false;
        _loadingThread = new (std::nothrow) std::thread(&TextureCache::loadImage, this);
    }

    if (0 == _asyncRefCount)
    {
        Director::getInstance()->getScheduler()->schedule(CC_SCHEDULE_SELECTOR(TextureCache::addImageAsyncCallBack), this, 0, false);
    }

    ++_asyncRefCount;

    // generate async struct
    AsyncStruct *data =
      new (std::nothrow) AsyncStruct(fullpath, callback, callbackKey);

    // add async struct into queue
    _asyncStructQueue.push_back(data);
    std::unique_lock<std::mutex> ul(_requestMutex);
    _requestQueue.push_back(data);
    _sleepCondition.notify_one();
}
```

首先获取资源完整路径：

```cpp
std::string fullpath = FileUtils::getInstance()->fullPathForFilename(path);
```

根据完整资源路径`path`在已经加载的map中找这个资源是否已经加载，如果已经加载就直接调用回调函数`callback`传递已经加载的`texture`

```cpp
auto it = _textures.find(fullpath);
if (it != _textures.end())
    texture = it->second;

if (texture != nullptr)
{
    if (callback) callback(texture);
    return;
}
```

检查资源是否存在，不存在的话回调`nullptr`

```cpp
if (fullpath.empty() || !FileUtils::getInstance()->isFileExist(fullpath)) {
    if (callback) callback(nullptr);
    return;
}
```

初始化一个新的子线程`_loadingThread`，调用对象为`loadImage`函数，字段`bool _needQuit`用来控制是否退出，当调用`Director::end()`的时候会通过调用`Director`类中`destroyTextureCache()`方法调用`TextureCache::waitForQuit()`方法而把这个变量置为`true`

```cpp
if (_loadingThread == nullptr) {
    // create a new thread to load images
    _needQuit = false;
    _loadingThread = new (std::nothrow) std::thread(&TextureCache::loadImage, this);
}
```

下面是子线程`_loadingThread`调用的`loadImage`函数：

```cpp
void TextureCache::loadImage() {
    // 初始化一个新的AsyncStruct结构体asyncStruct
    AsyncStruct *asyncStruct = nullptr;
    while (!_needQuit) {
        // 子线程加互斥_requestMutex锁
        std::unique_lock<std::mutex> ul(_requestMutex);
        // pop an AsyncStruct from request queue
        // 如果_requestQueue双端队列为空则asyncStruct为nullptr，不为空则获取_requestQueue双端队列的首元素，并移除_requestQueue的首元素
        if (_requestQueue.empty()) {
            asyncStruct = nullptr;
        } else {
            asyncStruct = _requestQueue.front();
            _requestQueue.pop_front();
        }

        // 当asyncStruct为nullptr时，如果_needQuit为true即跳出while循环，否则此子线程阻塞等待主线程的通知(这个时候再回去看看主线程在干什么=。=)
        if (nullptr == asyncStruct) {
            if (_needQuit) {
                break;
            }
            // 主线程在创建了新的数据结构体asyncStruct并添加到_requestQueue后会通过_sleepCondition.notify_one()来通知此子线程后，因为continue会从新执行一次while循环
            _sleepCondition.wait(ul);
            continue;
        }
        ul.unlock();

        // asyncStruct不为nullptr时，通过initWithImageFileThreadSafe()加载纹理数据
        asyncStruct->loadSuccess = asyncStruct->image.initWithImageFileThreadSafe(asyncStruct->filename);

        // ETC1 ALPHA supports特殊处理.
        if (asyncStruct->loadSuccess && asyncStruct->image.getFileType() == Image::Format::ETC && !s_etc1AlphaFileSuffix.empty()) {     // check whether alpha texture exists & load it
            auto alphaFile = asyncStruct->filename + s_etc1AlphaFileSuffix;
            if (FileUtils::getInstance()->isFileExist(alphaFile))
                asyncStruct->imageAlpha.initWithImageFileThreadSafe(alphaFile);
        }
        // 加载完image数据后再把结构体asyncStruct添加到双端队列_responseQueue中(最后再来看看addImageAsyncCallBack()函数，这个函数通过刚开始创建的计时器每一帧都在调用，用来处理此子线程修改的双端队列_responseQueue中的数据，函数往下)
        _responseMutex.lock();
        _responseQueue.push_back(asyncStruct);
        _responseMutex.unlock();
    }
}
```

加载资源数量`_asyncRefCount`为0时创建计时器，每一帧调用`addImageAsyncCallBack`函数，然后`_asyncRefCount`自增1：

```cpp
if (0 == _asyncRefCount) {
    Director::getInstance()->getScheduler()->schedule(CC_SCHEDULE_SELECTOR(TextureCache::addImageAsyncCallBack), this, 0, false);
}

++_asyncRefCount;
```

结构体AsyncStruct定义如下：

```cpp
struct TextureCache::AsyncStruct
{
public:
    AsyncStruct
    ( const std::string& fn,const std::function<void(Texture2D*)>& f,
      const std::string& key )
      : filename(fn), callback(f),callbackKey( key ),
        pixelFormat(Texture2D::getDefaultAlphaPixelFormat()),
        loadSuccess(false)
    {}

    std::string filename;
    std::function<void(Texture2D*)> callback;
    std::string callbackKey;
    Image image;
    Image imageAlpha;
    Texture2D::PixelFormat pixelFormat;
    bool loadSuccess;
};
```

创建异步加载结构体AsyncStruct(**此时子线程`_loadingThread`调用的函数`loadImage`中因为从`_requestQueue`中获取不到是否有新的纹理需要加载，所以在等待主线程的通知**)：

```cpp
// generate async struct
// 构造函数初始化三个参数：资源路径path、加载完成后的回调callback、callbackKey，其中path和callbackKey其实都是资源路径string，callbackKey在unbindImageAsync中用来解绑加载的函数回调
AsyncStruct *data = new (std::nothrow) AsyncStruct(fullpath, callback, callbackKey);
```

把异步加载数据`AsyncStruct`结构体添加到异步结构体双端队列`std::deque<AsyncStruct*> _asyncStructQueue`中，并且在主线程添加互斥`std::mutex _requestMutex`，把异步加载数据`AsyncStruct`结构体添加到请求双端队列`std::deque<AsyncStruct*> _requestQueue`中，并通过`std::condition_variable _sleepCondition`通知此时正在wait的子线程`_loadingThread`(这个时候再回去看看子线程`_loadingThread`调用的函数``loadImage``)：

```cpp
// add async struct into queue
_asyncStructQueue.push_back(data);
std::unique_lock<std::mutex> ul(_requestMutex);
_requestQueue.push_back(data);
_sleepCondition.notify_one();
```

处理`_responseQueue`的函数`addImageAsyncCallBack`：

```cpp
// 次函数是通过计时器每一帧都在调用的，用来处理_responseQueue中的数据
void TextureCache::addImageAsyncCallBack(float /*dt*/) {
    Texture2D *texture = nullptr;
    AsyncStruct *asyncStruct = nullptr;
    while (true) {
        // 首先加互斥_responseMutex，然后获取_responseQueue中的数据
        _responseMutex.lock();
        if (_responseQueue.empty()) {
            asyncStruct = nullptr;
        } else {
            asyncStruct = _responseQueue.front();
            _responseQueue.pop_front();

            // the asyncStruct's sequence order in _asyncStructQueue must equal to the order in _responseQueue
            CC_ASSERT(asyncStruct == _asyncStructQueue.front());
            _asyncStructQueue.pop_front();
        }
        _responseMutex.unlock();

        // 没有从_responseQueue中获取到数据，则跳出while循环
        if (nullptr == asyncStruct) {
            break;
        }

        // 有asyncStruct数据时，先从std::unordered_map<std::string, Texture2D*> _textures中找是否是之前已经加载缓存过的纹理，如果是的话直接给texture赋值
        auto it = _textures.find(asyncStruct->filename);
        if (it != _textures.end())
        {
            texture = it->second;
        }
        else
        {
            // asyncStruct中image数据加载成功时
            if (asyncStruct->loadSuccess)
            {
                // 初始化texture数据
                Image* image = &(asyncStruct->image);
                // generate texture in render thread
                texture = new (std::nothrow) Texture2D();

                texture->initWithImage(image, asyncStruct->pixelFormat);
                //parse 9-patch info
                this->parseNinePatchImage(image, texture, asyncStruct->filename);
#if CC_ENABLE_CACHE_TEXTURE_DATA
                // cache the texture file name
                VolatileTextureMgr::addImageTexture(texture, asyncStruct->filename);
#endif
                // cache the texture. retain it, since it is added in the map
                // 缓存加载的texture数据
                _textures.emplace(asyncStruct->filename, texture);
                texture->retain();

                texture->autorelease();
                // ETC1 ALPHA supports.
                if (asyncStruct->imageAlpha.getFileType() == Image::Format::ETC) {
                    auto alphaTexture = new(std::nothrow) Texture2D();
                    if(alphaTexture != nullptr && alphaTexture->initWithImage(&asyncStruct->imageAlpha, asyncStruct->pixelFormat)) {
                        texture->setAlphaTexture(alphaTexture);
                    }
                    CC_SAFE_RELEASE(alphaTexture);
                }
            } else {
                texture = nullptr;
                CCLOG("cocos2d: failed to call TextureCache::addImageAsync(%s)", asyncStruct->filename.c_str());
            }
        }

        // 调用异步加载传过来的回调函数
        if (asyncStruct->callback)
        {
            (asyncStruct->callback)(texture);
        }

        // release the asyncStruct
        delete asyncStruct;
        --_asyncRefCount;// _asyncRefCount自减1
    }

    // _asyncRefCount数量为0时代表已经没有需要异步加载的纹理了，此时注销掉计时器
    if (0 == _asyncRefCount) {
        Director::getInstance()->getScheduler()->unschedule(CC_SCHEDULE_SELECTOR(TextureCache::addImageAsyncCallBack), this);
    }
}
```

以上就是主要流程的分析了，下面自己也总结一下吧：

通过主线程的addImageAsync()函数创建子线程`_loadingThread`、新增计时器每一帧调用函数`addImageAsyncCallBack`，然后向`_requestQueue`中添加需要加载的纹理结构体，完成后通过`_sleepCondition`通知子线程`_loadingThread`从`_requestQueue`中获取需要加载的纹理结构体并且在子线程中加载image数据后添加到`_responseQueue`中，然后再主线程`addImageAsyncCallBack`中通过while循环一个个读取`_responseQueue`中的AsyncStruct结构数据并初始化`texture`数据并缓存到`std::unordered_map<std::string, Texture2D*>`结构`_textures`中，并回调异步加载函数。当加载数量`_asyncRefCount`为0时，注销掉计时器。

```cpp
std::deque<AsyncStruct*> _requestQueue;
std::deque<AsyncStruct*> _responseQueue;
```

其中这两个双端队列是主线程和子线程共同使用的数据，所以也有两个互斥mutex用来控制不同线程同时读取共享数据

```cpp
std::mutex _requestMutex;
std::mutex _responseMutex;
```

然后主线程通过`std::condition_variable _sleepCondition`和`std::mutex _requestMutex`来通知子线程的阻塞与非阻塞。

最后贴一下官方的解释吧~

```cpp
/**
 The addImageAsync logic follow the steps:
 - find the image has been add or not, if not add an AsyncStruct to _requestQueue  (GL thread)
 - get AsyncStruct from _requestQueue, load res and fill image data to AsyncStruct.image, then add AsyncStruct to _responseQueue (Load thread)
 - on schedule callback, get AsyncStruct from _responseQueue, convert image to texture, then delete AsyncStruct (GL thread)
 
 the Critical Area include these members:
 - _requestQueue: locked by _requestMutex
 - _responseQueue: locked by _responseMutex
 
 the object's life time:
 - AsyncStruct: construct and destruct in GL thread
 - image data: new in Load thread, delete in GL thread(by Image instance)
 
 Note:
 - all AsyncStruct referenced in _asyncStructQueue, for unbind function use.
 
 How to deal add image many times?
 - At first, this situation is abnormal, we only ensure the logic is correct.
 - If the image has been loaded, the after load image call will return immediately.
 - If the image request is in queue already, there will be more than one request in queue,
 - In addImageAsyncCallback, will deduplicate the request to ensure only create one texture.
 
 Does process all response in addImageAsyncCallback consume more time?
 - Convert image to texture faster than load image from disk, so this isn't a
 problem.

 The callbackKey allows to unbind the callback in cases where the loading of
 path is requested by several sources simultaneously. Each source can then
 unbind the callback independently as needed whilst a call to
 unbindImageAsync(path) would be ambiguous.
 */
```