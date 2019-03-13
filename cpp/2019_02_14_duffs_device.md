
先来一段神奇的代码

```c

int func(int num, int *first)
{
    int *array = first;
    switch(num % 8)
        case 0:
            do {
                *(array++) = int((array - first - 1));
                case 7: *(array++) = int((array - first - 1));
                case 6: *(array++) = int((array - first - 1));
                case 5: *(array++) = int((array - first - 1));
                case 4: *(array++) = int((array - first - 1));
                case 3: *(array++) = int((array - first - 1));
                case 2: *(array++) = int((array - first - 1));
                case 1: *(array++) = int((array - first - 1));
            } while (array - first < num);
}

int main()
{   
    int a[10];
    func(10, a);
    for (int i = 0; i < 10; i++)
        printf("i = %d, a[%d] = %d\n", i, i, a[i]);

    return 0;
}
```
输出结果
```
debian:~/LearnTest/cppswitch$ ./a.out 
i = 0, a[0] = 0
i = 1, a[1] = 1
i = 2, a[2] = 2
i = 3, a[3] = 3
i = 4, a[4] = 4
i = 5, a[5] = 5
i = 6, a[6] = 6
i = 7, a[7] = 7
i = 8, a[8] = 8
i = 9, a[9] = 9
```
以上的用法叫Duff's Device，是早期用于人肉展开循环提高执行效率的一种方式。由于现代架构的流水线机制的优化，这种展开方式在当代已经无法提升效率了，但是展现的语法特性却偶尔还有一些用处。

这个用法能成立的原因是：
- 按照C语言规范规范，在switch控制语句内，条件标号（case）可以出现在任意子语句之前，充作其前缀。
- 同时由于fall-through特性，若未加入break语句，则在switch语句在根据条件判定，跳转到对应标号，并开始执行后，控制流会无视其余条件标号限定，一直执行到switch嵌套语句的末尾。

可以巧妙利用这些特性来使代码按照运行期情况来分段执行，达到类似于协程的目的。

```
#define YIELD_I(n) \
    flag = n;\
    goto yield_return; \
    case n:
    
#define YIELD YIELD_I(__COUNTER__ + 1)

int func()
{
    int &flag = GetFlag();
    switch (flag)
        case 0:
        {
            ...
            /*代码正文1*/
            YIELD
            /*代码正文2*/
            YIELD
            /*代码正文3*/
            ...
            goto end;
        }
end:
    flag = -1;
    DoEnd();
yield_return:
    return;
}
```
上面示例的代码，一旦某处需要挂起然后执行异步，就可以塞一个YIELD，然后下次再次运行的时候，由于flag是保存好的，因此会直接跳到上次执行的位置下面执行，达到类似协程的目的。


上面的代码是简化后的示意，可能还需要修改后才能实际运行。原来准备的实际用例经过推敲认为有些画蛇添足，思考后再添加这部分
