# Python自定义迭代器

在Python 程序中,如果是创建一个自己的迭代器,只需要定义一个实现迭代器协议方法的类即可

```python
class Use:
    def __init__(self, x=2, max=50):
        self.__mul, self.__x = x, x # 初始化元素  __mul =2 __x =2
        self.__max = max # 初始化最大值 __max = 50

    def __iter__(self): #定义迭代器协议方法
        return self # 返回类本身

    def __next__(self):
        if self.__x and self.__x != 1:
            self.__mul *= self.__x
            if self.__mul < self.__max: # 如果小于最大值 则返回mul的值
                return self.__mul
            else: # 不小于则停止迭代 raise 表示迭代
                raise StopIteration
        raise StopIteration


if __name__ == '__main__':
    my = Use()
    for i in my:
        print("迭代的元素是:", i)
```

输出结果如下:



![QdkMPH.png](https://s2.ax1x.com/2019/12/08/QdkMPH.png)







注意:当在Python程序中使用迭代器类时,一定要在某个条件下引发StopIteration错误,这样可以结束遍历循环,否则会产生死循环