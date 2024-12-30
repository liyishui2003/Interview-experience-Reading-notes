#### C标准里的数据类型概述
- int/long/short 用来表示整数，其中unsigned是无符号类型，能表示的范围是对应有符号类型的2倍数
- char 用来表示字符，比如#、$、%、*和较小的整数(视具体情况而定，[-128,127]或[0,255])
- float/double/long double表示带小数点的数
- _Bool 布尔值(居然是带下划线的)
- _Complex/_Imaginary 复数和虚数

#### 位、字节和字
- 位(bit)，存储0或1，内存最小的构成
- 字节(byte)，1字节为8位
- 字(word)，计算机的字长，也就是计算机进行一次运算能支持的最大长度
  
### int
printf函数不止支持打印"%d"，它还可以打印八进制(%o)、十六进制(%x)。
int类型有很多的修饰符，比如short、unsigned，这些修饰符还可以一定程度上组合。
区别在于：short和int相比，占用存储空间更少，所以数值更小，long则相反。
而unsigned的存在能保证不出现负数，且表示的范围扩大一倍。
如果用signed，意义只在于强调"使用有符号类型"这一意图，对类型本身没有实质改变。
所以实际上：short、short int、signed short、signed short int都表示同一类型。
现在PC的常见设置是：long long占64位，long占32位，short占16位，int占16位或32位（依计算机的自然字长而定)。
