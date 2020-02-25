对于大多数的服务器程序，其定时器一般支持单线程就够了，一般使用方法见下面代码。如果需要多线程怎么办，笔者一般用一个简单的办法：多线程的业务线程中不包含定时器管理器，单独启一个线程用来管理所有定时器，当时间触发时，向业务线程投递定时器消息即可。  


```c++
#include <iostream>
#include <thread>
#include <chrono>
#include "Timer.h"

void TimerHandler()
{
    std::cout << "TimerHandler" << std::endl;
}

int main()
{
    TimerManager tm;
    Timer t(tm);
    t.Start(&TimerHandler, 1000);
    while (true)
    {
        tm.DetectTimers();
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    std::cin.get();
    return 0;
}
```



# 一 最小堆实现  

```c++
#include <vector>
#include <boost/function.hpp>
 
class TimerManager;
 
class Timer
{
public:
    enum TimerType { ONCE, CIRCLE };
 
    Timer(TimerManager& manager);
    ~Timer();
 
    template<typename Fun>
    void Start(Fun fun, unsigned interval, TimerType timeType = CIRCLE);
    void Stop();
 
private:
    void OnTimer(unsigned long long now);
 
private:
    friend class TimerManager;
    TimerManager& manager_;
    TimerType timerType_;
    boost::function<void(void)> timerFun_;
    unsigned interval_;
    unsigned long long expires_;
 
    size_t heapIndex_;
};
 
class TimerManager
{
public:
    static unsigned long long GetCurrentMillisecs();
    void DetectTimers();
private:
    friend class Timer;
    void AddTimer(Timer* timer);
    void RemoveTimer(Timer* timer);
 
    void UpHeap(size_t index);
    void DownHeap(size_t index);
    void SwapHeap(size_t, size_t index2);
 
private:
    struct HeapEntry
    {
        unsigned long long time;
        Timer* timer;
    };
    std::vector<HeapEntry> heap_;
};
 
template<typename Fun>
inline void Timer::Start(Fun fun, unsigned interval, TimerType timeType)
{
    Stop();
    interval_ = interval;
    timerFun_ = fun;
    timerType_ = timeType;
    timer->expires_ = timer->interval_ + TimerManager::GetCurrentMillisecs();
    manager_.AddTimer(this);
}
 
// cpp file //////////////////////////////////////////////////////////////////
#define _CRT_SECURE_NO_WARNINGS
#include "config.h"
#include "timer.h"
#ifdef _MSC_VER
# include <sys/timeb.h>
#else
# include <sys/time.h>
#endif
 
//////////////////////////////////////////////////////////////////////////
// Timer
 
Timer::Timer(TimerManager& manager)
    : manager_(manager)
    , heapIndex_(-1)
{
}
 
Timer::~Timer()
{
    Stop();
}
 
void Timer::Stop()
{
    if (heapIndex_ != -1)
    {
        manager_.RemoveTimer(this);
        heapIndex_ = -1;
    }
}
 
void Timer::OnTimer(unsigned long long now)
{
    if (timerType_ == Timer::CIRCLE)
    {
        expires_ = interval_ + now;
        manager_.AddTimer(this);
    }
    else
    {
        heapIndex_ = -1;
    }
    timerFun_();
}
 
//////////////////////////////////////////////////////////////////////////
// TimerManager
 
void TimerManager::AddTimer(Timer* timer)
{
    timer->heapIndex_ = heap_.size();
    HeapEntry entry = { timer->expires_, timer };
    heap_.push_back(entry);
    UpHeap(heap_.size() - 1);
}
 
void TimerManager::RemoveTimer(Timer* timer)
{
    size_t index = timer->heapIndex_;
    if (!heap_.empty() && index < heap_.size())
    {
        if (index == heap_.size() - 1)
        {
            heap_.pop_back();
        }
        else
        {
            SwapHeap(index, heap_.size() - 1);
            heap_.pop_back();
            size_t parent = (index - 1) / 2;
            if (index > 0 && heap_[index].time < heap_[parent].time)
                UpHeap(index);
            else
                DownHeap(index);
        }
    }
}
 
void TimerManager::DetectTimers()
{
    unsigned long long now = GetCurrentMillisecs();
 
    while (!heap_.empty() && heap_[0].time <= now)
    {
        Timer* timer = heap_[0].timer;
        RemoveTimer(timer);
        timer->OnTimer(now);
    }
}
 
void TimerManager::UpHeap(size_t index)
{
    size_t parent = (index - 1) / 2;
    while (index > 0 && heap_[index].time < heap_[parent].time)
    {
        SwapHeap(index, parent);
        index = parent;
        parent = (index - 1) / 2;
    }
}
 
void TimerManager::DownHeap(size_t index)
{
    size_t child = index * 2 + 1;
    while (child < heap_.size())
    {
        size_t minChild = (child + 1 == heap_.size() || heap_[child].time < heap_[child + 1].time)
            ? child : child + 1;
        if (heap_[index].time < heap_[minChild].time)
            break;
        SwapHeap(index, minChild);
        index = minChild;
        child = index * 2 + 1;
    }
}
 
void TimerManager::SwapHeap(size_t index1, size_t index2)
{
    HeapEntry tmp = heap_[index1];
    heap_[index1] = heap_[index2];
    heap_[index2] = tmp;
    heap_[index1].timer->heapIndex_ = index1;
    heap_[index2].timer->heapIndex_ = index2;
}
 
 
unsigned long long TimerManager::GetCurrentMillisecs()
{
#ifdef _MSC_VER
    _timeb timebuffer;
    _ftime(&timebuffer);
    unsigned long long ret = timebuffer.time;
    ret = ret * 1000 + timebuffer.millitm;
    return ret;
#else
    timeval tv;         
    ::gettimeofday(&tv, 0);
    unsigned long long ret = tv.tv_sec;
    return ret * 1000 + tv.tv_usec / 1000;
#endif
}
```


# 二 时间轮实现  

```c++
// header file //////////////////////////////////
#pragma once
#include <list>
#include <vector>
#include <boost/function.hpp>
 
class TimerManager;
 
class Timer
{
public:
    enum TimerType {ONCE, CIRCLE};
 
    Timer(TimerManager& manager);
    ~Timer();
 
    template<typename Fun>
    void Start(Fun fun, unsigned interval, TimerType timeType = CIRCLE);
    void Stop();
 
private:
    void OnTimer(unsigned long long now);
 
private:
    friend class TimerManager;
 
    TimerManager& manager_;
    TimerType timerType_;
    boost::function<void(void)> timerFun_;
    unsigned interval_;
    unsigned long long expires_;
 
    int vecIndex_;
    std::list<Timer*>::iterator itr_;
};
 
class TimerManager
{
public:
    TimerManager();
 
    static unsigned long long GetCurrentMillisecs();
    void DetectTimers();
 
private:
    friend class Timer;
    void AddTimer(Timer* timer);
    void RemoveTimer(Timer* timer);
 
    int Cascade(int offset, int index);
 
private:
    typedef std::list<Timer*> TimeList;
    std::vector<TimeList> tvec_;
    unsigned long long checkTime_;
};
 
template<typename Fun>
inline void Timer::Start(Fun fun, unsigned interval, TimerType timeType)
{
    Stop();
    interval_ = interval;
    timerFun_ = fun;
    timerType_ = timeType;
    expires_ = interval_ + TimerManager::GetCurrentMillisecs();
    manager_.AddTimer(this);
}
 
 
// cpp file //////////////////////////////////////////////////
 
#define _CRT_SECURE_NO_WARNINGS
#include "config.h"
#include "timer2.h"
#ifdef _MSC_VER
# include <sys/timeb.h>
#else
# include <sys/time.h>
#endif
 
#define TVN_BITS 6
#define TVR_BITS 8
#define TVN_SIZE (1 << TVN_BITS)
#define TVR_SIZE (1 << TVR_BITS)
#define TVN_MASK (TVN_SIZE - 1)
#define TVR_MASK (TVR_SIZE - 1)
#define OFFSET(N) (TVR_SIZE + (N) *TVN_SIZE)
#define INDEX(V, N) ((V >> (TVR_BITS + (N) *TVN_BITS)) & TVN_MASK)
 
//////////////////////////////////////////////////////////////////////////
// Timer
 
Timer::Timer(TimerManager& manager)
    : manager_(manager)
    , vecIndex_(-1)
{
}
 
Timer::~Timer()
{
    Stop();
}
 
void Timer::Stop()
{
    if (vecIndex_ != -1)
    {
        manager_.RemoveTimer(this);
        vecIndex_ = -1;
    }
}
 
void Timer::OnTimer(unsigned long long now)
{
    if (timerType_ == Timer::CIRCLE)
    {
        expires_ = interval_ + now;
        manager_.AddTimer(this);
    }
    else
    {
        vecIndex_ = -1;
    }
    timerFun_();
}
 
//////////////////////////////////////////////////////////////////////////
// TimerManager
 
TimerManager::TimerManager()
{
    tvec_.resize(TVR_SIZE + 4 * TVN_SIZE);
    checkTime_ = GetCurrentMillisecs();
}
 
void TimerManager::AddTimer(Timer* timer)
{
    unsigned long long expires = timer->expires_;
    unsigned long long idx = expires - checkTime_;
   
    if (idx < TVR_SIZE)
    {
        timer->vecIndex_ = expires & TVR_MASK;
    }
    else if (idx < 1 << (TVR_BITS + TVN_BITS))
    {
        timer->vecIndex_ = OFFSET(0) + INDEX(expires, 0);
    }
    else if (idx < 1 << (TVR_BITS + 2 * TVN_BITS))
    {
        timer->vecIndex_ = OFFSET(1) + INDEX(expires, 1);
    }
    else if (idx < 1 << (TVR_BITS + 3 * TVN_BITS))
    {
        timer->vecIndex_ = OFFSET(2) + INDEX(expires, 2);
    }
    else if ((long long) idx < 0)
    {
        timer->vecIndex_ = checkTime_ & TVR_MASK;
    }
    else
    {   
        if (idx > 0xffffffffUL)
        {
            idx = 0xffffffffUL;
            expires = idx + checkTime_;
        }
        timer->vecIndex_ = OFFSET(3) + INDEX(expires, 3);
    }
 
    TimeList& tlist = tvec_[timer->vecIndex_];
    tlist.push_back(timer);
    timer->itr_ = tlist.end();
    --timer->itr_;
}
 
void TimerManager::RemoveTimer(Timer* timer)
{
    TimeList& tlist = tvec_[timer->vecIndex_];
    tlist.erase(timer->itr_);
}
 
void TimerManager::DetectTimers()
{
    unsigned long long now = GetCurrentMillisecs();
    while (checkTime_ <= now)
    {
        int index = checkTime_ & TVR_MASK;
        if (!index &&
            !Cascade(OFFSET(0), INDEX(checkTime_, 0)) &&
            !Cascade(OFFSET(1), INDEX(checkTime_, 1)) &&
            !Cascade(OFFSET(2), INDEX(checkTime_, 2)))
        {
            Cascade(OFFSET(3), INDEX(checkTime_, 3));
        }
        ++checkTime_;
 
        TimeList& tlist = tvec_[index];
        TimeList temp;
        temp.splice(temp.end(), tlist);
        for (TimeList::iterator itr = temp.begin(); itr != temp.end(); ++itr)
        {
            (*itr)->OnTimer(now);
        }
    }
}
 
int TimerManager::Cascade(int offset, int index)
{
    TimeList& tlist = tvec_[offset + index];
    TimeList temp;
    temp.splice(temp.end(), tlist);
 
    for (TimeList::iterator itr = temp.begin(); itr != temp.end(); ++itr)
    {
        AddTimer(*itr);
    }
 
    return index;
}
 
unsigned long long TimerManager::GetCurrentMillisecs()
{
#ifdef _MSC_VER
    _timeb timebuffer;
    _ftime(&timebuffer);
    unsigned long long ret = timebuffer.time;
    ret = ret * 1000 + timebuffer.millitm;
    return ret;
#else
    timeval tv;         
    ::gettimeofday(&tv, 0);
    unsigned long long ret = tv.tv_sec;
    return ret * 1000 + tv.tv_usec / 1000;
#endif
}
```


在曾经的很多项目中，定时器的实现都是使用map，也许效率不是太高，却从来没有成为性能的瓶颈。但是程序员通常是追求完美的，既然有更好解决方案，且其实现又不那么复杂，那就完全可以去尝试。