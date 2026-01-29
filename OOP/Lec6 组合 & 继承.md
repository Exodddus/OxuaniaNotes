## 组合 (Composition)

**has-a**逻辑，新类由多个已有类的对象 “组装” 而成的。
```cpp
class HealthPlan {…}; 
class SalaryHistory {…};

class Employee {
private:
    std::string name;          // 有一个姓名（string对象）
    std::vector<SalaryHistory> salaryHistories; // 有一个薪资历史列表（vector对象）
    std::string address;       // 有一个地址（string对象）
    HealthPlan healthPlan;     // 有一个健康计划（HealthPlan对象）
    Employee* supervisor;      // 有一个上级（Employee指针，可共享）
};
```

组合的两种包含逻辑：
1. **完全包含**：新对象创建时一同创建，新对象销毁时一起销毁，如`name`，`healthplan`。
2. **引用 / 指针包含**：指针指向对象销毁与否，与包含这个指针的对象没有关系，如`supervisor`。

所有嵌入的成员对象必须初始化，采用初始化列表是较好的方法

```cpp
SavingsAccount::SavingsAccount(const char* name, const char* address, int cents) 
    : m_saver(name, address),  // 直接调用m_saver的带参构造函数
      m_balance(0, cents)      // 直接调用m_balance的带参构造函数
{}
```

成员对象一般推荐设为private，这体现了**封装**的思想。

## 继承 (Inheritance)

**is-a** 逻辑，新类（子类 / 派生类）是原有类（父类 / 基类）的特殊版本
e.g. `CD`类和`DVD`类存在属性和方法的冗余，那么可以抽取这些相同的属性和方法，建立一个父类`Item`，然后让这两类作为子类继承。
```cpp
// 父类：不同子类的共同特征
class Item {
public:
    void setComment(const std::string& com) { comment = com; }
    bool getOwn() const { return gotIt; }
    virtual void print() const { /* 打印共同属性 */ }
protected:
    std::string title;
    int playingTime;
    bool gotIt;
    std::string comment;
};

// 子类CD：继承Item，添加自己的属性
class CD : public Item {
public:
    std::string getArtist() const { return artist; }
    void print() const override { /* 先调用父类print，再打印artist和曲目数 */ }
private:
    std::string artist;
    int numberOfTracks;
};

// 子类DVD：继承Item，添加自己的属性
class DVD : public Item {
public:
    std::string getDirector() const { return director; }
    void print() const override { /* 先调用父类print，再打印director */ }
private:
    std::string director;
};
```
子类在构造时，需要先初始化父类部分；在析构时，则应当先销毁子类，再销毁父类。

继承的**访问控制**：

| 父类成员权限    | 继承方式：public（公有继承） | 继承方式：protected（保护继承） | 继承方式：private（私有继承） |
| --------- | ----------------- | -------------------- | ------------------ |
| public    | 子类中仍为 public      | 子类中变为 protected      | 子类中变为 private      |
| protected | 子类中仍为 protected   | 子类中仍为 protected      | 子类中变为 private      |
| private   | 子类中不可访问（隐藏）       | 子类中不可访问（隐藏）          | 子类中不可访问（隐藏）        |
|           |                   |                      |                    |

**向上转型**（Up-casting)：

把子类指针 / 引用转为父类指针 / 引用，实际上就是直接把子类当做父类使用。在这个过程中会丢失子类的特有信息，但这样做可以统一接口，方便扩展复用。

```cpp
Manager pete("Pete", "444-55-6666", "Bakery"); // 子类对象
Employee* ep = &pete; // 向上转型：子类指针→父类指针
Employee& er = pete;  // 向上转型：子类引用→父类引用
```


## 友元函数

不是类的成员函数，但能直接访问类的所有成员。需要在类内用`friend`关键字声明才能使用，是单向的授权，不影响类的封装。

e.g. 复数类的加法
1. 类内声明友元函数
```cpp
class Complex {
private:
    double real;   // 实部（私有成员）
    double imag;   // 虚部（私有成员）
public:
    // 构造函数：初始化实部和虚部
    Complex(double r = 0.0, double i = 0.0) : real(r), imag(i) {}

    // 声明友元函数：告诉编译器这个外部函数可以访问我的私有成员
    friend Complex addComplex(Complex a, Complex b); 

    // 也可以声明运算符重载为友元（更常用）
    friend Complex operator+(Complex a, Complex b);
};
```

2. 类外实现友元函数
```cpp
// 实现友元函数 addComplex：计算两个复数的和
Complex addComplex(Complex a, Complex b) {
    // 直接访问 a 和 b 的 private 成员 real、imag
    return Complex(a.real + b.real, a.imag + b.imag);
}

// 实现友元运算符重载 +：更直观的用法（a + b）
Complex operator+(Complex a, Complex b) {
    return Complex(a.real + b.real, a.imag + b.imag);
}
```

3. 友元函数的调用
```cpp
Complex a(1.5, 2.5);  // a = 1.5 + 2.5i
Complex b(3.0, 4.0);  // b = 3.0 + 4.0i

// 调用友元函数 addComplex
Complex c1 = addComplex(a, b);  // c1 = 4.5 + 6.5i

// 调用友元运算符 +（更自然）
Complex c2 = a + b;             // c2 = 4.5 + 6.5i
```

友元关系是单向的，不具备传递性，也不能继承给子类。