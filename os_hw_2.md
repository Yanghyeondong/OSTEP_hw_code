---
date: '2023-08-21'
title: 'OS: Three Easy Pieces [Remzi] <14.Debugging Memory> 풀이'
categories: ['Operating System']
summary: '교재 Operating Systems: Three Easy Pieces [Remzi] 속 Homework 문제 풀이'
thumbnail: './os/book2.png'
---
*해당 포스트는 필자가 직접 풀이한 것으로 오답이나 잘못된 해석이 있을 수도 있습니다.*  
*혹시나 지적할 부분이 있다면 항상 환영입니다!* 😏
## 문제 및 풀이
#### 1. First, write a simple program called null.c that creates a pointer to an integer, sets it to NULL, and then tries to dereference it. Compile this into an executable called null.

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int* x;
    x = NULL;
    int y = *x;
    return 0;
}
```

```c
2019312219@swji:~/hw/2$ gcc -o null null.c
```

What happens when you run this program?  
프로그램 실행 시 아래와 같이 Segmentation fault가 발생한다.  

```c
2019312219@swji:~/hw/2$ null
Segmentation fault (core dumped)
```

#### 2. Next, compile this program with symbol information included (with the -g flag). Doing so let’s put more information into the executable, enabling the debugger to access more useful information about variable names and the like. Run the program under the debugger by typing gdb null and then, once gdb is running, typing run. 

```c
2019312219@swji:~/hw/2$ gcc -g -o null null.c
2019312219@swji:~/hw/2$ gdb null
GNU gdb (Ubuntu 8.1.1-0ubuntu1) 8.1.1
Copyright (C) 2018 Free Software Foundation, Inc.
(생략..)
Reading symbols from null...done.
(gdb) run
Starting program: /home/2019312219/hw/2/null 

Program received signal SIGSEGV, Segmentation fault.
0x000055555555460a in main () at null.c:7
7           int y = *x;
```

What does gdb show you?  
dereference한 x를 사용하는 코드에서 SIGSEGV signal과 함께 Segmentation fault 가 발생함을 알려주고 있다  


#### 3. Finally, use the valgrind tool on this program. We’ll use the memcheck tool that is a part of valgrind to analyze what happens. Run this by typing in the following: valgrind --leak-check=yes null. 

```c
2019312219@swji:~/hw/2$ valgrind --leak-check=yes null
==10900== Memcheck, a memory error detector
==10900== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==10900== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==10900== Command: null
==10900== 
==10900== Invalid read of size 4
==10900==    at 0x10860A: main (null.c:7)
==10900==  Address 0x0 is not stack'd, malloc'd or (recently) free'd
==10900== 
==10900== 
==10900== Process terminating with default action of signal 11 (SIGSEGV)
==10900==  Access not within mapped region at address 0x0
==10900==    at 0x10860A: main (null.c:7)
==10900==  If you believe this happened as a result of a stack
==10900==  overflow in your program's main thread (unlikely but
==10900==  possible), you can try to increase the size of the
==10900==  main thread stack using the --main-stacksize= flag.
==10900==  The main thread stack size used in this run was 8388608.
==10900== 
==10900== HEAP SUMMARY:
==10900==     in use at exit: 0 bytes in 0 blocks
==10900==   total heap usage: 0 allocs, 0 frees, 0 bytes allocated
==10900== 
==10900== All heap blocks were freed -- no leaks are possible
==10900== 
==10900== For counts of detected and suppressed errors, rerun with: -v
==10900== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
Segmentation fault (core dumped)
```

What happens when you run this? Can you interpret the output from the tool?  
`int y = *x;` 를 통해 포인터x에 해당하는 값을 불러오려 하니 `Invalid read of size 4, Address 0x0 is not stack'd, malloc'd or (recently) free'd` 오류가 발생한다. 이는 x가 null이기 때문에 dereference를 할 때 해당 주소의 값을 찾을 수 없기 때문이다.   
추가로 `Process terminating with default action of signal 11 (SIGSEGV), Access not within mapped region at address 0x0` 등 SIGSEGV signal 과 address 참조 오류도 출력된다.  

#### 4. Write a simple program  that allocates memory using malloc() but forgets to free it before exiting. 

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int* x = (int*) malloc(sizeof(int) * 1);
    *x = 1;
    printf("x: %d\n",*x);
    return 0;
}
```
  
What happens when this program runs?  
아래 출력과 같이 아무 이상 없이 실행된다.  

```c
2019312219@swji:~/hw/2$ gcc -g -o q4 q4.c
2019312219@swji:~/hw/2$ q4
x: 1
```

Can you use gdb to find any problems with it?  
아래의 출력과 같이 아무 문제가 없음을 확인할 수 있다.  

```c
2019312219@swji:~/hw/2$ gdb q4
GNU gdb (Ubuntu 8.1.1-0ubuntu1) 8.1.1
Copyright (C) 2018 Free Software Foundation, Inc.
(생략)...
Reading symbols from q4...done.
(gdb) run
Starting program: /home/2019312219/hw/2/q4 
x: 1
[Inferior 1 (process 21774) exited normally]
(gdb)
```

How about valgrind (again with the --leak-check=yes flag)?  
valgrind로 확인해보면 아래 출력과 같이 오류가 출력된다. 4 bytes in 1 blocks are definitely lost in loss record 1 of 1 부분을 통해 malloc으로 할당한 메모리를 `free()` 해주지 않아서 발생하는 오류임을 알 수 있다.  

```c
2019312219@swji:~/hw/2$ valgrind --leak-check=yes q4
==22300== Memcheck, a memory error detector
==22300== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==22300== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==22300== Command: q4
==22300== 
x: 1
==22300== 
==22300== HEAP SUMMARY:
==22300==     in use at exit: 4 bytes in 1 blocks
==22300==   total heap usage: 2 allocs, 1 frees, 1,028 bytes allocated
==22300== 
==22300== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==22300==    at 0x4C31B0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==22300==    by 0x10869B: main (q4.c:5)
==22300== 
==22300== LEAK SUMMARY:
==22300==    definitely lost: 4 bytes in 1 blocks
==22300==    indirectly lost: 0 bytes in 0 blocks
==22300==      possibly lost: 0 bytes in 0 blocks
==22300==    still reachable: 0 bytes in 0 blocks
==22300==         suppressed: 0 bytes in 0 blocks
==22300== 
==22300== For counts of detected and suppressed errors, rerun with: -v
==22300== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

#### 5. Write a program that creates an array of integers called data of size 100 using malloc; then, set data[100] to zero. 

```c
#include <stdio.h>
#include <stdlib.h>

int main() {

    int* data = (int*) malloc(100);
    data[100] = 0;
    free(data);
    return 0;
    
}
```

What happens when you run this program?  
아래 출력과 같이 아무 이상 없이 실행된다.  

```c
2019312219@swji:~/hw/2$ gcc -g -o q5 q5.c
2019312219@swji:~/hw/2$ q5
2019312219@swji:~/hw/2$
```

What happens when you run this program using valgrind? Is the program correct?  
valgrind로 확인해보면 아래 같이 오류가 출력된다. Invalid write of size 4 출력을 통해 `data[100] = 0;` 부분에서 올바르지 않은(범위를 벗어나는) write 를 했음을 알려준다. 그 이유는 int형의 크기가 4이므로 data[100]이 가르키는 위치는 기존의 malloc(100) 범위를 벗어나기 때문이다.  

```c
2019312219@swji:~/hw/2$ valgrind --leak-check=yes q5
==26470== Memcheck, a memory error detector
==26470== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==26470== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==26470== Command: q5
==26470== 
==26470== Invalid write of size 4
==26470==    at 0x1086AA: main (q5.c:7)
==26470==  Address 0x522f1d0 is 224 bytes inside an unallocated block of size 4,194,032 in arena "client"
==26470== 
==26470== 
==26470== HEAP SUMMARY:
==26470==     in use at exit: 0 bytes in 0 blocks
==26470==   total heap usage: 1 allocs, 1 frees, 100 bytes allocated
==26470== 
==26470== All heap blocks were freed -- no leaks are possible
==26470== 
==26470== For counts of detected and suppressed errors, rerun with: -v
==26470== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

#### 6. Create a program that allocates an array of integers (as above), frees them, and then tries to print the value of one of the elements of the array.

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int* data = (int*) malloc(100);
    free(data);
    printf("data[0]: %d\n", data[0]);
    return 0;
}
```

Does the program run?   
아래 출력과 같이 아무 이상 없이 실행된다.  

```c
2019312219@swji:~/hw/2$ gcc -g -o q6 q6.c
2019312219@swji:~/hw/2$ q6
data[0]: 0
2019312219@swji:~/hw/2$
```

What happens when you use valgrind on it?  
valgrind로 확인해보면 아래 같이 오류가 출력된다. Invalid read of size 4 출력을 통해 `printf("data[0]: %d\n", data[0]);` 부분에서 올바르지 않은read 를 했음을 알려준다. 추가적으로 `Address 0x522f040 is 0 bytes inside a block of size 100 free'd` 를 통해 해당 주소는 이미 free()가 완료 되었음을 알려주며, Block was alloc'd at 을 통해 과거 malloc으로 할당 되었던 부분임도 알려준다.  

```c
2019312219@swji:~/hw/2$ valgrind --leak-check=yes q6
==21293== Memcheck, a memory error detector
==21293== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==21293== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==21293== Command: q6
==21293== 
==21293== Invalid read of size 4
==21293==    at 0x108700: main (q6.c:7)
==21293==  Address 0x522f040 is 0 bytes inside a block of size 100 free'd
==21293==    at 0x4C32D3B: free (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==21293==    by 0x1086FB: main (q6.c:6)
==21293==  Block was alloc'd at
==21293==    at 0x4C31B0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==21293==    by 0x1086EB: main (q6.c:5)
==21293== 
data[0]: 0
==21293== 
==21293== HEAP SUMMARY:
==21293==     in use at exit: 0 bytes in 0 blocks
==21293==   total heap usage: 2 allocs, 2 frees, 1,124 bytes allocated
==21293== 
==21293== All heap blocks were freed -- no leaks are possible
==21293== 
==21293== For counts of detected and suppressed errors, rerun with: -v
==21293== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

#### 7. Now pass a funny value to free (e.g., a pointer in the middle of the array you allocated above). 

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    int* data = (int*) malloc(100);
    free(&data[7]);
    return 0;
}
```

What happens? Do you need tools to find this type of problem?  
우선 해당 코드를 컴파일 후 실행할 시 아래 출력과 같이 `free(): invalid pointer, Aborted (core dumped)` 오류가 출력된다. 해당 부분을 통해 free()함수에 잘못된 포인터가 전달되어서 오류가 발생함을 알 수 있다.  

```c
2019312219@swji:~/hw/2$ gcc -g -o q7 q7.c
2019312219@swji:~/hw/2$ q7
free(): invalid pointer
Aborted (core dumped)
```

Valgrind를 실행시켜 보면 아래 같이 오류가 출력된다. Invalid free() / delete / delete[] / realloc() 라고 오류 이유를 알려주며, `Address 0x522f05c is 28 bytes inside a block of size 100 alloc'd` 와 같이 free()하려고 한 주소 &data[7]가 총 100 사이즈의 블록 중 28 (4*7)에 해당하는 주소임을 알려준다. 또한 100 bytes in 1 blocks are definitely lost in loss record 1 of 1 로 해제되지 못한 메모리도 알려준다.  

```c
2019312219@swji:~/hw/2$ valgrind --leak-check=yes q7
==6234== Memcheck, a memory error detector
==6234== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==6234== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==6234== Command: q7
==6234== 
==6234== Invalid free() / delete / delete[] / realloc()
==6234==    at 0x4C32D3B: free (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==6234==    by 0x1086AF: main (q7.c:6)
==6234==  Address 0x522f05c is 28 bytes inside a block of size 100 alloc'd
==6234==    at 0x4C31B0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==6234==    by 0x10869B: main (q7.c:5)
==6234== 
==6234== 
==6234== HEAP SUMMARY:
==6234==     in use at exit: 100 bytes in 1 blocks
==6234==   total heap usage: 1 allocs, 1 frees, 100 bytes allocated
==6234== 
==6234== 100 bytes in 1 blocks are definitely lost in loss record 1 of 1
==6234==    at 0x4C31B0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==6234==    by 0x10869B: main (q7.c:5)
==6234== 
==6234== LEAK SUMMARY:
==6234==    definitely lost: 100 bytes in 1 blocks
==6234==    indirectly lost: 0 bytes in 0 blocks
==6234==      possibly lost: 0 bytes in 0 blocks
==6234==    still reachable: 0 bytes in 0 blocks
==6234==         suppressed: 0 bytes in 0 blocks
==6234== 
==6234== For counts of detected and suppressed errors, rerun with: -v
==6234== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

#### 8. Try out some of the other interfaces to memory allocation. For example, create a simple vector-like data structure and related routines that use realloc() to manage the vector. Use an array to store the vectors elements; when a user adds an entry to the vector, use realloc() to allocate more space for it. 

```c
#include <stdio.h>
#include <stdlib.h>

struct my_vector
{
    int max_size;
    int now_size;
    int* data;
};

void my_vector_show(struct my_vector* v) {
    printf("now vector: ");
    for(int i=0;i<v->now_size;i++){
        printf("%d ",v->data[i]);
    }
    printf("\n");
}

int my_vector_at(struct my_vector* v, int index) {
    if (index < 0 || (v->now_size-1) < index){
        printf("wrong index\n");
        return -1;
    }
    int result = v->data[index];
    printf("vector at [%d] is: %d\n",index, result);
    return result;
}

void my_vector_push_back(struct my_vector* v, int value) {
    printf("push_back %d \n", value);

    if (v->now_size == v->max_size) {
        printf("vector is full. make space by realloc ... \n");
        v->max_size = 2 * v->now_size;
        v->data = (int*) realloc(v->data, v->max_size * sizeof(int));
    }
    v->data[v->now_size] = value;
    v->now_size = v->now_size + 1;
}

int my_vector_pop_back(struct my_vector* v) {
    int result = v->data[v->now_size-1];
    v->data[v->now_size-1] = 0;
    v->now_size = v->now_size - 1;
    printf("pop_back %d \n", result);
    return result;
}

void my_vector_free(struct my_vector* v) {
    printf("free vector!\n");
    v->max_size = 0;
    v->now_size = 0;
    free(v->data);
}

int main() {

    struct my_vector v = {2, 0, (int*)malloc(sizeof(int)*2)};
    my_vector_push_back(&v, 1);
    my_vector_push_back(&v, 2);
    my_vector_show(&v);
    my_vector_push_back(&v, 3);
    my_vector_push_back(&v, 4);
    my_vector_at(&v,0);
    my_vector_pop_back(&v);
    my_vector_pop_back(&v);
    my_vector_show(&v);
    my_vector_free(&v);
    my_vector_show(&v);
    return 0;

}
```

How well does such a vector perform?   
위의 코드에선 my_vector struct를 정의하고 해당하는 함수를 구현하였다. `my_vector_show()`는 현재 벡터의 원소를 보여주며, `my_vector_at()`은 특정 인덱스의 값을 리턴 한다.`my_vector_push_back()`과 `my_vector_pop_back()`은 각각 벡터의 맨 뒤에서 값을 넣고 삭제하는 역할을 한다. 마지막으로 my_vector_free()는 벡터의 메모리를 해제하는 역할을 한다.  
`Main()` 함수에서는 최초의 벡터를 `struct my_vector v = {2, 0, (int*)malloc(sizeof(int)*2)};` 과 같이 설정하고, 이어서 push, pop 등을 진행한다. 이때 기존의 size 2를 넘어설 경우 vector is full. make space by realloc ... 문구와 함께 `realloc()` 재할당이 되는 것을 확인 할 수 있다.   

```c
2019312219@swji:~/hw/2$ gcc -g -o q8 q8.c
2019312219@swji:~/hw/2$ q8
push_back 1 
push_back 2 
now vector: 1 2 
push_back 3 
vector is full. make space by realloc ... 
push_back 4 
vector at [0] is: 1
pop_back 4 
pop_back 3 
now vector: 1 2 
free vector!
now vector:
```

How does it compare to a linked list?   
linked list에 비해 직접 만든vector가 가지는 장점은 기본적으로 배열을 활용하기 때문에 빠른 속도로 인덱스에 대한 값 접근이 가능하다는 점이다. 다만, 단점의 경우 linked list는 중간에 삽입을 할 때 새로운 node에 link를 할당해 주기만 하면 되지만, vector는 모든 값들을 하나씩 밀어야 하는 과정이 필요하여 상대적으로 속도가 느리다. 마지막으로 vector는 기존의 크기 재설정이 불가능했던 배열과 다르게linked list처럼 필요할 때 마다 그 크기를 늘릴 수 있다는 장점이 있다.  

Use valgrind to help you find bugs.  
처음 코드를 작성했을 때는, 아래의 첫번째 valgrind 출력처럼 오류가 발생했다. 그 이유를 확인해보니 my_vector_pop_back() 함수에서 v->data[v->now_size] = 0; 코드에서 버그가 발생하였다. [now_size] 인덱스로 접근하면 실제 사용하는 크기보다 더 큰, 즉 realloc에서 할당하지 않은 부분까지 써버린다. 그래서 valgrind 오류가 발생했던 것이다. 이를 v->data[v->now_size-1] = 0; 로 고치고 나니 두번째 valgrind 출력처럼 오류문구가 출력되지 않았다.  
\
고치기 전:  
```c
2019312219@swji:~/hw/2$ valgrind --leak-check=yes q8
==22122== Memcheck, a memory error detector
==22122== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==22122== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==22122== Command: q8
==22122== 
push_back 1 
push_back 2 
now vector: 1 2 
push_back 3 
vector is full. make space by realloc ... 
push_back 4 
vector at [0] is: 1
==22122== Invalid write of size 4
==22122==    at 0x1089EC: my_vector_pop_back (q8.c:43)
==22122==    by 0x108AFD: main (q8.c:65)
==22122==  Address 0x522f4e0 is 0 bytes after a block of size 16 alloc'd
==22122==    at 0x4C33D2F: realloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==22122==    by 0x10896A: my_vector_push_back (q8.c:35)
==22122==    by 0x108ACF: main (q8.c:62)
==22122== 
pop_back 4 
pop_back 3 
now vector: 1 2 
free vector!
now vector: 
==22122== 
==22122== HEAP SUMMARY:
==22122==     in use at exit: 0 bytes in 0 blocks
==22122==   total heap usage: 3 allocs, 3 frees, 1,048 bytes allocated
==22122== 
==22122== All heap blocks were freed -- no leaks are possible
==22122== 
==22122== For counts of detected and suppressed errors, rerun with: -v
==22122== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```
고친 후:  

```c
2019312219@swji:~/hw/2$ valgrind --leak-check=yes q8
==22318== Memcheck, a memory error detector
==22318== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==22318== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==22318== Command: q8
==22318== 
push_back 1 
push_back 2 
now vector: 1 2 
push_back 3 
vector is full. make space by realloc ... 
push_back 4 
vector at [0] is: 1
pop_back 4 
pop_back 3 
now vector: 1 2 
free vector!
now vector: 
==22318== 
==22318== HEAP SUMMARY:
==22318==     in use at exit: 0 bytes in 0 blocks
==22318==   total heap usage: 3 allocs, 3 frees, 1,048 bytes allocated
==22318== 
==22318== All heap blocks were freed -- no leaks are possible
==22318== 
==22318== For counts of detected and suppressed errors, rerun with: -v
==22318== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```
#### 9. Spendmore time and read about using gdb and valgrind. Knowing your tools is critical; spend the time and learn how to become an expert debugger in the UNIX and C environment.
앞선 문제를 통해 알 수 있듯이, 메모리 문제가 있는 코드일지라도 일반적인 실행을 통해서는 그 문제를 파악하기가 쉽지 않았다. 그러나 실행 시 나타나지 않는다고 이를 방치하면 나중에 메모리 누수와 같은 문제가 생기게 된다. 따라서 gdb, valgrind 같은 툴을 통해 자신의 프로그램에 대한 디버깅을 충분히 해나 갈수 있도록 노력이 필요함을 깨닫는 계기가 되었다.  


## Source

- 『Operating Systems: Three Easy Pieces』 *-Remzi H. Arpaci-Dusseau 지음*  
  [https://pages.cs.wisc.edu/~remzi/OSTEP/](https://pages.cs.wisc.edu/~remzi/OSTEP/)