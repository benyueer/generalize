# 解释器相关
1. 对象缓存机制
   又称`对象缓冲池`
   对象有三要素
   - id: 标识
   - type：类型
   - value：内存值
   
   ```py
    a, b, c, d = 1, 1, 1000, 1000
    print(a is b, c is d)

    def foo():
        e = 1000
        f = 1000
        print(e is f, e is d)
        g = 1
        print(g is a)

    foo()
    输出是：
    True False
    True False
    True
   ```
   CPython解释器出于性能优化的考虑，把频繁使用的整数对象用一个叫`small_ints`的对象池缓存起来造成的。`small_ints`缓存的整数值被设定为`[-5, 256]`这个区间，也就是说，在任何引用这些整数的地方，都不需要重新创建int对象，而是直接引用缓存池中的对象。如果整数不在该范围内，那么即便两个整数的值相同，它们也是不同的对象。
   CPython底层为了进一步提升性能还做了另一个设定，对于同一个代码块中值不在`small_ints`缓存范围内的整数，如果同一个代码块中已经存在一个值与其相同的整数对象，那么就直接引用该对象，否则创建新的int对象。
  
2. Python 的内存管理
   不同解释器有所区别
   内存空间的分配与释放都是由Python解释器在运行时自动进行的
   CPython：引用计数、标记清理、分代收集。

# 语法与特性
1. lambda 函数
   `Lambda`函数也叫匿名函数，它是功能简单用一行代码就能实现的小型函数。Python中的`Lambda`函数只能写一个表达式，这个表达式的执行结果就是函数的返回值，不用写`return`关键字。`Lambda`函数因为没有名字，所以也不会跟其他函数发生命名冲突的问题。
   ```py
    items = [12, 5, 7, 10, 8, 19]
    items = list(map(lambda x: x ** 2, filter(lambda x: x % 2, items)))
    print(items)    # [25, 49, 361]
   ```
2. 迭代器和生成器
   `__next__`和`__iter__`这两个魔法方法就代表了迭代器协议。可以通过`for-in`循环从迭代器对象中取出值，也可以使用`next`函数取出迭代器对象中的下一个值。生成器是迭代器的语法升级版本，可以用更为简单的代码来实现一个迭代器。
   ```py
   # 迭代器的Fib
   class Fib(object):
    
    def __init__(self, num):
        self.num = num
        self.a, self.b = 0, 1
        self.idx = 0
   
    def __iter__(self):
        return self

    def __next__(self):
        if self.idx < self.num:
            self.a, self.b = self.b, self.a + self.b
            self.idx += 1
            return self.a
        raise StopIteration()

   # 生成器的Fib
   def fib(num):
    a, b = 0, 1
    for _ in range(num):
        a, b = b, a + b
        yield a
   ```

3. 闭包
   py中的闭包与js很相似，但稍有区别
   1. js中有作用域链
   2. js可以直接访问外部变量，py需要声明`nonlocal`
   ```py
   def multiply():
      return [lambda x: i * x for i in range(4)]

   print([m(100) for m in multiply()])
   ```
   对于上面这段代码，一般可能会给出错误答案`[0, 100, 200, 300]`，这是因为在`multiply`中`i`的生命周期被延长了，那么`i`最后的值是`3`，所以到最后执行时取到的值就是`3`，那么结果就是`[300, 300, 300, 300]`
   怎么解决呢？
   3. 使用生成式
      ```py
      def multiply():
          return (lambda x: i * x for i in range(4))

      print([m(100) for m in multiply()])

      def multiply():
        for i in range(4):
            yield lambda x: x * i
      ```
   4. 使用偏函数
      ```py
      from functools import partial
      from operator import __mul__

      def multiply():
          return [partial(__mul__, i) for i in range(4)]

      print([m(100) for m in multiply()])
      ```
4. 生成式 推导式
   推导式有三种：
      1. 列表
         ```py
         # 生成一个包含 1 到 10 的平方的列表
         squares = [x**2 for x in range(1, 11)]
         ```
      2. 集合
         ```py
         # 生成一个包含 1 到 10 的平方的集合
         squares_set = {x**2 for x in range(1, 11)}
         ```
      3. 字典
         ```py
         # 生成一个包含 1 到 10 的平方的字典，键为数字，值为平方
         squares_dict = {x: x**2 for x in range(1, 11)}
         ```

5. 生成器表达式
   ```py
    generator = (x for x in range(1, 4))

    print(next(generator))  # 输出: 1
    print(next(generator))  # 输出: 2
    print(next(generator))  # 输出: 3
   ```
   