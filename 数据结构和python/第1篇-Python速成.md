---
title: 第 1 篇：Python 速成
layout: article
---
# 第 1 篇：Python 速成——看懂后面所有代码所需的最小语法

这篇是整套文章的地基：只讲**看懂后面七篇代码所必需**的那点 Python。你完全零基础也没关系，只要会开电脑打字就行。看完这一篇，你应该能读懂一段两百行以内的数据结构代码，并且自己动手改两行跑起来。

先说清楚一件事：这篇文章**不是** Python 教程。Python 能讲的东西太多了，装饰器、生成器、多线程、虚拟环境……这些期末考试一个都不考。我们要的是一把能开锁的最小钥匙，不是一整套工具箱。所以下面每个语法点，我都只讲"数据结构代码里真的会用到的那个用法"，多一个字的废话都不写。

准备好了就往下看。建议你打开电脑，把每段代码亲手敲一遍——**看代码和敲代码，是两种完全不同的学习**。看的时候觉得"懂了"，敲的时候才发现少了个冒号。

---

## 一、变量：给数据贴个标签

你可以这样想：变量就是储物柜上的便利贴。

你在柜子里放了个东西（数据），然后在柜门上贴一张纸条写上名字（变量名），下次要找东西，喊名字就行，不用记住柜子是第几号。

```python
name = "小明"      # 把字符串 "小明" 放进名为 name 的柜子
age = 19           # 整数
score = 87.5       # 小数（浮点数 float）
is_pass = True     # 布尔值，只有 True 和 False 两种，注意首字母大写

print(name, age, score, is_pass)
```

运行结果：

```text
小明 19 87.5 True
```

这里面有四个类型，考试里最常出现的是前三个：

| 类型 | 写法 | 数据结构里拿来干嘛 |
| --- | --- | --- |
| `int` 整数 | `1`、`-3`、`0` | 下标、计数、节点值 |
| `float` 小数 | `3.14`、`2.0` | 平均值、复杂度里的常数 |
| `str` 字符串 | `"abc"`、`'a'` | 括号匹配里的字符、姓名 |
| `bool` 布尔 | `True` / `False` | 判断结果、访问标记 |
| `None` 空值 | `None` | 表示"这里还没有东西" |

`None` 你会在链表里天天见到：一个节点如果没有下一个节点，它的 `next` 就是 `None`。注意它是 `None`，不是 `null`，也不是 `NULL`，Python 只认这一个写法。

**注意这里**：Python 是"动态类型"，变量不用提前声明类型，同一个名字可以一会儿装整数一会儿装字符串：

```python
x = 10
x = "现在是字符串了"   # 合法，但不推荐，考试读代码时容易看晕
```

另外，`=` 不是数学里的"等于"，它是**赋值**：把右边算出来的东西，塞进左边的柜子。判断相等用的是 `==`（两个等号）。这个区别后面写 `if` 的时候非常重要。

---

## 二、输出和输入：print 与 input

### print：把东西显示出来

```python
print("hello")
print(1, 2, 3)              # 多个值用逗号隔开，输出时自动加空格
print("a", "b", sep="-")    # 自定义分隔符，输出 a-b
print("不换行", end="")     # 默认会换行，end="" 改成不换行
print("接在后面")
```

输出：

```text
hello
1 2 3
a-b
不换行接在后面
```

考试写代码时，最常用的其实是**格式化输出**（f-string），在字符串前面加个 `f`，里面用 `{}` 把变量塞进去：

```python
name = "小明"
score = 87.5
print(f"{name} 的成绩是 {score} 分")
print(f"平均分保留两位小数：{score:.2f}")
```

输出：

```text
小明 的成绩是 87.5 分
平均分保留两位小数：87.50
```

`{score:.2f}` 里的 `.2f` 意思是"保留两位小数的浮点数"，做复杂度或者平均值题目时很实用，能避免输出 `81.66666666666667` 这种一串。

### input：从键盘读一行

```python
name = input("请输入你的名字：")
print("你好，" + name)
```

**很多同学会在这里踩坑**：`input()` 读回来的**永远是字符串**。哪怕你输入的是 `18`，拿到的也是 `"18"` 而不是 `18`。所以要做数学运算，必须先转换：

```python
age = int(input("请输入年龄："))       # 转成整数
height = float(input("请输入身高："))  # 转成小数
print(age + 1)
```

不转换会怎样？

```python
age = input("请输入年龄：")
print(age + 1)     # 报错：TypeError: can only concatenate str (not "int") to str
```

这条报错的意思是：字符串只能和字符串用 `+` 拼接，不能和整数相加。看到 `TypeError` 并且提到 `str` 和 `int`，八成就是忘了 `int()`。

顺便记两个转换函数：`int("123")` 把字符串变整数，`str(123)` 把整数变字符串。它们在题目里出现的频率高得离谱——比如你要把数字一位一位拆开、或者把结果拼成字符串输出。

---

## 三、if：让程序会做选择题

生活里你每天都在写 if：

> **如果**下雨，就带伞；**否则如果**只是阴天，就带件外套；**否则**，就穿短袖。

Python 把它翻译过来就是这样：

```python
score = 87

if score >= 90:
    level = "优秀"
elif score >= 60:
    level = "及格"
else:
    level = "不及格"

print(level)
```

输出 `及格`。

逐块解释：

- `if 条件:` —— 条件成立（结果是 `True`）才执行下面缩进的那几行。
- `elif` 是 else if 的缩写，可以有零个、一个或多个。它**只有在前面的条件都不成立时才会被检查**。所以上面 `score = 87` 时，第一个 `>= 90` 不成立，才会去检查 `>= 60`。
- `else` 兜底，前面全不成立就执行它。可以没有。
- **冒号 `:` 和缩进不能少**。Python 用缩进（一般是 4 个空格）表示"这几行属于这个 if"。

### 比较运算符和逻辑运算符

| 运算符 | 含义 | 例子 |
| --- | --- | --- |
| `==` | 等于 | `a == 3` |
| `!=` | 不等于 | `a != 3` |
| `>` `<` `>=` `<=` | 大小比较 | `a >= 60` |
| `and` | 并且（两边都真才真） | `60 <= s < 90` 也可写成 `s >= 60 and s < 90` |
| `or` | 或者（一边真就真） | `x < 0 or x > 10` |
| `not` | 取反 | `not flag` |

Python 有个很舒服的写法，别的语言没有：`60 <= score < 90` 可以直接连写，不用 `and`。

### 空值判断的小习惯

后面写链表、树的时候，你会反复看到这种写法：

```python
def visit(node):
    if node is None:
        return          # 走到尽头（空节点）就直接收工
    print(node.val)
```

判断"是不是空"用 `is None`，而不是 `== None`。这是 Python 的惯例，考试写 `== None` 一般也不算错，但 `is None` 更规范。

**踩坑提醒**：`if x:` 和 `if x is not None:` 不是一回事。`if x:` 只要 `x` 是"假值"就进不去，而 `0`、`""`、`[]`、`{}`、`None` 全都是假值。所以如果 `x` 可能是 0 或者空列表，而你只想判断它是不是 `None`，一定要写 `is None` / `is not None`。

---

## 四、循环：while 和 for

循环就是"把同一件事重复做很多遍"。数据结构的代码里，**十行有八行在遍历**，所以这一节要看得特别仔细。

### while：只要条件成立就一直做

```python
n = 5
while n > 0:
    print(n, end=" ")
    n -= 1        # 等价于 n = n - 1，千万别忘
print("发射！")
```

输出：

```text
5 4 3 2 1 发射！
```

`while` 的逻辑是：先看条件，成立就执行循环体，执行完再回来看条件，直到条件不成立为止。

**这里是最经典的死循环现场**：如果忘了 `n -= 1`，`n` 永远是 5，条件永远成立，程序就卡死了，你只能按 `Ctrl + C` 强行中断。写 `while` 的时候养成习惯：**先问自己"哪一行会让条件最终变成假"**。

循环里还能用两个关键字：

```python
i = 0
while True:              # 故意写成永真，靠 break 出来
    i += 1
    if i == 3:
        continue         # 跳过本次循环剩下的部分，直接回去看条件
    if i > 5:
        break            # 整个循环立刻结束
    print(i, end=" ")
```

输出 `1 2 4 5`。注意 `3` 被 `continue` 跳过了。

### for：从一堆东西里挨个取

`for` 是数据结构里出现频率最高的语句，因为它的本职就是"遍历"。

```python
scores = [78, 92, 65]
for s in scores:
    print(s)
```

输出：

```text
78
92
65
```

这句话读作："对于 `scores` 里的每一个元素，把它叫做 `s`，然后执行下面缩进的代码"。你不用关心列表有多长，也不用管下标，`for` 自己会一个个取完。

### range：生成一串整数

`range` 专门用来配合 `for` 数数，它是考试填空题的常客：

```python
for i in range(5):        # 0, 1, 2, 3, 4     —— 从 0 开始，不含 5
    print(i, end=" ")
print()

for i in range(1, 6):     # 1, 2, 3, 4, 5     —— 左闭右开，不含 6
    print(i, end=" ")
print()

for i in range(0, 10, 2): # 0, 2, 4, 6, 8     —— 第三个参数是步长
    print(i, end=" ")
print()

for i in range(5, 0, -1): # 5, 4, 3, 2, 1     —— 步长为负就是倒着来
    print(i, end=" ")
```

输出：

```text
0 1 2 3 4
1 2 3 4 5
0 2 4 6 8
5 4 3 2 1
```

**记住这个口诀：`range(a, b)` 从 a 数到 b-1，含头不含尾。** 数组下标从 0 开始，长度是 n 的数组最后一个下标是 n-1，所以 `range(n)` 刚好能把下标 0 到 n-1 全走一遍——这两件事是配套的，考试特别爱在这个"少一"上做文章。

### 带下标的遍历

有些题目需要"既知道元素，又知道它是第几个"，这时候用下标：

```python
scores = [78, 92, 65]
for i in range(len(scores)):
    print(f"第 {i} 个元素是 {scores[i]}")
```

输出：

```text
第 0 个元素是 78
第 1 个元素是 92
第 2 个元素是 65
```

`len(scores)` 是列表长度（3），`range(3)` 给出 `0, 1, 2`，正好是合法下标。**`for i in range(len(列表))` 这个组合你必须闭着眼睛能写出来**，后面手写排序、手写查找全靠它。

Python 还提供一个更省事的 `enumerate`，一次给你下标和值：

```python
scores = [78, 92, 65]
for i, s in enumerate(scores):
    print(i, s)          # 输出 0 78 / 1 92 / 2 65
```

效果和上面一样，只是少写一个 `scores[i]`。看到别人代码里 `for i, x in enumerate(arr)` 不要慌，就是"带编号的遍历"。

---

## 五、列表 list：数据结构的第一块积木

前面 `scores` 就是列表。你可以把它想成一列**带编号的火车车厢**：每节车厢装一个东西，编号（下标）从 **0** 开始，想拿哪节车厢的东西就报编号。

在 Python 里，列表几乎就是"线性表"这个词的具体化身。第 2 篇我们会认真算它的复杂度，这一篇先把用法摸熟。

### 创建和访问

```python
a = [10, 20, 30, 40]     # 四个元素
b = []                   # 空列表，后面要用 append 往里加东西
c = [0] * 5              # [0, 0, 0, 0, 0]，这个写法在初始化数组时超常用

print(a[0])      # 10      第一个
print(a[3])      # 40      最后一个（长度 4，最大下标 3）
print(a[-1])     # 40      负数表示从后往前数，-1 就是最后一个
print(len(a))    # 4
```

**踩坑第一名**：`a[4]` 会报 `IndexError: list index out of range`。长度 4 的列表，合法下标只有 0、1、2、3。每次写下标前，脑子里过一遍"长度是多少，最大下标是长度减一"。

### 增：append 与 insert

```python
a = [10, 20, 30]
a.append(40)        # 加到末尾 → [10, 20, 30, 40]
a.insert(1, 15)     # 在下标 1 的位置插入 15 → [10, 15, 20, 30, 40]
print(a)
```

输出 `[10, 15, 20, 30, 40]`。

`append` 是**最常用**的列表操作，没有之一。它只在末尾加东西，速度很快。`insert` 插在中间，后面的元素都得往后挪一位——这个"挪"的动作，就是第 2 篇要算的复杂度来源。

### 删：pop 与 remove

```python
a = [10, 20, 30, 40]
x = a.pop()         # 删掉并返回最后一个 → x 是 40，a 变成 [10, 20, 30]
y = a.pop(0)        # 删掉并返回下标 0 的元素 → y 是 10，a 变成 [20, 30]
a.remove(30)        # 按"值"删除第一个 30 → a 变成 [20]
print(x, y, a)
```

输出 `40 10 [20]`。

区别要记牢：**`pop` 按位置删，`remove` 按值删**。`pop()` 不写参数默认删最后一个。

### 改和查

```python
a = [10, 20, 30]
a[1] = 99                 # 直接改：把下标 1 改成 99
print(a)                  # [10, 99, 30]

print(99 in a)            # True，判断"在不在里面"
print(a.index(99))        # 1，返回第一次出现的下标
print(a.count(99))        # 1，数一数出现了几次
```

`in` 这个关键字后面会大量出现，它就是"存在性判断"。注意它一般是从头到尾挨个看一遍，所以对列表用 `in` 是 O(n)——第 2 篇会解释这个记号。

### 切片：一次取一段

```python
a = [0, 1, 2, 3, 4, 5]
print(a[1:4])     # [1, 2, 3]      下标 1 到 3，含头不含尾
print(a[:3])      # [0, 1, 2]      省略开头就是从头开始
print(a[3:])      # [3, 4, 5]      省略结尾就是取到最后
print(a[::-1])    # [5, 4, 3, 2, 1, 0] 步长 -1，把列表倒过来
```

切片和 `range` 一样是**含头不含尾**，`a[1:4]` 取的是下标 1、2、3，不是 1 到 4。这又是一处"少一"，考试读代码题很爱考。

`a[::-1]` 这个"反转列表"的写法值得单独记一下，做回文判断时会想到它。

### 列表里能装任何东西

这一点对后面很重要——列表的元素可以是列表、字典、甚至对象：

```python
matrix = [[1, 2, 3],
          [4, 5, 6]]        # 二维列表，邻接矩阵就用它
print(matrix[1][2])         # 6，先取第 1 行，再取该行第 2 个
print(len(matrix))          # 2，行数
print(len(matrix[0]))       # 3，列数
```

`matrix[1][2]` 是"先坐电梯到 1 楼，再走到 2 号房间"，两个下标从左往右读。图那一篇的邻接矩阵就是这样一个二维列表。

---

## 六、字典 dict：靠名字找东西，不靠编号

列表靠编号找东西，字典靠**键（key）**找东西。生活类比：通讯录。你不会说"我要找第 37 个联系人"，你会说"我要找小明的电话"——"小明"就是键，"电话"就是值。

```python
student = {
    "name": "小明",
    "age": 19,
    "scores": [78, 92, 65]
}

print(student["name"])        # 小明
print(student["scores"][0])   # 78，先取出列表，再取第 0 个
```

字典的写法是 `{键: 值, 键: 值}`。**键必须是不可变的**（字符串、数字、元组都行，列表不行），值随便什么都可以。

### 增、改、查、删

```python
student = {"name": "小明", "age": 19}

student["age"] = 20           # 键已存在 → 修改
student["city"] = "北京"      # 键不存在 → 新增
print(student)                # {'name': '小明', 'age': 20, 'city': '北京'}

print(student.get("phone"))             # None，键不存在时返回 None，不报错
print(student.get("phone", "未填写"))    # 未填写，可以给个默认值
print("name" in student)                # True，判断键在不在

del student["city"]                     # 删掉某个键
print(student)
```

**踩坑第二名**：用 `student["phone"]` 访问一个不存在的键，会直接 `KeyError: 'phone'` 崩掉。所以"不确定键在不在"的时候，用 `.get()` 更安全。这是字典和列表的一个重要区别：列表下标越界报 `IndexError`，字典键不存在报 `KeyError`。

### 遍历字典

```python
student = {"name": "小明", "age": 19, "city": "北京"}

for key in student:                   # 默认遍历的是键
    print(key)

for key, value in student.items():    # 同时拿键和值，最常用
    print(key, "→", value)
```

输出：

```text
name
age
city
name → 小明
age → 19
city → 北京
```

`for k, v in d.items()` 这个写法你会在计数、建表、图的邻接表里反复看到，是必须条件反射写出来的东西。

### 为什么第 7 篇要专门讲字典

现在你只需要知道它"好用"。但注意一个细节：字典查找一个键，**几乎不花时间**，而列表用 `in` 找一个值，要从头看到尾。这个差别就是哈希表的核心，第 7 篇我们会把它拆开讲透。现在先记住结论：**"快速判断某个东西出现过没有"，用字典（或集合 set）比用列表快得多**。

---

## 七、函数 def：把一段逻辑打包

你可以这样想：函数就是一台豆浆机。你把黄豆和水放进去（参数），按一下开关（调用），它给你倒出豆浆（返回值）。你不需要知道里面刀片怎么转的。

```python
def average(nums):
    """求一个列表的平均值。"""      # 三个引号里的是说明文字（文档字符串），可写可不写
    if len(nums) == 0:              # 防一手空列表，不然除以 0 会报错
        return 0
    total = 0
    for x in nums:
        total += x                  # 累加
    return total / len(nums)        # return 把结果交出去，函数到此结束

result = average([78, 92, 65])
print(result)                       # 78.33333333333333
print(f"{result:.2f}")              # 78.33
```

逐块看：

- `def 函数名(参数列表):` 是定义，注意冒号和缩进。
- 参数就是"进货口"，可以有 0 个、1 个或多个，用逗号隔开。
- `return` 把结果**交出去**，并且**立刻结束整个函数**，后面的代码不再执行。
- 没有 `return` 的函数，返回的是 `None`。

**踩坑第三名**：函数里算出了结果，但忘了 `return`，直接 `print` 了一下。这样调用者拿到的是 `None`，后面一用就报错。

```python
def bad_average(nums):
    total = sum(nums)
    print(total / len(nums))   # 只是打印了，没有交出去

x = bad_average([1, 2, 3])     # 屏幕上出现 2.0
print(x)                       # None —— 拿不到值
```

`print` 是"给人看"，`return` 是"给程序用"。考试手写函数题，**没有 `return` 直接扣分**。

### 默认参数

```python
def greet(name, greeting="你好"):
    return f"{greeting}，{name}"

print(greet("小明"))              # 你好，小明
print(greet("小明", "早上好"))     # 早上好，小明
```

有默认值的参数要放在没默认值的参数**后面**。这个规则考试偶尔考填空。

### 一个常见约定：函数不要偷偷改传进来的列表

```python
def add_one(nums):
    result = []                # 新建一个列表
    for x in nums:
        result.append(x + 1)
    return result              # 返回新的，原列表不动

a = [1, 2, 3]
b = add_one(a)
print(a)      # [1, 2, 3]  原列表没变
print(b)      # [2, 3, 4]
```

如果你直接 `nums.append(...)`，就会把调用者的列表改掉。数据结构题里，除非题目明确要求"原地修改"（比如原地反转、原地排序），否则**返回新结构**更安全，也不容易被判错。

---

## 八、类 class：自己造一个数据类型

前面所有东西都是 Python 自带的类型。但数据结构里我们要描述"节点""树""图"这些 Python 没有的东西，就得自己造。造的工具就是 `class`。

你可以这样想：**类是饼干模子，对象是用模子压出来的饼干。** 模子只有一个，饼干可以有很多块，每块饼干上的糖霜（数据）可以不一样。

```python
class Student:
    def __init__(self, name, score):
        self.name = name        # 把传进来的 name 存到"这块饼干"身上
        self.score = score

    def is_pass(self):          # 方法：绑定在这类对象上的函数
        return self.score >= 60

    def __str__(self):          # 决定 print(对象) 时显示什么
        return f"<Student {self.name}: {self.score}>"


s1 = Student("小明", 87)        # 用模子压一块饼干，自动调用 __init__
s2 = Student("小红", 55)

print(s1.name)                  # 小明
print(s1.is_pass())             # True
print(s2.is_pass())             # False
print(s1)                       # <Student 小明: 87>
print(s2)                       # <Student 小红: 55>
```

这段是重点，我们一块一块拆：

**`class Student:`** —— 定义一个类，类名习惯用**大驼峰**（每个单词首字母大写）。

**`def __init__(self, name, score):`** —— 这是"构造方法"，前后各两个下划线。你在写 `Student("小明", 87)` 的时候，Python 会自动调用它。它的工作是**给新对象装数据**。

**`self`** —— 这是最容易懵的地方。`self` 代表"当前这个对象自己"。`self.name = name` 的意思是：把参数 `name` 的值，存到**这个对象**的 `name` 属性上。

> 你可以这样想：`self` 就是"我"。`s1.is_pass()` 里的 `self` 是 `s1`，`s2.is_pass()` 里的 `self` 是 `s2`。同一个方法，被不同的对象调用，操作的就是不同的数据。

**注意这里**：定义方法时 `self` 必须写在参数列表第一个，但**调用时不用传**。`s1.is_pass()` 括号里是空的，Python 自动把 `s1` 塞给 `self`。很多同学第一次写会写成 `s1.is_pass(s1)`，那就错了。

**`__str__`** —— 可选。不写也能跑，只是 `print(s1)` 会输出 `<__main__.Student object at 0x...>` 这种看不懂的地址。写上它，打印才好看。

### 后面会怎么用类

第 4 篇的链表节点，长这样：

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next
```

第 5 篇的树节点：

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

你看，就是同一个套路：`__init__` + `self.xxx = xxx`。现在看不懂 `next=None` 没关系，第 4 篇会讲"为什么节点的下一个还是节点"这种套娃结构。

### 类里面可以装列表和字典

```python
class Stack:
    def __init__(self):
        self.items = []          # 每个栈对象自带一个空列表

    def push(self, x):
        self.items.append(x)

    def pop(self):
        if len(self.items) == 0:
            return None
        return self.items.pop()

    def is_empty(self):
        return len(self.items) == 0


st = Stack()
st.push(1)
st.push(2)
print(st.pop())          # 2
print(st.pop())          # 1
print(st.is_empty())     # True
```

这就是第 3 篇栈的雏形。**类本身不难，难的是"用类来描述一个结构"这个思维方式**。你只要记住：属性（`self.xxx`）存数据，方法（`def xxx(self)`）做操作。

---

## 九、综合实战：一个迷你成绩单

把上面所有东西串起来，写一个完整的小程序。这段代码不长，但用到了变量、列表、字典、循环、判断、函数、类——**如果你能独立默写出来，这一篇就过关了**。

```python
"""学生成绩单：添加学生、统计平均分、找出最高分并打印报告。"""

from dataclasses import dataclass

PASS_SCORE = 60  # 及格线，改这里即可，不用到处找魔法数字


@dataclass(frozen=True)
class Student:
    """一条学生成绩记录。is_pass 由 score 实时派生，避免数据不一致。"""

    name: str
    score: float

    @property
    def is_pass(self) -> str:
        return "及格" if self.score >= PASS_SCORE else "不及格"


class ScoreSheet:
    def __init__(self) -> None:
        self.students: list[Student] = []

    def add_student(self, name: str, score: float) -> None:
        if not name.strip():
            raise ValueError("姓名不能为空")
        if not 0 <= score <= 100:
            raise ValueError(f"分数必须在 0~100 之间，收到：{score}")
        self.students.append(Student(name.strip(), score))

    def counts(self) -> int:
        return len(self.students)

    # 支持 len(sheet) / for stu in sheet，比 counts() 更 Pythonic
    def __len__(self) -> int:
        return len(self.students)

    def __iter__(self):
        return iter(self.students)

    def __str__(self) -> str:
        return f"ScoreSheet({self.students})"

    def average(self) -> float | None:
        """返回平均分（float），没有学生时返回 None。格式化交给展示层。"""
        if not self.students:
            return None
        return sum(stu.score for stu in self.students) / len(self.students)

    def best(self) -> dict | None:
        """返回 {"score": 最高分, "stu": [姓名, ...]}，没有学生时返回 None。"""
        if not self.students:
            return None
        top = max(stu.score for stu in self.students)
        top_stu = [stu.name for stu in self.students if stu.score == top]
        return {"score": top, "stu": top_stu}

    def report(self) -> None:
        # 用 self，而不是全局变量 sheet，多实例时才不会串数据
        average = self.average()
        if average is None:
            print(f"一共有{self.counts()}名学生，尚无平均分")
        else:
            print(f"一共有{self.counts()}名学生，平均分：{average:.1f}")

        for stu in self.students:
            print(f"\t{stu.name}：{stu.score}，{stu.is_pass}")

        best = self.best()  # 只算一次，不再重复遍历
        if best is None:
            print("没有最高分")
        else:
            print(f"最高分是：{best['score']}\t" + "、".join(best["stu"]))

        print("\n")


if __name__ == "__main__":
    sheet = ScoreSheet()
    for name, score in [
        ("小明", 99),
        ("周杰伦", 99),
        ("小红", 98),
        ("小刚", 97),
        ("小亮", 95.5),
        ("小丑", 15.5),
    ]:
        sheet.add_student(name, score)
    sheet.report()

```

运行结果：

```text
一共有6名学生，平均分：84.0
        小明：99，及格
        周杰伦：99，及格
        小红：98，及格
        小刚：97，及格
        小亮：95.5，及格
        小丑：15.5，不及格
最高分是：99    小明、周杰伦
```


---

## 十、零基础最容易踩的 8 个坑

这一节建议你考试前再看一遍。真的，很多分就是这么丢的。

**1. 缩进错乱**

Python 靠缩进划分代码块，缩进错了逻辑就全错，甚至直接 `IndentationError`。统一用 4 个空格，别混用 Tab 和空格。

```python
for i in range(3):
print(i)          # IndentationError: expected an indented block
```

**2. 中文标点**

`：`、`（）`、`，`、`""` 这些中文符号看起来和英文的一模一样，但 Python 不认。报错 `SyntaxError: invalid character` 基本就是它。写完代码切到英文输入法检查一遍。

**3. 少写冒号**

`if x > 0`、`for i in range(3)`、`def f():`、`class A:` 后面都要冒号。忘了就是 `SyntaxError: expected ':'`。

**4. `=` 和 `==` 混用**

`if x = 3:` 是语法错误；`if x == 3:` 才是判断。反向的坑是：想赋值却写成 `==`，程序能跑但结果莫名其妙。

**5. 列表的"赋值"其实是"起外号"**

```python
a = [1, 2, 3]
b = a            # 注意：这不是复制！b 和 a 是同一个列表的两个名字
b.append(4)
print(a)         # [1, 2, 3, 4]   a 也变了

c = a[:]         # 这才叫复制（切片），或者用 a.copy() / list(a)
c.append(5)
print(a)         # [1, 2, 3, 4]   不受影响
```

这个坑在读代码题里杀伤力极大。记住：**`=` 只是让两个名字指向同一块内存，不会复制内容。**

**6. 边遍历边删元素**

```python
a = [1, 2, 2, 3]
for x in a:
    if x == 2:
        a.remove(x)      # 边遍历边删，会漏掉元素
print(a)                 # [1, 2, 3] —— 咦，还有一个 2？
```

原因是删除后后面的元素会往前挪，而循环的下标还在往前走，于是跳过一个。正确做法是倒着删，或者新建一个列表筛选。这个坑第 2 篇讲线性表删除时会再提一次。

**7. `range` 少一 / 下标越界**

`range(1, 5)` 是 1 到 4，只有 4 个数。写双层循环处理 n×n 矩阵时，内层写成 `range(n)` 而不是 `range(n+1)`，多写一个就 `IndexError`。

**8. 函数忘了 return**

上面讲过了，再强调一遍：手写函数题，最后一行的 `return` 就是你的分数。

---

## 十一、典型考题

### 考题 1（读代码写输出）

```python
info = {"name": "小明", "scores": [80, 90, 70]}
total = 0
for s in info["scores"]:
    total += s
info["avg"] = total / len(info["scores"])

print(info["avg"])
print("name" in info, "age" in info)
print(len(info))
```

**参考答案：**

```text
80.0
True False
3
```

**解析（这三行正好踩了三个小坑）：**

第一行：`total = 80 + 90 + 70 = 240`，列表长度 3，`240 / 3 = 80.0`。注意 Python 3 里 `/` 一定得到浮点数，所以输出是 `80.0` 而不是 `80`——写 `80` 要扣分。如果要整除，得写 `//`。

第二行：`"name"` 这个键存在，`"age"` 不存在，所以是 `True False`。`in` 对字典判断的是**键**，不是值，这点别搞混。

第三行：`len(字典)` 数的是**键的个数**。原始字典只有 `"name"`、`"scores"` 两个键，中间 `info["avg"] = ...` 往字典里**新增**了一个键，所以现在是 3 个。**新增键会让长度 +1，修改已有键不会。** 做这类题时，拿笔把字典的键写下来逐个勾，比在脑子里数靠谱得多。

### 考题 2（手写代码）

> 写一个函数 `second_max(nums)`，返回列表中**第二大**的数。假设列表长度至少为 2，且元素互不相同。

**参考答案：**

```python
def second_max(nums):
    first = float("-inf")     # 最大值，先设成负无穷
    second = float("-inf")    # 第二大，也设成负无穷
    for x in nums:
        if x > first:         # 来了个新冠军
            second = first    # 原来的冠军退居第二
            first = x         # 新冠军上位
        elif x > second:      # 没超过冠军，但超过了当前的第二名
            second = x
    return second


print(second_max([3, 1, 4, 1, 5, 9, 2, 6]))   # 6
print(second_max([10, 20]))                   # 10
print(second_max([-5, -3, -1]))               # -3
```

**解析：** 这是"打擂台"的升级版，关键是两个变量要一起更新，而且**顺序不能反**：必须先把 `first` 的旧值交给 `second`，再让 `first` 换人。如果写成 `first = x` 再 `second = first`，两个变量就都变成 `x` 了。

再说初始化。为什么用 `float("-inf")` 而不是 `0`？因为如果列表是 `[-5, -3, -1]`，全是负数，`second` 初始化成 0 就永远更新不了，最后返回错误的 0。用负无穷可以保证任何数都比它大。这是本题最大的坑，也是老师最爱设的边界。

**评分点提示：** 如果时间紧，写 `sorted(nums)[-2]` 也能拿分（时间复杂度 O(n log n)），但老师通常想看的是**一次遍历**的 O(n) 写法。能写循环就写循环，还能顺手拿复杂度分析的分。

---

## 十二、考试怎么考

把这一篇对应到具体题型上：

**选择题**常考：

- 表达式求值：`7 // 2`、`7 % 2`、`2 ** 3`、`"3" + "4"` 分别是多少（分别是 3、1、8、`"34"`）。
- `range(2, 10, 3)` 产生哪些数（2, 5, 8）。
- 列表切片 `a[1:3]` 取几个元素（2 个，含头不含尾）。
- `len()`、`append()`、`pop()` 的返回值分别是什么（`append` 返回 `None`，这点很爱考）。

**填空题**常考：

- 补全循环边界：`for i in range(____)`。
- 补全累加/计数：`total ____ x`（填 `+=`）。
- 补全函数返回值：`return ____`。
- 补全类的构造方法：`def ____(self, ...)`。

**读代码写输出**：这是本课最稳定的题型，一般给 10 到 20 行代码，涉及列表 + 循环 + 判断，让你写出屏幕输出。**方法只有一个：拿笔，一行一行在纸上维护变量的值。** 不要心算，不要跳步，尤其是循环第三轮之后。

**手写代码**：通常是"写一个函数完成 XXX"。评分看三点——**能不能跑（语法）、逻辑对不对（边界）、有没有 `return`**。哪怕写不出最优解，也要先写一个能跑出来的笨办法，有分。

**复杂度分析**：这一篇还没正式讲，但从第 2 篇开始，每段代码你都要能说出它循环了几次。现在先埋个种子：上面 `second_max` 只遍历了一遍列表，长度 n 的列表就转 n 圈，这个"转几圈"就是后面 O(n) 的来源。

---

## 十三、小结

这一篇我们只做了一件事：**把 Python 削到只剩数据结构需要的那部分**。

回过头看，其实就六样东西：

1. **变量**——给数据贴标签，`=` 是赋值不是相等。
2. **输入输出**——`print` 显示，`input` 读进来永远是字符串。
3. **if / elif / else**——让程序分岔，注意冒号和缩进。
4. **for / while / range**——重复，`range` 含头不含尾。
5. **list 和 dict**——两个容器，一个按编号找，一个按键找。
6. **def 和 class**——把逻辑打包，把数据和行为打包。

如果你现在还有点虚，非常正常。**这些语法的真正熟练，是靠后面七篇反复用出来的**，不是靠这一篇背下来的。你只要做到"看到 `for i in range(len(a))` 不害怕，看到 `self.left = left` 知道是在存属性"，就可以往下走了。

最后留一个小练习，不用交，但建议你动手写：用 `class` 写一个 `Queue`，内部用列表，提供 `enqueue(x)`、`dequeue()`、`is_empty()` 三个方法，然后依次放入 1、2、3 再全部取出，打印结果。写出来，第 3 篇你会轻松很多。

---

## 下一篇讲什么

语法有了，接下来要回答一个数据结构里最核心的问题：**同样一个操作，为什么有的写法快，有的写法慢？**

下一篇《复杂度入门 + 线性表》会做两件事：先用大白话把 O(1)、O(n)、O(n²)、O(log n) 讲明白——不讲数学证明，只讲"数循环次数"；然后拿 Python 列表开刀，逐个数清楚 `append`、`insert`、`pop(0)`、`in` 到底各要动几下，为什么"尾部操作快、头部操作慢"。这一篇是后面所有复杂度分析的地基，也是期末考试选择题和简答题的高频区。

---

**第 1 篇完成，回复"继续"我就写第 2 篇。**
