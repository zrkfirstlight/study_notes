# 书籍信息
书名：《编写高质量python代码的59个有效方法》  
作者： Brett Slatkin  
评论： 我买的版本是16年的，书中主要涉及py2和py3.4，可以说我买到的那一刻这本书的内容就严重过时了，幸好此书主要讲述pythonic，这正是我所欠缺的，因此我一边看，一边应用，一边批判，最终提升我对python代码码风的理解。  

# pythonic （代码风格）


## 10. 利用enumerate 代替 range
这是人尽皆知的技巧，一种在遍历迭代器的同时，快捷的获取内容的索引

## 11. 用zip遍历两个迭代器
用于平行的遍历多个迭代器，同时不需要担心索引溢出。

## 12. 不使用 for - else/ while - else
这两种语法结构的特点在于，如果循环过程中没有被break所终止，那么会额外运行else中的内容，相反，如果循环过程中被break打断，那么else中的语句就不会被执行。

一般用于判断某个范围中的数据，是否都**不满足**某个条件，但是完全可以使用辅助函数来达到同样的效果，并且辅助函数更易读，示例如下

    def func(*args):
        for i in range(10):
            if ...:
            return false
        return ture

## *13.try-except-else-finally语法应对不同场景*
主要用于错误处理，根据不同场景选择不同结构使用，个人一般try/except就够用了

### 1. try-finally

无论try是否会报错，finally都会被执行，一般是用于进行数据清理。

    handle = open("data.txt")
    try:
        data =  handle.read()
    finally:
        handle.close()

若文件数据无法被打开，那么try中会报错，此时文件句柄已经被锁定，因此需要被释放。至于try中的报错，则是掷出由更高级的代码来处理。所以，外面至少应当在嵌套一层才能保证代码的整体稳定性。正如下面的代码所写。

    try:
        handle = open("data.txt")   # 文件打开错误
        try:
            data =  handle.read()   # 数据读取错误
        finally:
            handle.close()          # 数据读取错误时保证句柄关掉，若文件打开错误，则不会被执行
    except Error as e:
        raise

### 2.else
else的作用是当try语句块不报错的时候执行，我有时在想，这是在干什么，其实这是一种错误处理的思路。分析下面的代码，json.loads(data)的时候可能会报错 ValueError，这个时候，我掷出KeyError。如果不报错，我读取文件中的数据，然后将其返回。

为什么掷出KeyError，这是因为result_dict[key]可能报错KeyError，因此在这一步进行错误类型的隐瞒，但是通过 KeyError from e又保留了报错的具体信息。

else意义，try和else分开的意义，这段代码只能处理ValueError，因此其他的错误需要掷出，但是else中语句不会出现ValueError错误，因此认为它是安全的不需要考虑的，仅考虑可能出现ValueError的try代码。

这里的思想就是，代码的错误传播更加的清晰，我知道那些代码可能会出现什么样子的错误。一般大团队用来维护大项目的时候，这种处理方法会很好用，如果是个人的话，那就算了吧。


    def load_json_key(data, key):
        try:
            result_dict = json.loads(data)
        except ValueError as e:
            raise KeyError from e
        else:
            return result_dict[key] 

### 3.try-except-else-finally

这四部分代码各司其职，相互配合，是一个非常有用的写法。

try/except代码块，写那些可能报错的代码，并给出对应的处理方式。else语句一方面是分割风险代码和正常代码，一方面是可以将错误信息正常掷出，一方面是可以在清理工作前执行额外工作。finally则是进行清理工作，如果进入了try，那么一定会执行一些清理工作，因此肯会导致没有清理工作的代码应当是在try代码段前面的。

    def divide_json(path):
        handle = open(path, 'r+')
        try:
            data = handle.read()
            json_data = json.loads(data)
            value = json_data['value1'] / json_data['value2']   # 除0错误
        except ZeroDivisionError as e:
            return UNDEFINED
        else:
            json_data[answer] = value
            result = json.dumps(json_data)
            handle.seek(0)
            handle.write(result)
            return value
        finally:
            handle.close()

在这段代码里面，即使else返回了值，但是finally的语句还是会在返回值之前执行，只有清理工作做完了之后，才会真的返回函数值。

# 函数
## 14. 特殊情况用异常来表示，而不是用None来表示
在两数相除的场景下，除0 是不被允许的，例如如果我们这样写，显然是看不出来错误的，但是我们在调用的时候会出错

    def divide(a, b):
        try:
            return a / b
        except ZeroDivisionError；
            return None

我们很可能会这样处理，看起来是正确的，但是因为python中0, false, None在条件判断中是视同的，因此分子为0时，输入没有错误，但是我们却认定输入错误了，显然这样就会产生无形的错误，这次可以看出，下次呢？

    result = divide(5, 0)
    if result is None:
        print("输入错误“)

解决方法就是在函数中，遇到特殊情况，直接掷出错误。然后我们在调用函数的时候，遇到了掷出的错误，然后就去处理它就好了。

    def divide(a, b):
        try:
            return a / b
        except ZeroDivisionError as e；
            return ValueError("invalid inputs") from e

    try:
        result = divide(5, 0)
    except ValueError：
        print("invalid inputs")
    else:
        print(f"result is {result}")

## 15.在闭包中使用外围作用域的变量
### 0.闭包的概念和基本用途
闭包是一个比较综合的概念，有一个定义是：函数内部的嵌套函数可以在函数结束后仍然使用函数局部变量中的函数。如下，嵌套函数counter在外部函数make_counter被执行完后，还是可以使用外部作用域中的count变量。

    def make_counter():
        count = 0  # 外部函数的局部变量，被闭包“记住”
        
        def counter():
            nonlocal count  # 声明修改外部变量
            count += 1
            return count
        
        return counter

    counter1 = make_counter()
    print(counter1())  # 1
    print(counter1())  # 2

    counter2 = make_counter()  # 新的闭包，拥有独立的count
    print(counter2())  # 1

一般来讲，闭包的作用是将外部函数中的信息进行隐藏，实现了封装；保存外部函数中的信息，使其可以持续存在。  
在封装的效果上，和类的封装是差不多的，在简单的场景下，会使用闭包来做。  
还有一种用法就是装饰器，在不修改原代码的基础上，动态的为函数添加新的功能，但终究不是纯粹的闭包了。

### 1.书中的例子
书中并没有强调闭包本身，而是在强调使用闭包时，对作用域的控制，扩展一下，就是使用闭包时，怎么避免或者利用不同作用域之间的污染。   
下面一段代码，实现对数据进行排序，检查是否有重点对象并返回标志，重点对象的优先级最高。

    def sort_priority(numbers, group):
        found = False
        def helper(x):
            if x in group:
                found = True
                return (0, x)
            return (1, x)
        numbers.sort(key=helper)
        return found

    numbers = [2,5,9,6,3,4,7,8,5,1]
    found = sort_priority(numbers, {4,6,2,9})
    print(f"found", found)
    print(numbers)
    # found False
    # [2, 4, 6, 9, 1, 3, 5, 5, 7, 8] 

可以看到，函数的返回值不是预料值，这是因为尾部函数和内部函数的作用域没有打通，要想打通，那就是用nonlocal关键字，如下： 

    def sort_priority(numbers, group):
        found = False
        def helper(x):
            nonlocal found
            if x in group:
                found = True
                return (0, x)
            return (1, x)
        numbers.sort(key=helper)
        return found

    numbers = [2,5,9,6,3,4,7,8,5,1]
    found = sort_priority(numbers, {4,6,2,9})
    print(f"found", found)
    print(numbers)
    # found True
    # [2, 4, 6, 9, 1, 3, 5, 5, 7, 8]

nonlocal和global有些相似，都是将不同作用域进行关联，但是nonlocal只能打通一层作用域，而global时全局作用域，实际上，使用nonlocal不如直接声明类。

    def sort_priority():
        found = False
        def helper():
            nonlocal found
            found = True

        helper()
        return found

    found = False
    found_local = sort_priority()
    print(f"local found", found_local)
    print(f"global found", found)
    # local found True
    # global found False

将内部作用域与外部作用域打通了，对全局作用域不起作用。

    def sort_priority():
        found = False
        def helper():
            global found
            found = True

        helper()
        return found

    found = False
    found_local = sort_priority()
    print(f"local found", found_local)
    print(f"global found", found)
    # local found True
    # global found False

将内部作用域与全局作用域打通了，对外部作用域不起作用。

    def sort_priority():
        nonlocal found 
        found = True
        return found

    found = False
    found_local = sort_priority()
    print(f"local found", found_local)
    print(f"global found", found)

    #     nonlocal found
    #     ^^^^^^^^^^^^^^
    # SyntaxError: no binding for nonlocal 'found' found

仅有函数作用域和全局作用域，因此nonlocal不能找到外部作用域而失效。


## 16.使用生成器来改写直接返回列表的函数
只能说根据需要来吧，这几种写法一般是没有啥问题的，生成器是惰性的，也就是他不会瞬间释放大量的内存，但是如果重复索引就不如列表灵活了。如果仅仅是单纯获取一次数据，类似生产者和消费者那样，在不考虑内存的情况下，列表是微微快于生成器的。

读取一段字符串中的所有单词的首字母，并返回其索引位置。需要函数返回一系列结果，可以使用列表直接返回结果，也可以直接返回一个生成器（迭代器）。

    def index_words(text):
        result = []
        if text:
            result.append(0)
        for index, letter in enumerate(text):
            if letter == ' ':
                result.append(index + 1)
        return result

    address = "long long years ago"
    result = index_words(address)
    print(result)

这是一个很常见的使用列表返回结果的写法。

    def index_words(text):
        yield 0
        for index, letter in enumerate(text):
            if letter == ' ':
                yield index + 1

    address = "long long years ago"
    result = list(index_words(address))
    print(result)

这是一个很常见的使用生成器返回结果的写法。

    from itertools import islice

    def index_file(f):
        result = 0
        for line in f:
            if line:
                yield result
            for letter in line:
                result += 1
                if letter == ' ':
                    yield result

    with open("data.txt", 'r') as f:
        swap = index_file(f)
        result = islice(swap, 0, 4)
        print(list(result))
        print(list(swap))

这是利用生成器来获取一个文件中，所有单词首字母索引的常见写法

