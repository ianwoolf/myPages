+++
date = "2015-11-04T16:30:09+08:00"
draft = true
title = "golang的锁"

+++
golang的锁分互斥锁和读写锁

### 互斥锁
比较简单,这里先略过,以后有时间补上.


### 读写锁

读写锁是通过对unit32加减以及阻塞来完成对读锁和写锁的操作。

需要注意的一点是 加/放开 写锁会 => 锁住/解锁读写锁；
而读锁无限制，只是注意不要unlock超过lock的数量。

这两个函数会阻塞
   
`func runtime_Semacquire(s *uint32)` => 用来等待解锁，解锁后加锁; 

`func runtime_Semrelease(s *uint32)` => 用来解锁，解锁后出发处于block的加锁动作

读写锁的部分源码,以后有时间补全注释

    type RWMutex struct {
        w           Mutex  // held if there are pending writers
    // 计数,同时用来控制wait => 加减锁
        writerSem   uint32 // semaphore for writers to wait for completing readers
        readerSem   uint32 // semaphore for readers to wait for completing writers
        readerCount int32  // number of pending readers
        readerWait  int32  // number of departing readers
    }

    const rwmutexMaxReaders = 1 << 30

    // RLock locks rw for reading.
    func (rw *RWMutex) RLock() {
        if raceenabled {
                _ = rw.w.state
                raceDisable()
        }
    // 原子操作
        if atomic.AddInt32(&rw.readerCount, 1) < 0 {
    // 这个地方用计数来控制wait
                runtime_Semacquire(&rw.readerSem)
        }
        if raceenabled {
                raceEnable()
                raceAcquire(unsafe.Pointer(&rw.readerSem))
        }
    }
    //  解锁类似
  
    // 写锁的加锁
    func (rw *RWMutex) Lock() {
        if raceenabled {
                _ = rw.w.state
                raceDisable()
        }
        rw.w.Lock()
    //原子操作，线程安全
        r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders  
        if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
    // 可能会阻塞
                runtime_Semacquire(&rw.writerSem)  
        }
        if raceenabled {
                raceEnable()
    // 读锁  写锁  都被锁住
                raceAcquire(unsafe.Pointer(&rw.readerSem)) // unsafe包获得指针
                raceAcquire(unsafe.Pointer(&rw.writerSem))
        }
    }
    // 解锁类似
