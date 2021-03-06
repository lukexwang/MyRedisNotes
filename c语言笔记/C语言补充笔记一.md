## C语言补充笔记一

- `malloc()`:动态分配空间并存放数据，<mark style="color:red;">**malloc() 不初始化，里边数据是未知的垃圾数据**</mark>

```c
buffer = (char*)malloc(i+1);  // 字符串最后包含 \0
if(buffer==NULL) exit(1);  // 判断是否分配成功
```

- `calloc()`:动态分配空间，<mark style="color:red;">**并初始化为**</mark>

```c
int i=100,n;
int * pData;
pData = (int*) calloc (i,sizeof(int)); //分配100个int的空间
if (pData==NULL) exit (1);
for (n=0;n<i;n++)
{
   printf ("请输入数字 #%d：",n+1);
   scanf ("%d",&pData[n]);
}
printf ("你输入的数字为：");
for (n=0;n<i;n++) printf ("%d ",pData[n]);
free (pData);
```

`calloc()`与`malloc()`一个重要区别: **calloc() 在动态分配完内存后，自动初始化该内存空间为零，而 malloc() 不初始化，里边数据是未知的垃圾数据。**

下面的两种写法是等价的:

```c
// calloc() 分配内存空间并初始化
char *str1 = (char *)calloc(10, 2);

// malloc() 分配内存空间并用 memset() 初始化
char *str2 = (char *)malloc(20);
memset(str2, 0, 20);
```

- 给一个struct做初始化,或者分配内存:

```c
struct addrinfo *hints=calloc(1, sizeof(struct addrinfo));
or
struct addrinfo *hints=(struct addrinfo*)malloc(sizeof(struct addrinfo));

memset(hints, 0, sizeof(struct addrinfo));
hints->ai_addr = NULL;
hints->ai_addrlen = 0
```

- `atoi()`:`char*`转换为`int`:

```c
#include <stdlib.h>
int atoi(const char *string);

示例:
int i;
char *s;
s = " -9885";
i = atoi(s);     /* i = -9885 */
```

- `sprintf()`:将int转换为格式化字符:

```c
#include <stdio.h>

int i=8;
char ret[20] = {0}; //先全部初始化为0
(ret, "%d\n", i);
```

- `isspace()`:判断某个字符是否是空格:

标准的空格有这些

```c
' '   (0x20)	space (SPC)
'\t'	(0x09)	horizontal tab (TAB)
'\n'	(0x0a)	newline (LF)
'\v'	(0x0b)	vertical tab (VT)
'\f'	(0x0c)	feed (FF)
'\r'	(0x0d)	carriage return (CR)
```

使用示例,<mark style="color:red;">**去除某个string前后的空格**</mark>:

```c
#include <ctype.h>
char *trimwhitespace(char *str)
{
    char *end;

    // Trim leading space
    while (isspace((unsigned char)*str))
        str++;

    if (*str == 0) // All spaces?
        return str;

    // Trim trailing space
    end = str + strlen(str) - 1;
    while (end > str && isspace((unsigned char)*end))
        end--;

    // Write new null terminator character
    end[1] = '\0';

    return str;
}
```

- 经常看到Redis中从一个`char *`类型中解析longlong or int,其实是非常合理的。

例如:

```c
int a = 97;
size_t len = sizeof(int);
char p01[10];
memcpy(p01, (char *)&a, len);//将int拷贝到char *中保存,因为a=97,4个字节就用了一个1个字节,其他3个字节是'\0'

int tmp = 0;
int *b = &tmp;
*b = *(int *)p01; //p01的前 4个字节解析成 int赋值给 *b
printf("intlen=>%d,p01=>%s,strlen=>%d,*b==%d,tmp=>%d\n", len, p01, strlen(p01), *b, tmp);

结果打印:
intlen=>4,p01=>a,strlen=>1,*b==97,tmp=>97 //字符'a'的ascii码值就是97,所以这里p01==a
```

- 解析二进制文件 或者 应用层的网络协议,<mark style="color:red;">**一般有下列的思路来定义消息，以保证完整进行读取**</mark>:

  - <mark style="color:red;">**定长消息**</mark>;

  - <mark style="color:red;">**消息尾部添加特殊分隔符，如Echo服务器,使用'\n'作为分割符**</mark>; 

  - <mark style="color:red;">**将消息分为header和body，在header中提供bodu总长度。这种分包方式称为LTV(length,type,value)。这是应用最广泛的策略，如HTTP协议。当从header中获得body长度后，io.ReadFULL函数会读取指定字节流，从而解析应用层消息**</mark>。

Redis的RESP协议，如AOF文件就是，分隔符和LTV包的结合使用。

不过Redis中也有定长消息的使用，下面是一个定长消息的示例，我们可以朝fd(fd可以是文件，可以是pipeline，可以是网络socket)中写入定长的struct student，读取时候也是定长去读取。

注意: **下面程序运行的前提是 struct student中没有指针**

```c
struct student{
  int id;
  char name[20];
  int age;
};
char *fname = "a.txt";
int f01 = open(fname, O_RDWR | O_CREAT);

//开始定长的去写入，此时f01中保存的就是二进制信息
write(f01, &s01, sizeof(struct student));
write(f01, &s02, sizeof(struct student));
close(f01);

int f02 = open(fname, O_RDWR | O_CREAT);
struct student s03;
//定长的去读取，读取的二进制数据直接解析成struct student
read(f02, &s03, sizeof(struct student));
printf("s03=>{%d,%s,%d}\n", s03.id, s03.name, s03.age);
read(f02, &s03, sizeof(struct student));
printf("s03=>{%d,%s,%d}\n", s03.id, s03.name, s03.age);
close(f02);
return 0;
```

Redis定长消息用在执行BGSAVE时，child进程 通过pipeline 不断向 parent进程 传输RDB备份进度。

![image-20211122203335223](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211122203335223.png)

![image-20211122203415897](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20211122203415897.png)

- `strcpy`和`strdup`的区别

  参考问题: [strcpy vs strdup](https://stackoverflow.com/questions/14020380/strcpy-vs-strdup)

  - `strcpy`和`strncpy`是C语言标准版的一部分。在使用这俩函数时，调用者**必须自己确保将所需内存分配好**；
  - 相反，`strdup`是Posix标准里的函数，<mark style="color:red" >**他会自动为你分配内存(函数中隐藏这一个`malloc`)**</mark>。然后返回新分配的内存的地址；新地址的内存由你自己释放。
  - `strcpy(ptr2,ptr1)`等价于 `while(*ptr2++ = *ptr1++)`;
  - `strdup`等价于

  ```c
  ptr2 = malloc(strlen(ptr1)+1);
  strcpy(ptr2,ptr1);
  ```

- 为了节省内存，C/C++中常量字符串放到一个单独的内存区域。

  **当几个指针赋予了相同的常量字符串时，它实际上会指向相同的内存地址**

  **但用常量字符串初始化数组时，则不是相同的地址**

  ```cpp
  char str1[]="hello world";
  char str2[]="hello world";
  
  char* str3="hello world"; # 指向同一块不可变的内存区域
  char* str4="hello world";
  
  if(str1 == str2)
    printf("str1 and str2 are same.\n");
  else
    printf("str1 and str2 are not same.\n");
  
  if(str3 == str4)
    printf("str3 and str4 are same.\n");
  else
    printf("str3 and str4 are not same.\n");
  
  最后结果:
  str1 and str2 are not same.
  str3 and str4 are same.
  ```

- 数组和指针的区别: 

  当我们声明一个数组时，数组的名字也是一个指针，该指针指向数组的第一个元素。

  我们可以用指针来访问数组。不过需要注意的是：C/C++中没有记录数组的大小，因此指针访问数组元素时，需确保数组没有越界。

  如:

  ```cpp
  #数组作为函数的参数进行传递时,数组会自动退化为同类型的指针
  #尽管该函数的形参data被声明为数组
  int GetSize(int data[]) { return sizeof(data); }
  
  int _tmain(int argc, char *argv[]) {
    int data1[] = {1, 2, 3, 4, 5};
    int size1 = sizeof(data1); #求数组的大小,数组中包含5个整数,每个整数4个字节
  
    int *data2 = data1;
    int size2 = sizeof(data2); #尽管指向一个数组,但本质还是指针
  
    int size3 = GetSize(data1); #数组作为函数的参数进行传递时,数组会自动退化为同类型的指针
  
    printf("%d, %d, %d", size1, size2, size3);
  }
  最后结果:
  20,4,4
  ```

  