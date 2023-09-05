---
date: '2023-09-05'
title: 'OS: Three Easy Pieces [Remzi] <39.FS APIs> 풀이'
categories: ['Operating System']
summary: '교재 Operating Systems: Three Easy Pieces [Remzi] 속 Homework 문제 풀이'
thumbnail: './os/book2.png'
---
*해당 포스트는 필자가 직접 풀이한 것으로 오답이나 잘못된 해석이 있을 수도 있습니다.*  
*혹시나 지적할 부분이 있다면 항상 환영입니다!* 😏
## 문제 및 풀이

#### 1. Stat: Write your own version of the command line program stat, which simply calls the stat() system call on a given file or directory. Print out file size, number of blocks allocated, reference (link) count, and so forth. What is the link count of a directory, as the number of entries in the directory changes? Useful interfaces: stat(), naturally.


```c
#include <stdio.h>     // fprintf, perror
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdlib.h>


int main(int argc, char *argv[]) {
    struct stat sb;

    stat(argv[1], &sb);

    printf("파일 종류:                     ");

    int info = sb.st_mode & S_IFMT;

    if (info == S_IFDIR){
        printf("directory\n");
    }
    else if (info == S_IFREG){
        printf("regular file\n");
    }
    else if (info == S_IFBLK){
        printf("BLK\n");
    }
    else if (info == S_IFCHR){
        printf("character device\n");
    }
    else if (info == S_IFIFO){
        printf("FIFO\n");
    }
    else if (info == S_IFLNK){
        printf("symbolic link\n");
    }
    else{
        printf("unknown");
    }
    

    printf("File size:                     %lld bytes\n", (long long) sb.st_size);
    printf("Number of blocks allocated:    %lld\n", (long long) sb.st_blocks);
    printf("Reference (link) count:        %ld\n", (long) sb.st_nlink);
}
```


```c
2019312219@swji:~/hw/8$ ./stat ./
파일 종류:                     directory
File size:                     4096 bytes
Number of blocks allocated:    8
Reference (link) count:        2
2019312219@swji:~/hw/8$ mkdir test
2019312219@swji:~/hw/8$ ./stat ./
파일 종류:                     directory
File size:                     4096 bytes
Number of blocks allocated:    8
Reference (link) count:        3
```



디렉토리내 새로운 폴더를 추가하면 다음과 같다. 잘 작동한다.




#### 2. List Files: Write a program that lists files in the given directory. When called without any arguments, the program should just print the file names. When invoked with the -l flag, the program should print out information about each file, such as the owner, group, permissions, and other information obtained from the stat() system call. The program should take one additional argument, which is the directory to read, e.g., myls -l directory. If no directory is given, the program should just use the current working directory. Useful interfaces: stat(), opendir(), readdir(), getcwd().



```c
#include <stdio.h>     // fprintf, perror
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>    // getopt
#include <stdlib.h>    // exit, EXIT_FAILURE, EXIT_SUCCESS
#include <dirent.h>    // opendir, readdir, closedir
#include <string.h>
#include <stdbool.h>


int main(int argc, char *argv[]) {
    char * path = ".";
    struct stat sb;
    int flag;
    bool flag_l = false;
    DIR *dp;
    opterr = 0;


    while ((flag = getopt(argc, argv, "l:")) != -1) {
        if (flag == 'l'){
            path = optarg;
            flag_l = true;
        }
        else{
            if (optopt == 'l')
                flag_l = true;
        }
    }

    stat(path, &sb);

    if (S_ISDIR(sb.st_mode)) {
        dp = opendir(path);
        struct dirent *d;
        if (flag_l) printf("<size>\t\t<owner>\t\t<mode>\t\t<link count>\t<name>\n");
        else printf("<name>\n");
        while ((d = readdir(dp)) != NULL) {
            if (flag_l) {
                char finalpath[256] = "";
                strncpy(finalpath, path, strlen(path));
                strncat(finalpath, "/", 1);
                strncat(finalpath, d->d_name, strlen(d->d_name));
                stat(finalpath, &sb);
                printf("%lld\t\t", (long long) sb.st_size);
                printf("%ld\t\t", (long) sb.st_uid);
                printf("%lo\t\t", (unsigned long) sb.st_mode);
                printf("%ld\t\t", (long) sb.st_nlink);
            }
            printf("%s\n", d->d_name);
        }
    } 
}
```

```c
2019312219@swji:~/hw/8$ ./list
<name>
search.c
..
r
stat.c
list
list.c
.
tail
stat
tail.c
test
search
2019312219@swji:~/hw/8$ ./list -l
<size>          <owner>         <mode>          <link count>    <name>
1392            1115            100644          1               search.c
4096            1115            40755           9               ..
114             1115            100755          1               r
989             1115            100644          1               stat.c
16808           1115            100750          1               list
1503            1115            100644          1               list.c
4096            1115            40755           3               .
12248           1115            100750          1               tail
11888           1115            100755          1               stat
853             1115            100644          1               tail.c
4096            1115            40750           2               test
17760           1115            100750          1               search
2019312219@swji:~/hw/8$ ./list -l ./test
<size>          <owner>         <mode>          <link count>    <name>
4096            1115            40755           3               ..
4096            1115            40750           2               .
12              1115            100644          1               test.txt

```


잘 출력되는 모습을 보인다



#### 3. Tail: Write a program that prints out the last few lines of a file. The program should be efficient, in that it seeks to near the end of the file, reads in a block of data, and then goes backwards until it finds the requested number of lines; at this point, it should print out those lines from beginning to the end of the file. To invoke the program, one should type: mytail -n file, where n is the number of lines at the end of the file to print. Useful interfaces: stat(), lseek(), open(), read(), close().



```c
#include <stdio.h>     // fprintf, perror
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>    // lseek, read
#include <stdlib.h>    // exit, atoi
#include <fcntl.h>     // open
#include <string.h>


int main(int argc, char *argv[]) {

    struct stat sb;
    int fd;
    int offset;
    int line_ptr;
    char * path = "";

    path = argv[2];
    stat(path, &sb);
    fd = open(path, O_RDONLY);
    lseek(fd, -1, SEEK_END);

    line_ptr = atoi(argv[1]);
    line_ptr = (-line_ptr) + 1;

    char buff[sb.st_size];

    while (1) {
        read(fd, buff, 1);
        if (buff[0] == '\n') line_ptr--;
        offset = lseek(fd, -2, SEEK_CUR);
        if ((offset == -1) || (line_ptr == 0)) break;
    }

    lseek(fd, 2, SEEK_CUR);
    memset(buff, 0, sb.st_size);
    read(fd, buff, sb.st_size);
    printf("%s", buff);
    close(fd);
}
```


```c
2019312219@swji:~/hw/8$ ./tail -2 test/test.txt
hello world!
hello world!
```

잘 출력하는 모습을 보인다.


#### 4. Recursive Search: Write a program that prints out the names of each file and directory in the file system tree, starting at a given point in the tree. For example, when run without arguments, the program should start with the current working directory and print its contents, as well as the contents of any sub-directories, etc., until the entire tree, root at the CWD, is printed. If given a single argument (of a directory name), use that as the root of the tree instead. Refine your recursive search with more fun options, similar to the powerful find command line tool. Useful interfaces: figure it out.


```c
#include <stdio.h>     // fprintf, perror
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>    // getopt
#include <stdlib.h>    // exit
#include <string.h>    // strncmp, strlen
#include <dirent.h>    // opendir, readdir, closedir
#include <errno.h>     // EACCES
#include <limits.h>    // INT_MAX
#include <regex.h>     // regcomp, regexec


void search(char * pathname, regex_t * preg) {
    DIR *dir;
    struct dirent *d;
    errno = 0;
    dir = opendir(pathname);
    while ((d = readdir(dir)) != NULL) {
        char path[256] = "";
        strncpy(path, pathname, strlen(pathname));
        strncat(path, "/", 1);
        strncat(path, d->d_name, strlen(d->d_name));

        if ((d->d_name)[0] != '.') {
            printf("%s\n", path);
            if (d->d_type == DT_DIR)
                search(path, preg);
        }
    }
    closedir(dir);
}

int main(int argc, char *argv[]) {
    char * pathname = ".";
    char * pt = "";
    struct stat sb;
    int opt;
    int is_name = 0;
    regex_t preg;

    opt = getopt(argc, argv, "n:");
    if (opt == 'n'){
        pt = optarg;
        is_name = 1;
    }
    if (optind == argc - 1) pathname = argv[optind];
    
    regcomp(&preg, pt, 0);
    stat(pathname, &sb);
    regexec(&preg, pathname, 0, NULL, 0);
    printf("%s\n", pathname);

    if (is_name) search(pathname, &preg);
    else search(pathname, NULL);
}
```


```c
2019312219@swji:~/hw/8$ ./search
.
./search.c
./r
./stat.c
./list
./list.c
./tail
./stat
./tail.c
./test
./test/test.txt
./search
2019312219@swji:~/hw/8$ ./search ./test
./test
./test/test.txt
```

잘 출력하는 모습을 보인다.

## Source

- 『Operating Systems: Three Easy Pieces』 *-Remzi H. Arpaci-Dusseau 지음*  
  [https://pages.cs.wisc.edu/~remzi/OSTEP/](https://pages.cs.wisc.edu/~remzi/OSTEP/)