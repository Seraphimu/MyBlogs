## 函数模板

模板就是基于用户制定的规则, 让编译器写代码. 在其他语言里这玩意叫泛型.

### 入门示例

```cpp
template<typename T>
void Swap(T & left, T & right) {
        T temp = left;
        left = right;
        right = temp;
}

int main(void) {

        int a = 5;
        int b = 6;
        Swap(a, b);

        double c = 7;
        double d = 8;
        Swap(c, d);

        return 0;
}
```

使用GDB调试结果如下:

```cpp
调试结果
Swap<double> (left=@0x7fffffffd9d0: 7, right=@0x7fffffffd9c8: 8) at template.cpp:4
4               T temp = left;

Swap<int> (left=@0x7fffffffd9dc: 5, right=@0x7fffffffd9d8: 6) at template.cpp:4
4               T temp = left;
```

<!-- 这里和STL有些相似, STL也会用vector<type>  -->

### 坑爹示例

```cpp
//隐形实例化与显形实例化
template<class T>
T Add(const T & left, const T & right) {
        return left + right;
}

int main(void) {
        int a = 10;
        int b = 20;
		//正常通过
        Add(a, b);

        int c = 30;
        double d = 114.5;
		//报错
		//deduced conflicting types for parameter ‘const T’ (‘int’ and ‘double’) 类型冲突了
        Add(c, d);

        return 0;
}
```

对于这种类型错误的解决, 有两种解决手段:

1. 用户强制类型转换:

```cpp
Add(c, (int)d);
```

2. 通过显式实例化:

```cpp
Add<int> (c, d);
```

### 有同名的函数, 一个模板一个非模板

```cpp
template<typename T>
//template<class T>
T Add(const T & left, const T & right){

        return left + right;
}

int Add(const int & left, const int & right) {

        return left + right;
}

int main(void) {
        Add(1, 2);
        Add(3, 5.5);
        return 0;
}
```

注意: 

1. 函数模板与非函数模板类型完全匹配，不需要函数模板实例化如上文的: `Add(1, 2)`
2. 函数模板与非函数模板类型不完全匹配, 编译器会根据实参选择匹配的模板;
3. 函数模板的参数不允许隐式类型转换, 普通函数允许

## 类模板

实例:

```cpp
template <class T, int N>

class Vector {
private:
        T array[N];
public:
        int getArraySize() const;
};

template <class T, int Z>
//注意这一步, 在类外定义函数时要声明类模板
int Vector<T, Z>::getArraySize() const{return Z;}

int main(void) {
        Vector<int, 11> v;
        int size = v.getArraySize();

        return 0;
}
```

实例: MyTinySTL

这下这段代码就容易理解得多了, 这里定义了一个通用的数据类型模板 `T`, 而基于 `T` 又创建了变量 `v`.

```cpp
template <class T, T v>
struct m_integral_constant
{
  static constexpr T value = v;
};
```


问题路径: 

```
Thread Pool ==> Template ==> MySTL ==> Thread Pool
```