
**Global vars** 全局变量在函数外定义，可以被不同的.cpp文件共享。
**extern** 关键字可以在定义某个变量/函数的程序之外调用这个变量/函数。
**static** 关键字禁止文件外对该变量/函数的访问。
- 对于局部变量，**static** 关键字表示只初始化一次，同时不同调用间保留变量值

## 访问控制

**封装**：通过访问说明符控制成员的访问权限。
**public**: 能被任何对象访问
**protected**：只能在类内部以及派生类内访问
**private**：只能在类内部被访问。一般将所有的成员变量都声明为`private` ，这样外部用户只能通过调用方法来访问/操控对象的成员，方便管理。

## 构造与析构

### 构造函数

名称与类名相同，无需指出返回类型。
1. 参数化构造函数
```C++
zjuID::zjuID(std::string name, int idNumber) {
    this->name = name;
    if (idNumber > 0)
        this->idNumber = idNumber;
}
```
  - 成员初始化列表
```C++
zjuID::zjuID(std::string n, int id): name(n), idNumber(id) { }
```
这种方法相比于在函数体内为变量赋值效率更高。
2. 默认构造函数：构造时没有参数

### 析构函数
在对象销毁前被调用，函数名 = ~+类名，无需参数，没有返回类型。如果在对象内使用 new 关键字分配数据，就一定要在该对象的析构函数里用 delete 释放内存空间。

## Getter & Setter

- getter：用于读取成员变量的函数
```C++
std::string zjuID::getName() {
    return this->name;
}

int zjuID::getIdNumber() {
    return this->idNumber;
}
```
- setter：用于修改成员变量的函数
```C++
void zjuID::setName(std::string name) {
    this->name = name;
}

void zjuID::setIdNumber(int idNumber) {
    if (idNumber >= 0)
        this->idNumber = idNumber;
}
```

## Friends 友元

```C++
class Engine;

class Car {
private:
    int speed;

public:
    Car() : speed(200) {}
    friend class Engine;
};

class Engine {
public:
    void boost(Car& c) {
        c.speed += 100;  
    }
};
```
以上这段代码将Engine类设为Car类的友元，因此Engine类中的方法可以直接调用Car类的私有的或受保护的成员。
友元类不具备对称性、传递性和继承性。

# 作用域和生命周期

## 局部对象

1. 字段 (fields)
	- 在类内定义，生命周期等于对象的生命周期
	- 作用域为整个类，可以被其中的构造函数或方法使用
2. 形参 (parameters)
	- 定义在构造函数或方法的签名上，接受来自外部的值，即为形参初始化的过程。
	- 生命周期仅在构造函数或方法的调用中，调用结束后消失。作用域限制在定义它们的构造函数内。
3. 局部变量 (local variables)
	- 生命周期和定义域限制在构造函数或方法的调用内

### 全局对象

### 静态对象

特点：静态存储，受限访问
1. 全局变量：作用域限制在当前文件
2. 局部变量：生命周期存在于整个程序的运行期间
3. 成员变量：成员变量属于类本身，而不是某个特定的对象；在类的内部只能对其进行声明而不能定义。
4. 成员函数：此时成员函数也属于类本身，而不是某个特定的对象


# 常量 Constants

1. 修饰变量
```C++
    const int x = y;
```
   之后x就不能被修改了

2. 修饰指针
- `const int* p1` 或 `int const* p1`: const修饰一个**值**，p1指向的值不得被修改，但p1本身可以被修改
- `int* const p2`：修饰**指针**，所以p2本身不得被修改，但可以修改其指向的值。
- `const int* const p3`：指针和指向的值都不变。

