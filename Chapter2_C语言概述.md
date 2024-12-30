### return 0
为什么main函数是return 0?因为linux环境下要求进程返回一个数字码表示
执行的状态，0在linux下是成功执行的意思。所以在window环境下不写也可以，
现在在大多数OJ上不写也可以。但为了保证规范(这是一个工程师的基本素养)
，要求自己该return就return。

#### 函数入口
C语言的用户函数(显然系统函数更复杂)入口是main()，一切用户程序的执行都从这开始。
这是人为地、从编译原理的角度规定的。
但既然是人为的，也可以改编译器的设计，或者用attribute机制：
```cpp
__attribute((constructor))void before()
{
printf(“before main\n”);\\这样before就能在main函数前执行
}
```
#### 函数原型
C标准建议要为程序中用到的所有函数提供原型，比如
```cpp
/* two_func.c -- 一个文件中包含两个函数 */
#include <stdio.h>
void butler(void); /* ANSI/ISO C函数原型 */
int main(void)
{
     printf("I will summon the butler function.\n");
     butler();
     printf("Yes. Bring me some tea and writeable DVDs.\n");

     return 0;
}
void butler(void) /* 函数定义开始 */
{
     printf("You rang, sir?\n");
}
```
中的void butler(void);但现在已经很少看到这种写法了。

#### 本章小结
对编译器来说，几乎正确也是错误，必须注意细节问题。


