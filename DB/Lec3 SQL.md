# SQL 语言数据类型

SQL 标准支持很多内建类型，包括：

- `char(n)`：定长字符串，由用户指定长度 `n`
    - 当比较两个 `char` 类型的值时，如果它们的长度不同，那么会给短的那个加上额外的空格，使得它们一样长
- `varchar(n)`：变长字符串，由用户指定最大长度 `n`
    - 当比较 `char` 类型和 `varchar` 类型时，有可能会在 `varchar` 值上添加额外的空格，但也有可能不会，这取决于系统。因此即使用 `char` 和 `varchar` 表示两个相同的字符串，比较结果也有可能是 `false`。建议一直使用相同类型的值比较
    - SQL 还提供了 `nvarchar` 类型以表示 Unicode 编码的多语言数据，但是很多数据库支持用 `varchar` 表示 Unicode 编码（尤其是 UTF-8）的字符
- `int`：整数（实际上是依赖于机器的整数的有限子集）
- `smallint`：较小的整数（实际上是依赖于机器的整数的有限子集）
- `numeric(p, d)`：定点 (fixed point) 数，由用户指定位数 `p`（包括符号位）以及十进制小数点右侧的位数 `d`
- `real` / `double precision`：分别对应单精度浮点数和双精度浮点数，其精度依赖于机器
- `float(n)`：浮点数，由用户指定最低精度位数 `n`
- `Null`：适用于所有数据类型。但可以在声明属性时禁用 Null 值
- `date`：日期，包含年（4 位数字）月日，比如 `2025-02-25`
- `time`：时间，包含时分秒，比如 `10:47:20`、`10:47:20.75`
- `timestamp`：时间戳，即日期 + 时间，比如 `2025-02-25 10:47:20.75`
    - 在 SQL Server 2000 里，这个类型被称为 `datetime`

## 完整性限制

primary key 必须非空
foreign key 应当指向外部表的 primary key
可设置部分值为 not null 或 default xxx
当 foreign key 指向的数据被删除，处理方式有：on delete cascade | set null | restrict | no action | set default

# SQL 语句

典型的 SQL 查询语句

```sql
SELECT A1, A2, ..., An []
FROM r1, r2, ..., rm []
WHERE P;
```

- SQL 语句中，`%`代表匹配任意子字符串，`_`代表匹配任意字符，如果想表示这个字符本身，需要打转义符
更完整的：
```sql
SELECT class, COUNT(*) 
FROM student 
WHERE age >= 18 -- 先过滤：只看成年学生 
GROUP BY class -- 按班级分组 
HAVING COUNT(*) > 30; -- 后过滤：只保留人数>30的班级
```