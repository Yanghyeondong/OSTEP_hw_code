---
date: '2023-08-30'
title: 'OS: Three Easy Pieces [Remzi] <27.Debugging Race> 풀이'
categories: ['Operating System']
summary: '교재 Operating Systems: Three Easy Pieces [Remzi] 속 Homework 문제 풀이'
thumbnail: './os/book2.png'
---
*해당 포스트는 필자가 직접 풀이한 것으로 오답이나 잘못된 해석이 있을 수도 있습니다.*  
*혹시나 지적할 부분이 있다면 항상 환영입니다!* 😏
## 문제 및 풀이

#### 1. First build main-race.c. Examine the code so you can see the (hopefully obvious) data race in the code. Now run helgrind (by typing valgrind --tool=helgrind main-race) to see how it reports the race. Does it point to the right lines of code? What other information does it give to you?

```c

#include <stdio.h>

#include "mythreads.h"

int done = 0;

void* worker(void* arg) {
    printf("this should print first\n");
    done = 1;
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p;
    Pthread_create(&p, NULL, worker, NULL);
    while (done == 0)
	;
    printf("this should print last\n");
    return 0;
}
```

```c
hdyang@DESKTOP-EU17S6T:~/os_hw/5$ valgrind --tool=helgrind ./main-race
==127== Helgrind, a thread error detector
==127== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.   
==127== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info==127== Command: ./main-race
==127==
==127== ---Thread-Announcement------------------------------------------
==127==
==127== Thread #1 is the program's root thread
==127==
==127== ---Thread-Announcement------------------------------------------  
==127==
==127== Thread #2 was created
==127==    at 0x4990BA3: clone (clone.S:76)
==127==    by 0x4991A9E: __clone_internal (clone-internal.c:83)
==127==    by 0x48FF758: create_thread (pthread_create.c:295)
==127==    by 0x490027F: pthread_create@@GLIBC_2.34 (pthread_create.c:828)==127==    by 0x4853767: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==127==    by 0x109564: Pthread_create (mythreads.h:51)
==127==    by 0x109654: main (main-race.c:17)
==127==
==127== ----------------------------------------------------------------  
==127==
==127== Possible data race during read of size 4 at 0x10C040 by thread #1 
==127== Locks held: none
==127==    at 0x109655: main (main-race.c:18)
==127==
==127== This conflicts with a previous write of size 4 by thread #2       
==127== Locks held: none
==127==    at 0x109609: worker (main-race.c:10)
==127==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==127==    by 0x48FFB42: start_thread (pthread_create.c:442)
==127==    by 0x4990BB3: clone (clone.S:100)
==127==  Address 0x10c040 is 0 bytes inside data symbol "balance"
==127==
==127== ----------------------------------------------------------------
==127==
==127== Possible data race during write of size 4 at 0x10C040 by thread #1==127== Locks held: none
==127==    at 0x10965E: main (main-race.c:18)
==127==
==127== This conflicts with a previous write of size 4 by thread #2
==127== Locks held: none
==127==    at 0x109609: worker (main-race.c:10)
==127==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==127==    by 0x48FFB42: start_thread (pthread_create.c:442)
==127==    by 0x4990BB3: clone (clone.S:100)
==127==  Address 0x10c040 is 0 bytes inside data symbol "balance"
==127==
==127==
==127== Use --history-level=approx or =none to gain increased speed, at
==127== the cost of reduced accuracy of conflicting-access information
==127== For lists of detected and suppressed errors, rerun with: -s
==127== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

helgrind를 보면 main의 15번째 줄(혹은 address)에 해당하는 "balance++;"에서 race가 나타난다는 것을 잘 알려주고 있다. 또한, Thread #1과 Thread #2가 서로 충돌을 일으킬수 있다는 사실과 사이즈는 4, 충돌이 가능한 변수가 "balance"라는 점도 알려준다.

#### 2. What happens when you remove one of the offending lines of code? Now add a lock around one of the updates to the shared variable, and then around both. What does helgrind report in each of these cases?
 
```c
==139== Helgrind, a thread error detector
==139== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.   
==139== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==139== Command: ./main-race
==139==
==139==
==139== Use --history-level=approx or =none to gain increased speed, at
==139== the cost of reduced accuracy of conflicting-access information 
==139== For lists of detected and suppressed errors, rerun with: -s    
==139== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

15번째 줄 "balance++;"을 제거하니 에러가 사라진 모습을 볼 수 있다.

```c
#include <stdio.h>

#include "mythreads.h"

int balance = 0;
pthread_mutex_t lock;

void *worker(void *arg)
{
    Pthread_mutex_lock(&lock);
    balance++; // unprotected access
    Pthread_mutex_unlock(&lock);
    return NULL;
}

int main(int argc, char *argv[])
{
    pthread_t p;
    Pthread_create(&p, NULL, worker, NULL);
    Pthread_mutex_lock(&lock);
    balance++; // unprotected access
    Pthread_mutex_unlock(&lock);
    Pthread_join(p, NULL);
    return 0;
}
```

```c
==151== Helgrind, a thread error detector
==151== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.   
==151== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==151== Command: ./main-race
==151==
==151==
==151== Use --history-level=approx or =none to gain increased speed, at
==151== the cost of reduced accuracy of conflicting-access information 
==151== For lists of detected and suppressed errors, rerun with: -s    
==151== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 7 from 7)
```
Pthread_mutex_lock(&lock)을 걸어주니 에러가 사라진 모습을 볼 수 있다.

#### 3. Now let’s look at main-deadlock.c. Examine the code. This code has a problem known as deadlock (which we discuss in much more depth in a forthcoming chapter). Can you see what problem it might have?

```c
#include <stdio.h>

#include "mythreads.h"

pthread_mutex_t m1 = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t m2 = PTHREAD_MUTEX_INITIALIZER;

void* worker(void* arg) {
    if ((long long) arg == 0) {
	Pthread_mutex_lock(&m1);
	Pthread_mutex_lock(&m2);
    } else {
	Pthread_mutex_lock(&m2);
	Pthread_mutex_lock(&m1);
    }
    Pthread_mutex_unlock(&m1);
    Pthread_mutex_unlock(&m2);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p1, p2;
    Pthread_create(&p1, NULL, worker, (void *) (long long) 0);
    Pthread_create(&p2, NULL, worker, (void *) (long long) 1);
    Pthread_join(p1, NULL);
    Pthread_join(p2, NULL);
    return 0;
}
```
두개의 thread p1, p2가 각각 m1과 m2에 대해 lock을 한 상태에서, 서로 다시 상대방의 m2와 m1을 교차해서 lock을 하게 될 가능성이 있다. 즉, p1은 m1을 lock 한 상태에서 m2가 lock 이 풀릴 때까지 기다리고 p2는 m2를 lock한 상태에서 m1의 lock이 풀릴 때까지 기다리므로 서로 dead lock 상태에 빠져서 영원히 실행되지 못하는 상황이 발생할 수 있다.

#### 4. Now run helgrind on this code. What does helgrind report?

```c

hdyang@DESKTOP-EU17S6T:~/os_hw/5$ valgrind --tool=helgrind ./main-deadlock
==159== Helgrind, a thread error detector
==159== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==159== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==159== Command: ./main-deadlock
==159==
==159== ---Thread-Announcement------------------------------------------
==159==
==159== Thread #3 was created
==159==    at 0x4990BA3: clone (clone.S:76)
==159==    by 0x4991A9E: __clone_internal (clone-internal.c:83)
==159==    by 0x48FF758: create_thread (pthread_create.c:295)
==159==    by 0x490027F: pthread_create@@GLIBC_2.34 (pthread_create.c:828)
==159==    by 0x4853767: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x109564: Pthread_create (mythreads.h:51)
==159==    by 0x1096C9: main (main-deadlock.c:24)
==159==
==159== ----------------------------------------------------------------
==159==
==159== Thread #3: lock order "0x10C040 before 0x10C080" violated
==159==
==159== Observed (incorrect) order is: acquisition of lock at 0x10C080
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109639: worker (main-deadlock.c:13)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==
==159==  followed by a later acquisition of lock at 0x10C040
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109648: worker (main-deadlock.c:14)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==
==159== Required order was established by acquisition of lock at 0x10C040
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109619: worker (main-deadlock.c:10)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==
==159==  followed by a later acquisition of lock at 0x10C080
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109628: worker (main-deadlock.c:11)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==
==159==  Lock at 0x10C040 was first observed
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109619: worker (main-deadlock.c:10)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==  Address 0x10c040 is 0 bytes inside data symbol "m1"
==159==
==159==  Lock at 0x10C080 was first observed
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109628: worker (main-deadlock.c:11)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==  Address 0x10c080 is 0 bytes inside data symbol "m2"
==159==
==159==
==159==
==159== Use --history-level=approx or =none to gain increased speed, at
==159== the cost of reduced accuracy of conflicting-access information
==159== For lists of detected and suppressed errors, rerun with: -s
==159== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 7 from 7)
```

lock order가 violated 됨을 알려준다. m1과 m2의 address를 알려주며 서로 요구하는 lock이 고착 상태에 빠진다는 사실을 알려준다.


#### 5. Now run helgrind on main-deadlock-global.c. Examine the code; does it have the same problem that main-deadlock.c has? Should helgrind be reporting the same error? What does this tell you about tools like helgrind?
 
 

```c
==159== Helgrind, a thread error detector
==159== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==159== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==159== Command: ./main-deadlock
==159==
==159== ---Thread-Announcement------------------------------------------
==159==
==159== Thread #3 was created
==159==    at 0x4990BA3: clone (clone.S:76)
==159==    by 0x4991A9E: __clone_internal (clone-internal.c:83)
==159==    by 0x48FF758: create_thread (pthread_create.c:295)
==159==    by 0x490027F: pthread_create@@GLIBC_2.34 (pthread_create.c:828)
==159==    by 0x4853767: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x109564: Pthread_create (mythreads.h:51)
==159==    by 0x1096C9: main (main-deadlock.c:24)
==159==
==159== ----------------------------------------------------------------
==159==
==159== Thread #3: lock order "0x10C040 before 0x10C080" violated
==159==
==159== Observed (incorrect) order is: acquisition of lock at 0x10C080
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109639: worker (main-deadlock.c:13)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==
==159==  followed by a later acquisition of lock at 0x10C040
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109648: worker (main-deadlock.c:14)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==
==159== Required order was established by acquisition of lock at 0x10C040
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109619: worker (main-deadlock.c:10)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==
==159==  followed by a later acquisition of lock at 0x10C080
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109628: worker (main-deadlock.c:11)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==
==159==  Lock at 0x10C040 was first observed
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109619: worker (main-deadlock.c:10)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==  Address 0x10c040 is 0 bytes inside data symbol "m1"
==159==
==159==  Lock at 0x10C080 was first observed
==159==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==159==    by 0x109628: worker (main-deadlock.c:11)
==159==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==159==    by 0x48FFB42: start_thread (pthread_create.c:442)
==159==    by 0x4990BB3: clone (clone.S:100)
==159==  Address 0x10c080 is 0 bytes inside data symbol "m2"
==159==
==159==
==159==
==159== Use --history-level=approx or =none to gain increased speed, at
==159== the cost of reduced accuracy of conflicting-access information
==159== For lists of detected and suppressed errors, rerun with: -s
==159== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 7 from 7)
hdyang@DESKTOP-EU17S6T:~/os_hw/5$ valgrind --tool=helgrind ./main-deadlock-global 
==163== Helgrind, a thread error detector
==163== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==163== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==163== Command: ./main-deadlock-global
==163==
==163== ---Thread-Announcement------------------------------------------
==163==
==163== Thread #3 was created
==163==    at 0x4990BA3: clone (clone.S:76)
==163==    by 0x4991A9E: __clone_internal (clone-internal.c:83)
==163==    by 0x48FF758: create_thread (pthread_create.c:295)
==163==    by 0x490027F: pthread_create@@GLIBC_2.34 (pthread_create.c:828)
==163==    by 0x4853767: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x109564: Pthread_create (mythreads.h:51)
==163==    by 0x1096E7: main (main-deadlock-global.c:27)
==163==
==163== ----------------------------------------------------------------
==163==
==163== Thread #3: lock order "0x10C080 before 0x10C0C0" violated
==163==
==163== Observed (incorrect) order is: acquisition of lock at 0x10C0C0
==163==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==163==    by 0x109648: worker (main-deadlock-global.c:15)
==163==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x48FFB42: start_thread (pthread_create.c:442)
==163==    by 0x4990BB3: clone (clone.S:100)
==163==
==163==  followed by a later acquisition of lock at 0x10C080
==163==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==163==    by 0x109657: worker (main-deadlock-global.c:16)
==163==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x48FFB42: start_thread (pthread_create.c:442)
==163==    by 0x4990BB3: clone (clone.S:100)
==163==
==163== Required order was established by acquisition of lock at 0x10C080
==163==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==163==    by 0x109628: worker (main-deadlock-global.c:12)
==163==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x48FFB42: start_thread (pthread_create.c:442)
==163==    by 0x4990BB3: clone (clone.S:100)
==163==
==163==  followed by a later acquisition of lock at 0x10C0C0
==163==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==163==    by 0x109637: worker (main-deadlock-global.c:13)
==163==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x48FFB42: start_thread (pthread_create.c:442)
==163==    by 0x4990BB3: clone (clone.S:100)
==163==
==163==  Lock at 0x10C080 was first observed
==163==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==163==    by 0x109628: worker (main-deadlock-global.c:12)
==163==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x48FFB42: start_thread (pthread_create.c:442)
==163==    by 0x4990BB3: clone (clone.S:100)
==163==  Address 0x10c080 is 0 bytes inside data symbol "m1"
==163==
==163==  Lock at 0x10C0C0 was first observed
==163==    at 0x4850CCF: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x1093A6: Pthread_mutex_lock (mythreads.h:23)
==163==    by 0x109637: worker (main-deadlock-global.c:13)
==163==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==163==    by 0x48FFB42: start_thread (pthread_create.c:442)
==163==    by 0x4990BB3: clone (clone.S:100)
==163==  Address 0x10c0c0 is 0 bytes inside data symbol "m2"
==163==
==163==
==163==
==163== Use --history-level=approx or =none to gain increased speed, at
==163== the cost of reduced accuracy of conflicting-access information
==163== For lists of detected and suppressed errors, rerun with: -s
==163== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 7 from 7)
```

main-deadlock-global.c의 경우, 글로벌 lock을 통해 이전의 문제를 해결한다. 다만, helgrind 상에서는 조금 차이가 있긴 하지만 여전히 에러문구가 발생한다. 이러한 점은 helgrind도 해당 코드의 에러를 잘못 판단할 수 있다는 점을 알려준다.


#### 6. Let’s next look at main-signal.c. This code uses a variable (done) to signal that the child is done and that the parent can now continue. Why is this code inefficient? (what does the parent end up spending its time doing, particularly if the child thread takes a long time to complete?)
해당 코드의 경우 child thread가 오랫동안 실행된다면, done이 1이 되기 전까지 parent는 아무 의미 없는 반복문만 계속 돌며(spin) 자원을 낭비하게 된다.

#### 7. Now run helgrind on this program. What does it report? Is the code correct?
 
 

```c
hdyang@DESKTOP-EU17S6T:~/os_hw/5$ valgrind --tool=helgrind ./main-signal
==167== Helgrind, a thread error detector
==167== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==167== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==167== Command: ./main-signal
==167==
this should print first
==167== ---Thread-Announcement------------------------------------------
==167==
==167== Thread #2 was created
==167==    at 0x4990BA3: clone (clone.S:76)
==167==    by 0x4991A9E: __clone_internal (clone-internal.c:83)
==167==    by 0x48FF758: create_thread (pthread_create.c:295)
==167==    by 0x490027F: pthread_create@@GLIBC_2.34 (pthread_create.c:828)
==167==    by 0x4853767: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==167==    by 0x109584: Pthread_create (mythreads.h:51)
==167==    by 0x109682: main (main-signal.c:15)
==167==
==167== ---Thread-Announcement------------------------------------------
==167==
==167== Thread #1 is the program's root thread
==167==
==167== ----------------------------------------------------------------
==167==
==167== Possible data race during write of size 4 at 0x10C014 by thread #2
==167== Locks held: none
==167==    at 0x109633: worker (main-signal.c:9)
==167==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==167==    by 0x48FFB42: start_thread (pthread_create.c:442)
==167==    by 0x4990BB3: clone (clone.S:100)
==167==
==167== This conflicts with a previous read of size 4 by thread #1
==167== Locks held: none
==167==    at 0x109684: main (main-signal.c:16)
==167==  Address 0x10c014 is 0 bytes inside data symbol "done"
==167==
==167== ----------------------------------------------------------------
==167==
==167== Possible data race during read of size 4 at 0x10C014 by thread #1
==167== Locks held: none
==167==    at 0x109684: main (main-signal.c:16)
==167==
==167== This conflicts with a previous write of size 4 by thread #2
==167== Locks held: none
==167==    at 0x109633: worker (main-signal.c:9)
==167==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==167==    by 0x48FFB42: start_thread (pthread_create.c:442)
==167==    by 0x4990BB3: clone (clone.S:100)
==167==  Address 0x10c014 is 0 bytes inside data symbol "done"
==167==
==167== ----------------------------------------------------------------
==167==
==167== Possible data race during write of size 1 at 0x52971A5 by thread #1
==167== Locks held: none
==167==    at 0x4859796: mempcpy (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==167==    by 0x48F66E4: _IO_new_file_xsputn (fileops.c:1235)
==167==    by 0x48F66E4: _IO_file_xsputn@@GLIBC_2.2.5 (fileops.c:1196)
==167==    by 0x48EBF9B: puts (ioputs.c:40)
==167==    by 0x10969C: main (main-signal.c:18)
==167==  Address 0x52971a5 is 21 bytes inside a block of size 1,024 alloc'd
==167==    at 0x484A919: malloc (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==167==    by 0x48E9C23: _IO_file_doallocate (filedoalloc.c:101)
==167==    by 0x48F8D5F: _IO_doallocbuf (genops.c:347)
==167==    by 0x48F7FDF: _IO_file_overflow@@GLIBC_2.2.5 (fileops.c:744)
==167==    by 0x48F6754: _IO_new_file_xsputn (fileops.c:1243)
==167==    by 0x48F6754: _IO_file_xsputn@@GLIBC_2.2.5 (fileops.c:1196)
==167==    by 0x48EBF9B: puts (ioputs.c:40)
==167==    by 0x109632: worker (main-signal.c:8)
==167==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==167==    by 0x48FFB42: start_thread (pthread_create.c:442)
==167==    by 0x4990BB3: clone (clone.S:100)
==167==  Block was alloc'd by thread #2
==167==
this should print last
==167==
==167== Use --history-level=approx or =none to gain increased speed, at
==167== the cost of reduced accuracy of conflicting-access information
==167== For lists of detected and suppressed errors, rerun with: -s
==167== ERROR SUMMARY: 24 errors from 3 contexts (suppressed: 40 from 34)
```

helgrind에서는 done이라는 변수에 대해 16번째 줄 "while (done == 0)"과 9번째 줄 "done = 1;" 에서 race 상태가 발생할 수 있다는 오류를 보여준다. 또한, printf()에 대해서도 순서 문제가 발생함을 알려준다. 다만, 실제로는 spin 등에 의해 이러한 부분이 발생하지 않게 된다. 즉, 실제 코드에서는 문제가 발생하지 않게 된다.  

#### 8. Now look at a slightly modified version of the code, which is found in main-signal-cv.c. This version uses a condition variable to do the signaling (and associated lock). Why is this code preferred to the previous version? Is it correctness, or performance, or both? 


```c
typedef struct __synchronizer_t {
    pthread_mutex_t lock;
    pthread_cond_t cond;
    int done;
} synchronizer_t;

synchronizer_t s;

void signal_init(synchronizer_t *s) {
    Pthread_mutex_init(&s->lock, NULL);
    Pthread_cond_init(&s->cond, NULL);
    s->done = 0;
}

void signal_done(synchronizer_t *s) {
    Pthread_mutex_lock(&s->lock);
    s->done = 1;
    Pthread_cond_signal(&s->cond);
    Pthread_mutex_unlock(&s->lock);
}

void signal_wait(synchronizer_t *s) {
    Pthread_mutex_lock(&s->lock);
    while (s->done == 0)
	Pthread_cond_wait(&s->cond, &s->lock);
    Pthread_mutex_unlock(&s->lock);
}

void* worker(void* arg) {
    printf("this should print first\n");
    signal_done(&s);
    return NULL;
}

int main(int argc, char *argv[]) {
    pthread_t p;
    signal_init(&s);
    Pthread_create(&p, NULL, worker, NULL);
    signal_wait(&s);
    printf("this should print last\n");

    return 0;
} 
```

우선, correctness 측면에서 해당 코드는 이전보다 좋아졌다. signaling 관련 기능을 추가함으로써 thread의 lock과 관련된 정보들이 서로 전달되고, 이를 통해 concurrency 관련 문제들을 미리 예방한다.

그리고, performance 측면에서도 해당 코드는 이전보다 좋아졌다. 의미 없는 spin을 반복해서 자원을 낭비하지 않고 pthread_cond_wait() 등을 통해서 스케줄링을 바꿈으로써 (ex. blocked) 더욱 효율이 좋기 때문이다.


#### 9. Once again run helgrind on main-signal-cv. Does it report any errors?


```c
hdyang@DESKTOP-EU17S6T:~/os_hw/5$ valgrind --tool=helgrind ./main-signal-cv 
==180== Helgrind, a thread error detector
==180== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==180== Using Valgrind-3.18.1 and LibVEX; rerun with -h for copyright info
==180== Command: ./main-signal-cv
==180==
this should print first
this should print last
==180==
==180== Use --history-level=approx or =none to gain increased speed, at
==180== the cost of reduced accuracy of conflicting-access information
==180== For lists of detected and suppressed errors, rerun with: -s
==180== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 12 from 12)
```

에러가 발생하지 않는다.


## Source

- 『Operating Systems: Three Easy Pieces』 *-Remzi H. Arpaci-Dusseau 지음*  
  [https://pages.cs.wisc.edu/~remzi/OSTEP/](https://pages.cs.wisc.edu/~remzi/OSTEP/)