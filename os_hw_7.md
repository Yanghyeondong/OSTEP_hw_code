---
date: '2023-09-03'
title: 'OS: Three Easy Pieces [Remzi] <32.Deadlock> 풀이'
categories: ['Operating System']
summary: '교재 Operating Systems: Three Easy Pieces [Remzi] 속 Homework 문제 풀이'
thumbnail: './os/book2.png'
---
*해당 포스트는 필자가 직접 풀이한 것으로 오답이나 잘못된 해석이 있을 수도 있습니다.*  
*혹시나 지적할 부분이 있다면 항상 환영입니다!* 😏
## 문제 및 풀이

#### 1. First let’s make sure you understand how the programs generally work, and some of the key options. Study the code in vector-deadlock.c, as well as in main-common.c and related files. Now, run ./vector-deadlock -n 2 -l 1 -v, which instantiates two threads (-n 2), each of which does one vector add (-l 1), and does so in verbose mode (-v). Make sure you understand the output. How does the output change from run to run?


```c
2019312219@swji:~/hw/7$ ./vector-deadlock -n 2 -l 1 -v
->add(0, 1)
<-add(0, 1)
              ->add(0, 1)
              <-add(0, 1)
2019312219@swji:~/hw/7$ ./vector-deadlock -n 2 -l 1 -v
->add(0, 1)
              ->add(0, 1)
              <-add(0, 1)
<-add(0, 1)
2019312219@swji:~/hw/7$ ./vector-deadlock -n 2 -l 1 -v
->add(0, 1)
<-add(0, 1)
              ->add(0, 1)
              <-add(0, 1)
```

실행시마다 좌측과 우측의 출력 순서가 가끔씩 바뀌는 모습을 보인다. 즉, 쓰레드의 순서가 조금씩 바뀌는 것을 확인할 수 있다.


#### 2. Now add the -d flag, and change the number of loops (-l) from 1 to higher numbers. What happens? Does the code (always) deadlock?

```c
->add(1, 0)
              <-add(1, 0)
              ->add(1, 0)
              <-add(1, 0)
              ->add(1, 0)
              <-add(1, 0)
              ->add(1, 0)
              <-add(1, 0)
              ->add(1, 0)
              <-add(1, 0)
              ->add(1, 0)
              <-add(1, 0)
              ->add(1, 0)
              <-add(1, 0)
              ->add(1, 0)
              <-add(1, 0)
              ->add(1, 0)
->add(0, 1)
^C
```
항상은 아니고 가끔씩 deadlock 에 빠지는 경우가 있다.


#### 3. How does changing the number of threads (-n) change the outcome of the program? Are there any values of -n that ensure no deadlock occurs?
`-n`의 값이 늘어나면 데드락이 더 자주 걸리는 것 같은 느낌이다. 당연하게도, 쓰레드의 개수 `-n` 이 2개 이상이 아니라면 deadlock이 발생하지 않는다. 즉, `-n` < 2 인 경우 deadlock이 불가능하다.

#### 4. Now examine the code in vector-global-order.c. First, make sure you understand what the code is trying to do; do you understand why the code avoids deadlock? Also, why is there a special case in this vector add() routine when the source and destination vectors are the same?
`v_dst`와 `v_src` 값의 차이에 따라서 `mutex_lock` 순서를 바꿔주기 때문에 avoids deadlock 이 가능하다. 그리고 special case가 있는 이유는 `v_dst`와 `v_src` 가 서로 동일할때 lock을 각자 걸게되면 결국 동일한 lock을 2번 거는게 되어버리기 때문이다. 따라서 special case로 이러한 상황을 처리해주게 된다.

#### 5. Now run the code with the following flags: -t -n 2 -l 100000 -d. How long does the code take to complete? How does the total time change when you increase the number of loops, or the number of threads?


```c
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 100000 -d
Time: 0.08 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 1000000 -d
Time: 0.59 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 10000000 -d
Time: 4.87 seconds
```

`-l` 의 값을 늘릴경우 실행시간이 점점 길어지는 모습을 보인다. 정확히 비례하지는 않지만, `-l` 이 증가함에따라 같이 증가하는 모습을 보인다.


```c
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 100000 -d
Time: 0.08 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 4 -l 100000 -d
Time: 0.25 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 8 -l 100000 -d
Time: 0.64 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 16 -l 100000 -d
Time: 1.96 seconds
```

`-n` 의 값을 늘릴경우 실행시간이 점점 길어지는 모습을 보인다.


#### 6. What happens if you turn on the parallelism flag (-p)? How much would you expect performance to change when each thread is working on adding different vectors (which is what -p enables) versus working on the same ones?


```c
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 100000 -d -p
Time: 0.08 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 4 -l 100000 -d -p
Time: 0.05 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 8 -l 100000 -d -p
Time: 0.09 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 16 -l 100000 -d -p
Time: 0.11 seconds
```

이전과 비교했을때 -n 의 값이 증가하더라도 실행시간에 엄청 큰 영향을 미치지는 않았다. 서로 병렬로 실행됨에 따라 각 쓰레드가 대기하는 시간이 없어져서 그런 것으로 예상된다.

#### 7. Now let’s study vector-try-wait.c. First make sure you understand the code. Is the first call to pthread mutex trylock() really needed? Now run the code. How fast does it run compared to the global order approach? How does the number of retries, as counted by the code, change as the number of threads increases?
해당 코드가 꼭 필요하지는 않다.


```c
2019312219@swji:~/hw/7$ ./vector-try-wait -t -n 2 -l 1000000
Retries: 69399
Time: 1.05 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 1000000
Time: 0.60 seconds
```

`global order approach` 에 비해 `vector-try-wait`의 시간이 조금더 걸리는 것을 볼 수 있다.

```c
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 1000000
Time: 0.69 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 4 -l 1000000
Time: 2.19 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 8 -l 1000000
Time: 5.67 seconds
```

`-n`의 개수가 늘수록 `Retries`와 `Time`도 확연히 많아지는 모습을 보인다.


#### 8. Now let’s look at vector-avoid-hold-and-wait.c. What is the main problem with this approach? How does its performance compare to the other versions, when running both with -p and without it?



```c
2019312219@swji:~/hw/7$ ./vector-avoid-hold-and-wait -t -n 2 -l 1000000
Time: 1.13 seconds
2019312219@swji:~/hw/7$ ./vector-try-wait -t -n 2 -l 1000000
Retries: 95539
Time: 0.90 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 1000000
Time: 0.49 seconds
2019312219@swji:~/hw/7$ ./vector-avoid-hold-and-wait -t -n 2 -l 1000000 -p
Time: 0.65 seconds
2019312219@swji:~/hw/7$ ./vector-try-wait -t -n 2 -l 1000000 -p
Retries: 0
Time: 0.25 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 1000000 -p
Time: 0.27 seconds
```

코드속 주석 `put GLOBAL lock around all lock acquisition` 의 설명처럼 하나를 제외하고 모든 쓰레드에 락이 걸리게되어 속도가 느려진다.

#### 9. Finally, let’s look at vector-nolock.c. This version doesn’t use locks at all; does it provide the exact same semantics as the other versions? Why or why not?
해당 코드는 다른코드와 다르게 lock이 아닌 `fetch_and_add()` 함수를 사용한다. 코드내의 주석 https://en.wikipedia.org/wiki/Fetch-and-add 사이트 설명을 보면 알 수 있듯이 CPU instruction atomically 한 성질을 지니기 때문에 lock 처럼 동작할 수 있다.
 
#### 10. Now compare its performance to the other versions, both when threads are working on the same two vectors (no -p) and when each thread is working on separate vectors (-p). How does this no-lock version perform?


```c
2019312219@swji:~/hw/7$ ./vector-avoid-hold-and-wait -t -n 2 -l 1000000
Time: 0.89 seconds
2019312219@swji:~/hw/7$ ./vector-try-wait -t -n 2 -l 1000000
Retries: 85995
Time: 0.93 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 1000000
Time: 0.50 seconds
2019312219@swji:~/hw/7$ ./vector-nolock -t -n 2 -l 1000000
Time: 2.54 seconds
2019312219@swji:~/hw/7$ ./vector-avoid-hold-and-wait -t -n 2 -l 1000000 -p
Time: 0.63 seconds
2019312219@swji:~/hw/7$ ./vector-try-wait -t -n 2 -l 1000000 -p
Retries: 0
Time: 0.19 seconds
2019312219@swji:~/hw/7$ ./vector-global-order -t -n 2 -l 1000000 -p
Time: 0.19 seconds
2019312219@swji:~/hw/7$ ./vector-nolock -t -n 2 -l 1000000 -p
Time: 0.96 seconds
```

`-p`가 없을때는 다른 버전에 비해 비교적 낮은 성능을 보여주며, -p 옵션이 들어가면 -p가 없을때의 자기 자신보단 빨라지지만 여전히 다른 버전에 비해 낮은 성능을 보여준다.

## Source

- 『Operating Systems: Three Easy Pieces』 *-Remzi H. Arpaci-Dusseau 지음*  
  [https://pages.cs.wisc.edu/~remzi/OSTEP/](https://pages.cs.wisc.edu/~remzi/OSTEP/)