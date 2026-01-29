多态的意义：用统一的接口处理不同对象，实现一个接口，多种行为。
实现的核心：**虚函数**；**动态绑定**

## 虚函数

在父类中用`virtual`关键字声明，子类中用`override`关键字重写。

定义父类
```cpp
class Point {...};
// (x,y) point

class Shape {
public:
	Shape();
	void move(const Point&);
	virtual void render();
	virtual void resize();
	virtual ~Shape();
protected:
	Point center;
}
```

加入子类
```cpp
class Ellipse: public Shape {
public:
	Ellipse(float major, float minor);
	virtual void render();   // will define own
protected:
	float major_axis, minor_axis;
};

class Circle: public Ellipse {
public:
	Circle(float radius) : Ellipse(radius, radius) {}
	virtual void render();
};
```

虚函数的使用——两种方式
```cpp
void render(Shape* p) {
	p->render(); // calls correct render function
} // for given Shape!

void func() {
	Ellipse ell(10, 20);
	ell.render(); // static -- Ellipse::render();
	Circle circ(40);
	circ.render(); // static -- Circle::render();
	render(&ell); // dynamic -- Ellipse::render();
	render(&circ); // dynamic -- Circle::render();
}
```


- 如果析构函数可能被继承，一定要设为虚函数

将子类对象直接赋值给父类对象，如`elly = circ;`（`Ellipse elly; Circle circ;`）会发生切片问题。即只有`Ellipse`部分的成员会被复制，`Circle`特有的成员会被切掉；而且调用`elly.render()`时，会执行`Ellipse::render()`，而不是`Circle::render()`。
所以说不应用子类对象给父类对象赋值。一般用父类指针/引用指向子类对象。

