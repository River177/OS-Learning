# Lab: Xv6 and Unix utilities

## sleep ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

Implement the UNIX program `sleep` for xv6; your `sleep` should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file `user/sleep.c`.

#### solution

实现sleep功能，只要调用已有的sleep函数和atoi方法即可。

```c
// user/sleep.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if(argc != 2){
    fprintf(2, "Error! Must pass an argument.\n");
    exit(1);
  }

  sleep(atoi(argv[1]));

  exit(0);
}
```



## pingpong ([easy](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file `user/pingpong.c`.

#### solution

写一个pipe并完成父子进程间的信息传递，通过查看pipe()和fork()方法完成。

```c
// user/pingpong.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
    int fd[2];
    if (pipe(fd) < 0) {
        fprintf(2, "pipe error");
        exit(1);
    }
    char msg = 'a';
    int pid = fork();

    if(pid == 0){
        char rec;
        int n = read(fd[0], &rec, 1);
        if(n > 0){
            printf("%d: received ping\n", getpid());
            write(fd[1], &rec, 1);
            exit(0);
        }
        else{
            fprintf(2, "Error!");
            exit(1);
        }
    }else{
        write(fd[1], &msg, 1);
        char rec;
        int n = read(fd[0], &rec, 1);
        if(n > 0){
            printf("%d: received pong\n", getpid());
            exit(0);
        }
        else{
            fprintf(2, "Error!");
            exit(1);
        }
    }
}
```



## primes ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))/([hard](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file `user/primes.c`.

#### solution

1. **主函数 main：**

- 创建一个管道 fd 并 fork 出第一个子进程。

- 父进程负责生成数字 2 到 35，通过写端传送到管道中，并在写完后关闭写端，然后等待子进程退出。

- 子进程关闭写端，调用 sieve(fd[0]) 启动素数筛选过程。

2. **sieve 函数：**

- 从传入的管道中读取第一个数字，将其作为当前素数并打印（格式为 "prime %d\n"）。

- 为过滤不被该素数整除的数字创建新的管道 pfd，然后 fork 出一个子进程。

- 父进程负责从原管道中读取每个数字，若数字不能被当前素数整除，则写入新管道。写完后关闭新管道的写端，并等待子进程退出。

- 子进程关闭新管道写端，递归调用 sieve(pfd[0]) 来处理新管道中的数字。

- 当读到 EOF（即 read 返回的字节数不等于 sizeof(int)）时退出。

```c
// user/primes.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void
sieve(int fd)
{
  int prime, n;

  if (read(fd, &prime, sizeof(prime)) != sizeof(prime))
    exit(0);
  printf("prime %d\n", prime);

  int pfd[2];
  if (pipe(pfd) < 0) {
    fprintf(2, "pipe error\n");
    exit(1);
  }

  int pid = fork();
  if (pid < 0) {
    fprintf(2, "fork error\n");
    exit(1);
  }

  if (pid == 0) {
    close(pfd[1]);
    sieve(pfd[0]);
    close(pfd[0]);
    exit(0);
  } else {
    close(pfd[0]);
    while (read(fd, &n, sizeof(n)) == sizeof(n)) {
      if (n % prime != 0)
        write(pfd[1], &n, sizeof(n));
    }
    close(pfd[1]);
    wait(0);
  }
  exit(0);
}

int
main(int argc, char *argv[])
{
  int fd[2];
  if (pipe(fd) < 0) {
    fprintf(2, "pipe error\n");
    exit(1);
  }
  int pid = fork();
  if (pid < 0) {
    fprintf(2, "fork error\n");
    exit(1);
  }
  if (pid == 0) {
    close(fd[1]);
    sieve(fd[0]);
    close(fd[0]);
    exit(0);
  } else {
    close(fd[0]);
    for (int i = 2; i <= 35; i++) {
      write(fd[1], &i, sizeof(i));
    }
    close(fd[1]);
    wait(0);
  }
  exit(0);
}
```



## find ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file `user/find.c`.

#### solution

这个lab涉及到很多文件操作，很多函数通过chatgpt进行了解，同时要考虑到不能陷入死循环和文件路径的完整性（利用strcpy），进行字符串的拼接。

```	c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

// 取出 path 最后的文件名部分（去掉前面的目录）
char*
fmtname(char *path)
{
  static char buf[DIRSIZ+1];
  char *p;

  // 找到字符串最后一个 '/'
  for(p = path + strlen(path); p >= path && *p != '/'; p--)
    ;
  p++; // p 指向最后一个 '/' 后的第一个字符

  // 拷贝到 buf
  memmove(buf, p, strlen(p)+1);
  return buf;
}

void
find(char *path, char *target)
{
  char buf[512];
  int fd;
  struct dirent de;
  struct stat st;

  // 打开 path
  if ((fd = open(path, 0)) < 0) {
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  // 获取 path 的 stat
  if (fstat(fd, &st) < 0) {
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }

  // 根据文件类型分支
  switch (st.type) {

  // 如果 path 本身是一个文件，比较其文件名是否与 target 相同，相同则打印
  case T_FILE: {
    // 拿到 path 最后的文件名
    char *filename = fmtname(path);
    if (strcmp(filename, target) == 0) {
      printf("%s\n", path);
    }
    break;
  }

  // 如果是目录，则遍历它的每个子项
  case T_DIR: {
    // 如果路径太长就退出（保护性判断）
    if (strlen(path) + 1 + DIRSIZ + 1 > sizeof(buf)) {
      fprintf(2, "find: path too long\n");
      break;
    }

    // 读取目录项
    while (read(fd, &de, sizeof(de)) == sizeof(de)) {
      // 如果 inum=0，说明这是个空的目录项；跳过 "." 或 ".."
      if (de.inum == 0) 
        continue;
      if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0)
        continue;

      // 把 dirent.name 搬到 dname 并确保结尾有 '\0'
      char dname[DIRSIZ+1];
      memmove(dname, de.name, DIRSIZ);
      dname[DIRSIZ] = '\0';

      // 在 buf 里拼出新的完整子路径
      strcpy(buf, path);
      int len = strlen(buf);
      // 如果末尾没有 '/', 补上 '/'
      if (len > 0 && buf[len-1] != '/') {
        buf[len] = '/';
        buf[len+1] = '\0';
        len++;
      }
      // 在末尾再拼上目录项名字
      strcpy(buf+len, dname);

      // 对新的子路径做 stat，判断它是文件还是目录
      struct stat st_sub;
      if (stat(buf, &st_sub) < 0) {
        fprintf(2, "find: cannot stat %s\n", buf);
        continue;
      }

      // 如果这个子项是文件，且名字等于 target，就打印出来
      if (st_sub.type == T_FILE) {
        if (strcmp(dname, target) == 0) {
          printf("%s\n", buf);
        }
      } 
      // 如果是子目录，则递归继续查找
      else if (st_sub.type == T_DIR) {
        find(buf, target);
      }
    }
    break;
  }

  }
  // 记得关闭文件描述符
  close(fd);
}

int
main(int argc, char *argv[])
{
  if (argc < 3) {
    fprintf(2, "usage: find <path> <target>\n");
    exit(1);
  }

  find(argv[1], argv[2]);
  exit(0);
}
```



## xargs ([moderate](https://pdos.csail.mit.edu/6.828/2021/labs/guidance.html))

Write a simple version of the UNIX xargs program: read lines from the standard input and run a command for each line, supplying the line as arguments to the command. Your solution should be in the file `user/xargs.c`.

### solution 

原本以为前面的输出会直接作为xargs的argv参数，后面发现是将其输出到标准输入，直接从标准输入进行读取并解析即可。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"

// 去除字符串首尾的双引号（如果存在的话），修改原字符串
void trim_quotes(char *s) {
  int len = strlen(s);
  if(len > 0 && s[0] == '"'){
    // 删除开头的双引号：将整个字符串向左移动1个字符
    memmove(s, s+1, len);
    len--;
  }
  if(len > 0 && s[len-1] == '"'){
    // 删除末尾的双引号：直接置为'\0'
    s[len-1] = '\0';
  }
}

// 声明辅助函数原型
void run_cmd_with_arg(int base_argc, char *base_argv[], char *arg);

// 一个简易版的 xargs：
//   xargs [-n 1] <cmd> [cmd_args...]
// 从标准输入中读取多行，每行作为额外参数追加给 <cmd>，并执行一次。
// 本程序仅支持 -n 1 选项，确保每次只追加一个参数。
int
main(int argc, char *argv[])
{
  int num = 1;      // 每次追加的参数个数，目前仅支持 1
  int cmd_start = 1; // 命令开始的下标

  if (argc < 2) {
    fprintf(2, "Usage: xargs [-n 1] <cmd> [cmd_args...]\n");
    exit(1);
  }

  // 检查是否有 "-n" 选项
  if (argc >= 3 && strcmp(argv[1], "-n") == 0) {
    num = atoi(argv[2]);
    if (num != 1) {
      fprintf(2, "Only -n 1 is supported\n");
      exit(1);
    }
    cmd_start = 3;
    if (argc < 4) {
      fprintf(2, "Usage: xargs -n 1 <cmd> [cmd_args...]\n");
      exit(1);
    }
  }

  // 固定命令及其参数的个数
  int base_argc = argc - cmd_start;
  char **base_argv = &argv[cmd_start];

  char buf[512];
  int idx = 0;
  while (1) {
    char c;
    int n = read(0, &c, 1); // 每次读取 1 字节
    if (n < 1) {
      // 读不到数据，可能是 EOF 或 read 出错
      break;
    }

    // 若遇到反斜杠，则检查是否为 "\n" 序列
    if (c == '\\') {
      char next;
      int m = read(0, &next, 1);
      if (m > 0 && next == 'n') {
        buf[idx] = '\0';
        run_cmd_with_arg(base_argc, base_argv, buf);
        idx = 0;
        continue;
      } else {
        // 不是 "\n"，把 '\' 和 next 都保存
        buf[idx++] = c;
        if (m > 0) {
          buf[idx++] = next;
        }
        continue;
      }
    }
    
    if (c == '\n') {
      buf[idx] = '\0';
      run_cmd_with_arg(base_argc, base_argv, buf);
      idx = 0;
    } else {
      buf[idx++] = c;
      if (idx >= sizeof(buf) - 1) {
        fprintf(2, "xargs: input line too long\n");
        exit(1);
      }
    }
  }

  // 处理最后一行没有 '\n' 结束的情况
  if (idx > 0) {
    buf[idx] = '\0';
    run_cmd_with_arg(base_argc, base_argv, buf);
  }

  exit(0);
}

// 将从标准输入中读取的一行作为一个参数，追加到固定命令参数之后，并执行该命令
void run_cmd_with_arg(int base_argc, char *base_argv[], char *arg)
{
  // 去除参数首尾的双引号（如果存在）
  trim_quotes(arg);

  // 构造用于 exec() 的参数列表：
  // 先复制原命令的参数（base_argv[0..base_argc-1]），再把从标准输入读到的一行追加在末尾
  char *nargv[MAXARG];
  int i;
  for (i = 0; i < base_argc; i++) {
    nargv[i] = base_argv[i];
  }
  nargv[base_argc] = arg;      // 追加的参数
  nargv[base_argc + 1] = 0;      // 用 NULL 结尾

  int pid = fork();
  if (pid < 0) {
    fprintf(2, "xargs: fork failed\n");
    exit(1);
  }
  if (pid == 0) {
    exec(nargv[0], nargv);
    fprintf(2, "xargs: exec %s failed\n", nargv[0]);
    exit(1);
  } else {
    wait(0);
  }
}
```



