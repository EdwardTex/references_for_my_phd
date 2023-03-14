# Core dump cases

```
*** glibc detected *** free(): invalid pointer:
*** glibc detected *** malloc(): memory corruption:
*** glibc detected *** double free or corruption (out): 0x00000000005c18a0 ***
*** glibc detected *** double free or corruption (!prev): 0x0000000000a01f40 ***
*** glibc detected *** corrupted double-linked list: 0x00000000005ab150 ***
```

你是否遇到过这样的情况，太沮丧了，程序总是无端coredump，gdb到core文件里面也看不出个所以然来，这对于一个大型的商业系统来说太令人恐怖了，事故随时可能发生。

遇到棘手的问题，慌张是没用的，解决不了任何问题。先坐下来，喝杯茶，舒缓一下神经。

内存问题始终是c++程序员需要去面对的问题，这也是c++语言的门槛较高的原因之一。通常我们会犯的内存问题大概有以下几种：

1.内存重复释放，出现double free时，通常是由于这种情况所致。

2.内存泄露，分配的内存忘了释放。

3.内存越界使用，使用了不该使用的内存。

4.使用了无效指针。

5.空指针，对一个空指针进行操作。

对于第一种和第二种，第五种情况，就不用多说，会产生什么后果大家应该都很清楚。

第四种情况，通常是指操作已释放的对象，如：

1.已释放对象，却再次操作该指针所指对象。

2.多线程中某一动态分配的对象同时被两个线程使用，一个线程释放了该对象，而另一线程继续对该对象进行操作。

我们重点探讨第三种情况，相对于另几种情况，这可以称得上是疑难杂症了（第四种情况也可以理解成内存越界使用）。

内存越界使用，这样的错误引起的问题存在极大的不确定性，有时大，有时小，有时可能不会对程序的运行产生影响，正是这种不易重现的错误，才是最致命的，一旦出错破坏性极大。

什么原因会造成内存越界使用呢？有以下几种情况，可供参考：
例1：
```
char buf[32] = {0};
for(int i=0; i<n; i++)// n < 32 or n > 32
{
    buf[i] = 'x';
}
....      
```
例2:
```
char buf[32] = {0};
string str = "this is a test sting !!!!";
sprintf(buf, "this is a test buf!string:%s", str.c_str()); //out of buffer space
....   
```
例3:
```
string str = "this is a test string!!!!";
char buf[16] = {0};
strcpy(buf, str.c_str()); //out of buffer space
```
类似的还存在隐患的函数还有：strcat,vsprintf等
同样，memcpy, memset, memmove等一些内存操作函数在使用时也一定要注意。
        
当这样的代码一旦运行，错误就在所难免，会带来的后果也是不确定的，通常可能会造成如下后果：

1.破坏了堆中的内存分配信息数据，特别是动态分配的内存块的内存信息数据，因为操作系统在分配和释放内存块时需要访问该数据，一旦该数据被破坏，以下的几种情况都可能会出现。
```
*** glibc detected *** free(): invalid pointer:
*** glibc detected *** malloc(): memory corruption:
*** glibc detected *** double free or corruption (out): 0x00000000005c18a0 ***
*** glibc detected *** corrupted double-linked list: 0x00000000005ab150 ***        
```
2.破坏了程序自己的其他对象的内存空间，这种破坏会影响程序执行的不正确性，当然也会诱发coredump，如破坏了指针数据。

3.破坏了空闲内存块，很幸运，这样不会产生什么问题，但谁知道什么时候不幸会降临呢？

通常，代码错误被激发也是偶然的，也就是说之前你的程序一直正常，可能由于你为类增加了两个成员变量，或者改变了某一部分代码，coredump就频繁发生，而你增加的代码绝不会有任何问题，这时你就应该考虑是否是某些内存被破坏了。

排查的原则，首先是保证能重现错误，根据错误估计可能的环节，逐步裁减代码，缩小排查空间。
检查所有的内存操作函数，检查内存越界的可能。常用的内存操作函数：
```
sprintf snprintf
vsprintf vsnprintf
strcpy strncpy strcat
memcpy memmove memset bcopy
```
如果有用到自己编写的动态库的情况，要确保动态库的编译与程序编译的环境一致。
