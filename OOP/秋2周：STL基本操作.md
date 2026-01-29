## 1. 字符串基本操作

合并、查找与删除
```C++
int main(){
    // 字符串的合并
    string str = "Hello, Hangzhou!";
    string str2("Who are you");
    string str3 = str+str2;
    cout << str3 << endl;

    // 子串的查找
    string sub_str = "Hangzhou";
    int pos = str.find(sub_str);
    cout << pos << endl;
  
    // 字符串的替换
    string rep_str = "Beijing";
    cout << str.replace(pos, rep_str.length(), rep_str) << endl;
    return 0;
}
```

## 2. fstream 输入输出

```C++
int main(){
    string str = "Hello, Hangzhou!";
    ofstream ofstr("text.txt");
    ofstr << str << endl;

    ifstream ifstr("text.txt");
    string input_str;
    ifstr >> input_str;      // 以空格为单位分割输入
    cout << input_str << endl;
    ifstr >> input_str;
    cout << input_str << endl;
}
```


## 3. Vector


## 4. List
![[List结构体基本操作.png]]