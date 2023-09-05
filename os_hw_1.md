---
date: '2023-08-20'
title: 'OS: Three Easy Pieces [Remzi] <5.Process API> 풀이'
categories: ['Operating System']
summary: '교재 Operating Systems: Three Easy Pieces [Remzi] 속 Homework 문제 풀이'
thumbnail: './os/book2.png'
---
*해당 포스트는 필자가 직접 풀이한 것으로 오답이나 잘못된 해석이 있을 수도 있습니다.*  
*혹시나 지적할 부분이 있다면 항상 환영입니다!* 😏
## 문제 및 풀이
#### 1. Write a program that calls fork(). Before calling fork(), have the main process access a variable (e.g., x) and set its value to something (e.g., 100).
```c
int main(int argc, char *argv[])
{
    int x = 100;
    printf("At the start, x is %d\n", x);
    
    int rc = fork();
    if (rc < 0) {
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child
        printf("I am child (pid:%d)\n", (int) getpid());
        printf("\tnow x is %d\n", x);
        printf("\tchange x into %d\n", 200);
        x = 200;
        printf("\tnow x is %d\n", x);
    } else {
        // parent
        printf("I am parent of %d (pid:%d)\n",
	       rc, (int) getpid());
        printf("\tnow x is %d\n", x);
        printf("\tchange x into %d\n", 300);
        x = 300;
        printf("\tnow x is %d\n", x);
    }
    return 0;
}
```
stdout:
```c
At the start, x is 100
I am parent of 26124 (pid:26123)
        now x is 100
        change x into 300
        now x is 300
I am child (pid:26124)
        now x is 100
        change x into 200
```

What value is the variable in the child process?  
⇒ child process에서 변수 x의 값은 처음에 설정한 100으로 나타난다. (복제된다)  
What happens to the variable when both the child and parent change the value of x?  
⇒ child와 parent는 fork() 전 x 값의 서로 다른 복제본을 사용한다. 따라서 child와 parent가 x를 바꾸더라도 서로 영향이 가지 않게 개별로 변형시킨다. child (x = 100 → 200), parent (x = 100 → 300)  

#### 2. Write a program that opens a file (with the open() system call) and then calls fork() to create a new process.  
```c
int main(int argc, char *argv[])
{
    int fd = open("./test.txt", O_CREAT|O_WRONLY|O_TRUNC, S_IRWXU);
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child
        char sent[] = "child\n";
        for(int i=0; i<10;i++){
            write(fd, sent, strlen(sent));
        }

    } else {
        // parent
        char sent[] = "parent\n";
        for(int i=0; i<10;i++){
            write(fd, sent, strlen(sent));
        }
    }
    return 0;
}
```
test.txt: (실행마다 바뀜)
```c
parent
parent
parent
parent
parent
parent
child
parent
child
parent
child
parent
child
parent
child
child
child
child
child
child
```


Can both the child and parent access the file descriptor returned by open()?  
⇒ child와 parent 모두 file descriptor 접근이 가능하다. write()를 통해 test.txt에 접근 가능함을 확인하였다.  
What happens when they are writing to the file concurrently, i.e., at the same time?  
⇒ child와 parent가 동시에 접근할 경우, 출력이 번갈아 되면서 겹치는 부분이 가끔씩 발생하였다. 즉, IO 접근 할당에 따라 child와 parent가 각자의 출력을 번갈아 실행하는 것이 확인된다.  

#### 3. Write another program using fork(). The child process should print “hello”; the parent process should print “goodbye”. You should try to ensure that the child process always prints first;
```c
int main(int argc, char *argv[])
{
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child
        printf("hello\n");
    } else {
        // parent

        int wc = wait(NULL);
        // sleep(10); 
        // wait()대신 충분한 sleep()을 사용해도 child가 먼저 print 되도록 가능함.

        printf("goodbye\n");
    }
    return 0;
}
```
stdout:

```c
hello
goodbye

```
can you do this without calling wait() in the parent?  
⇒ parent 실행 부분에 wait()를 제거하고 sleep() 함수를 추가할 경우, parent가 먼저 실행되더라도 일정시간 뒤에 스케줄러에 의해 child가 실행되어 hello가 먼저 print되도록 만든다. 단, sleep()을 충분히 주지 않을 경우 parent가 먼저 print 될 수도 있다.  

#### 4. Write a program that calls fork() and then calls some form of exec() to run the program /bin/ls. See if you can try all of the variants of exec(), including (on Linux) execl(), execle(), execlp(), execv(), execvp(), and execvpe().

```c
int main(int argc, char *argv[])
{
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        execl("/bin/ls", "ls", NULL);

        // extern char **environ;
        // execle("/bin/ls", "ls", NULL, environ);

        // execlp("ls", "ls", NULL);

        // char *myargs[2];
        // myargs[0] = strdup("/bin/ls");
        // myargs[1] = NULL;
        // execv(myargs[0], myargs);

        // char *myargs[2];
        // myargs[0] = strdup("ls");
        // myargs[1] = NULL;
        // execvp(myargs[0], myargs);

        /* 
        execvpe은 #define _GNU_SOURCE가 필요함.
        */
        // extern char **environ;
        // char *myargs[2];
        // myargs[0] = strdup("/bin/ls");
        // myargs[1] = NULL;
        // execvpe(myargs[0], myargs, environ);
        
    } else {
        // parent
        int wc = wait(NULL);
	assert(wc >= 0);
    }
    return 0;
}
```

stdout:
```c
1.c  3.c  5.c  7.c  ans_1.out  ans_3.out  ans_5.out  ans_7.out  r
2.c  4.c  6.c  8.c  ans_2.out  ans_4.out  ans_6.out  ans_8.out  test.txt
```

Why do you think there are so many variants of the same basic call?  
⇒ exec() 함수가 여러 개로 파생되는 이유는 디렉토리 이름, 명령라인 인수, 환경변수 등의 여부가 다르기 때문이다. exec 뒤에 l이 붙는 경우는 명령라인 인수를 리스트로 받는 것이고, v가 붙는 것은 명령라인 인수를 배열로 받는 것이다. e가 붙는 것은 환경변수 envp를 받는 것이고, p가 붙는 것은 디렉토리와 파일 이름 전체가 아닌 파일 이름만 받는 것이다. 이렇게 다양한 기능과 옵션에 따라 exec()가 파생되는 것이다.  

#### 5. Now write a program that uses wait() to wait for the child process to finish in the parent.   
```c
int main(int argc, char *argv[])
{
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child
        int wc = wait(NULL);
        printf("I am child (wc:%d) (pid:%d)\n",
	        wc, (int) getpid());
    } else {
        // parent
        int wc = wait(NULL);
        printf("I am parent of %d (wc:%d) (pid:%d)\n",
	       rc, wc, (int) getpid());
    }
    return 0;
}
```

stdout:
```c
I am child (wc:-1) (pid:31828)
I am parent of 31828 (wc:31828) (pid:31827)
```

What does wait() return?   
⇒ parent의 경우 wait()는 child의 pid를 return 한다.  
What happens if you use wait() in the child?  
⇒ child의 경우 wait()는 -1을 return 한다.    

#### 6. Write a slight modification of the previous program, this time using waitpid() instead of wait().   
```c
int main(int argc, char *argv[])
{
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
        // child (new process)
        printf("I am child (pid:%d)\n", (int) getpid());
    } else {
        // parent goes down this path (original process)
        int wc = waitpid(rc, NULL, 0);
        printf("I am parent of %d (wc:%d) (pid:%d)\n",
	       rc, wc, (int) getpid());
    }
    return 0;
}
```
stdout:
```c
I am child (pid:32730)
I am parent of 32730 (wc:32730) (pid:32729)
```
When would waitpid() be useful?  
부모 프로세스가 특정 자식 프로세스를 기다리거나 자식의 종료를 기다리지 않는 등 특수한 조건을 넣고 싶을 때, waitpid()가 유용하다. waidpid 함수는 다음과 같은 인자들을 받는다. pid_t waitpid (pid_t pid, int* status, int options) 여기서 pid는 자식 프로세스의 id를 의미하며, -1, 0 등 값을 바꿈으로써 자식 프로세스 전체를 기다릴 수도, 특정 자식만 기다릴 수도 있다. status의 경우 wait() 함수와 동일한 역할을 하며, options는 WNOHANG과 같이 자식 프로세스의 종료를 기다리지 않는 기능을 한다.  

#### 7. Write a program that creates a child process, and then in the child closes standard output (STDOUT FILENO). 

```c
int main(int argc, char *argv[])
{
    int rc = fork();
    if (rc < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc == 0) {
	// child: redirect standard output to a file
	    close(STDOUT_FILENO); 
        printf("I am child (pid:%d)\n", (int) getpid());
    } else {
        int wc = wait(NULL);
	assert(wc >= 0);
    }
    return 0;
}
```
stdout:

```c
   
```
What happens if the child calls printf() to print some output after closing the descriptor?  
⇒ child가 descriptor(STDOUT FILENO)를 닫은 후 printf()를 호출할 경우, 아무것도 출력되지 않음을 알 수 있다.  

#### 8. Write a program that creates two children, and connects the standard output of one to the standard input of the other, using the pipe() system call.

```c
int main(int argc, char *argv[])
{
    int fd[2];
    pipe(fd);

    int rc_1 = fork();
    if (rc_1 < 0) {
        // fork failed; exit
        fprintf(stderr, "fork failed\n");
        exit(1);
    } else if (rc_1 == 0) {
	    // child_1
        printf("I am child_1 (pid:%d)\n", (int) getpid());
        char msg_send[100] = "i'm child_1. who are you?";

        close(fd[0]);
        printf("\twrite msg to STDOUT_FILENO: %s\n", msg_send);
        dup2(fd[1], STDOUT_FILENO);
        write(STDOUT_FILENO, msg_send, 100);

    } else {
        int rc_2 = fork();
        if (rc_2 < 0) {
            // fork failed; exit
            fprintf(stderr, "fork failed\n");
            exit(1);
        } else if (rc_2 == 0) {
            // child_2
            printf("I am child_2 (pid:%d)\n", (int) getpid());
            char msg_get[100];

            close(fd[1]);
            dup2(fd[0], STDIN_FILENO);
            read(STDIN_FILENO, msg_get, 100);
            printf("\tread msg from STDIN_FILENO: %s\n", msg_get);

        } else {
            printf("I am parents (pid:%d)\n", (int) getpid());
        }
        int wc_1 = wait(NULL);
        int wc_2 = wait(NULL);
    }
    return 0;
}
```
stdout:

```c
I am parents (pid:2213)
I am child_1 (pid:2214)
        write msg to STDOUT_FILENO: i'm child_1. who are you?
I am child_2 (pid:2215)
        read msg from STDIN_FILENO: i'm child_1. who are you?
```

⇒ 위의 코드는 pipe() 함수로 file descriptor를 만들고, fork() 함수로 child_1과 child_2 자식 프로세스를 만든다. 이때 child_1 은 dup2() 함수를 통해 기존의 STDOUT을 fd[1](output stream)으로 교체하고, child_2은 기존의 STDIN을 fd[0] 로 교체한다. 즉, 한 자식의 standard output을 다른 자식의 standard input으로 만든다. 이경우, child_1이 write를 통해 문자열을 STDOUT로 보내면 child_2가 read를 통해 STDIN에서 받게 된다. 결과적으로 자식 프로세스 간의 통신이 가능해진 것이다.

## Source

- 『Operating Systems: Three Easy Pieces』 *-Remzi H. Arpaci-Dusseau 지음*  
  [https://pages.cs.wisc.edu/~remzi/OSTEP/](https://pages.cs.wisc.edu/~remzi/OSTEP/)