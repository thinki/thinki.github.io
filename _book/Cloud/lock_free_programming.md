# Lock Free Programming

## 1 What is lock free programming

### 1.1 What is lock ?

primitive that exclude other threads from a critical section until the current thread unlocks

### 1.2 What is lock free ?

#### 1.2.1 definition

From [wiki](https://en.wikipedia.org/wiki/Non-blocking_algorithm#:~:text=Lock-freedom,-Lock-freedom allows&text=All wait-free algorithms are,algorithm is not lock-free.):

In particular, **if one thread is suspended, then a lock-free algorithm guarantees that the remaining threads can still make progress**. Hence, if two threads can contend for the same mutex lock or spinlock, then the algorithm is not lock-free. (If we suspend one thread that holds the lock, then the second thread will block.)

#### 1.2.2 lock free vs w/o locking

lock free is not strictly equal to without locking, some program without using lock may not be lock free. But the program using lock can't be locked free(since it introduce critical section, refer below for more details)

One typical way to implement lock free program is that : Thread-safe access to shared data without the use of synchronization primitives such as mutexes

## 2 Why lock free

### 2.1 lock free benefits:

Lock has many problems :

- Deadlock
- Priority Inversion
- Convoying
- “Async-signal-safety”
- Kill-tolerant availability
- Pre-emption tolerance
- Overall performance

(refer to https://www.cnblogs.com/tuowang/p/9399052.html for more details)

Most of the above problems are caused by potential scheduling while thread is in the [critical section](https://en.wikipedia.org/wiki/Critical_section). 

When great efforts is needed to solve these problems, lock free is another alternative

### 2.2 lock free problems:

ABA problem

## 3 How to do lock free

Designing generalized lock-free algorithms is hard.

Design lock-free data structures instead: Buffer, list, stack, queue, map, deque, snapshot

Often implemented in terms of simpler primitives: e.g. Compare and Swap

## 4 lock free example:

- [Linux RCU](https://lwn.net/Articles/262464/)
- Linux kfifo
- [DPDK RTE_RING](https://dpdk.readthedocs.io/en/v16.04/prog_guide/ring_lib.html)



## 5 lock-based vs lock free

lock free:

- may be higher performed compared to the condition when lock-based sleep is heavy and critical section is very small.
  lock free vs spinlock: https://lumian2015.github.io/lockFreeProgramming/lock-free-vs-spin-lock.html 
- Is not a cure for contention:
  1. It’s still possible to have too many threads competing for a lock free data structure
  2. Starvation is still a possibility
- Requires the same hardware support as mutexes do
- Is not a magic bullet
- may have more difficult

## 6 Reference

https://www.cs.cmu.edu/~410-s05/lectures/L31_LockFree.pdf

http://www.rossbencina.com/code/lockfree?q=~rossb/code/lockfree/

https://stackoverflow.com/questions/41146527/is-a-spinlock-lock-free

https://www.zhihu.com/question/53303879

## 7 Appendix

RTE RING dpdk test benchmark:

```shell
➜  dpdk-master git:(dpdk_21.02) ✗ sudo ./release/app/test/dpdk-test -l 1,2-15 --
[sudo] jim 的密码：
EAL: Detected 56 lcore(s)
EAL: Detected 2 NUMA nodes
EAL: Detected static linkage of DPDK
EAL: Multi-process socket /var/run/dpdk/rte/mp_socket
EAL: Selected IOVA mode 'VA'
EAL: No available 1048576 kB hugepages reported
EAL: Probing VFIO support...
EAL: VFIO support initialized
EAL: No legacy callbacks, legacy socket not created
APP: HPET is not enabled, using TSC as default timer
RTE>>ring_
 ring_autotest [Mul-choice STRING]: launch autotest
 ring_perf_autotest [Mul-choice STRING]: launch autotest
 ring_stress_autotest [Mul-choice STRING]: launch autotest
 ring_pmd_perf_autotest [Mul-choice STRING]: launch autotest
 ring_pmd_autotest [Mul-choice STRING]: launch autotest
RTE>>ring_p
 ring_perf_autotest [Mul-choice STRING]: launch autotest
 ring_pmd_perf_autotest [Mul-choice STRING]: launch autotest
 ring_pmd_autotest [Mul-choice STRING]: launch autotest
RTE>>ring_perf_autotest
 
### Testing single element enq/deq ###
legacy APIs: SP/SC: single: 9.24
legacy APIs: MP/MC: single: 37.83
 
### Testing burst enq/deq ###
legacy APIs: SP/SC: burst (size: 8): 25.53
legacy APIs: SP/SC: burst (size: 32): 63.49
legacy APIs: MP/MC: burst (size: 8): 52.43
legacy APIs: MP/MC: burst (size: 32): 89.03
 
### Testing bulk enq/deq ###
legacy APIs: SP/SC: bulk (size: 8): 24.68
legacy APIs: SP/SC: bulk (size: 32): 62.18
legacy APIs: MP/MC: bulk (size: 8): 53.07
legacy APIs: MP/MC: bulk (size: 32): 89.35
 
### Testing empty bulk deq ###
legacy APIs: SP/SC: bulk (size: 8): 3.31
legacy APIs: MP/MC: bulk (size: 8): 3.32
 
### Testing using two physical cores ###
legacy APIs: SP/SC: bulk (size: 8): 38.09
legacy APIs: MP/MC: bulk (size: 8): 102.69
legacy APIs: SP/SC: bulk (size: 32): 22.72
legacy APIs: MP/MC: bulk (size: 32): 30.57
 
### Testing using all worker nodes ###
 
Bulk enq/dequeue count on size 8
Core [1] count = 13900
Core [2] count = 13886
Core [3] count = 13862
Core [4] count = 13847
Core [5] count = 13858
Core [6] count = 13838
Core [7] count = 13847
Core [8] count = 13839
Core [9] count = 13879
Core [10] count = 13878
Core [11] count = 13847
Core [12] count = 13839
Core [13] count = 13842
Core [14] count = 13856
Core [15] count = 13864
Total count (size: 8): 207882
 
Bulk enq/dequeue count on size 32
Core [1] count = 12876
Core [2] count = 12884
Core [3] count = 12902
Core [4] count = 12902
Core [5] count = 12895
Core [6] count = 12881
Core [7] count = 12883
Core [8] count = 12888
Core [9] count = 12890
Core [10] count = 12928
Core [11] count = 12912
Core [12] count = 12894
Core [13] count = 12887
Core [14] count = 12871
Core [15] count = 12900
Total count (size: 32): 193393
 
### Testing single element enq/deq ###
elem APIs: element size 16B: SP/SC: single: 8.07
elem APIs: element size 16B: MP/MC: single: 38.39
 
### Testing burst enq/deq ###
elem APIs: element size 16B: SP/SC: burst (size: 8): 27.90
elem APIs: element size 16B: SP/SC: burst (size: 32): 67.64
elem APIs: element size 16B: MP/MC: burst (size: 8): 56.52
elem APIs: element size 16B: MP/MC: burst (size: 32): 97.26
 
### Testing bulk enq/deq ###
elem APIs: element size 16B: SP/SC: bulk (size: 8): 29.23
elem APIs: element size 16B: SP/SC: bulk (size: 32): 67.49
elem APIs: element size 16B: MP/MC: bulk (size: 8): 56.46
elem APIs: element size 16B: MP/MC: bulk (size: 32): 96.35
 
### Testing empty bulk deq ###
elem APIs: element size 16B: SP/SC: bulk (size: 8): 2.65
elem APIs: element size 16B: MP/MC: bulk (size: 8): 1.99
 
### Testing using two physical cores ###
elem APIs: element size 16B: SP/SC: bulk (size: 8): 36.40
elem APIs: element size 16B: MP/MC: bulk (size: 8): 109.03
elem APIs: element size 16B: SP/SC: bulk (size: 32): 20.36
elem APIs: element size 16B: MP/MC: bulk (size: 32): 27.51
 
### Testing using all worker nodes ###
 
Bulk enq/dequeue count on size 8
Core [1] count = 13715
Core [2] count = 13684
Core [3] count = 13657
Core [4] count = 13679
Core [5] count = 13667
Core [6] count = 13650
Core [7] count = 13680
Core [8] count = 13678
Core [9] count = 13682
Core [10] count = 13652
Core [11] count = 13673
Core [12] count = 13656
Core [13] count = 13657
Core [14] count = 13693
Core [15] count = 13683
Total count (size: 8): 205106
 
Bulk enq/dequeue count on size 32
Core [1] count = 13408
Core [2] count = 13390
Core [3] count = 13395
Core [4] count = 13411
Core [5] count = 13399
Core [6] count = 13396
Core [7] count = 13390
Core [8] count = 13409
Core [9] count = 13395
Core [10] count = 13384
Core [11] count = 13407
Core [12] count = 13400
Core [13] count = 13398
Core [14] count = 13361
Core [15] count = 13402
Total count (size: 32): 200945
Test OK
```
