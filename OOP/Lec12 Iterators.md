# 迭代器基础

- STL的设计思想之一是“算法”和“容器”的相互分离。迭代器的核心价值在于，提供统一的接口，屏蔽不同容器的底层差异，让同一个算法能无缝适配所有容器。
- 基础操作
	- `++` 移动到下一个元素。
	- `*` 解引用，获取当前指向的元素。
	- `!=`/`==` 判断两个迭代器是否指向同一个位置。
	- `->` （部分支持）访问元素的成员。
	
```cpp
// 通用find算法：适配所有支持InputIterator的容器
template <class InputIterator, class T>
InputIterator find(InputIterator first, InputIterator last, const T &value) {
    while (first != last && *first != value) {
        ++first; // 统一的移动操作
    }
    return first; // 返回指向目标元素的迭代器（未找到返回last）
}
```

* 自定义迭代器：以`List`容器为例

```cpp
// 定义容器 List 和节点 ListItem
template<class T>
class ListItem { // 链表节点
public:
    T& val() { return _value; } // 获取节点值
    ListItem* next() { return _next; } // 获取下一个节点
private:
    T _value; // 节点存储的数据
    ListItem<T>* _next; // 指向 next 节点的指针
};

template<class T>
class List { // 链表容器
public:
    ListIter<T> begin() { return ListIter<T>(front); } // 迭代器指向第一个节点
    ListIter<T> end() { return ListIter<T>(nullptr); } // 迭代器指向nullptr（哨兵）
private:
    ListItem<T>* front; // 链表头节点
    ListItem<T>* end;   // 链表尾节点
};
```

```cpp
// 定义迭代器 ListIter
template<class T>
class ListIter {
private:
    ListItem<T>* ptr; // 指向链表节点的指针（迭代器的核心数据）
public:
    ListIter(ListItem<T>* p = 0) : ptr(p) {} // 构造函数

    // 重载++：移动到下一个节点
    ListIter<T>& operator++() {
        ptr = ptr->next();
        return *this;
    }

    // 重载==：判断两个迭代器是否指向同一个节点
    bool operator==(const ListIter& i) const {
        return ptr == i.ptr;
    }

    // 重载!=：判断是否不相等
    bool operator!=(const ListIter& i) const {
        return !(ptr == i.ptr);
    }

    // 重载*：解引用获取节点值
    T& operator*() {
        return ptr->val();
    }

    // 重载->：访问节点的成员（如节点是对象时）
    T* operator->() {
        return &(**this); // 等价于 &(ptr->val())
    }
};
```

# 迭代器的关联类型

> Q: 如何获取迭代器指向的元素类型？

