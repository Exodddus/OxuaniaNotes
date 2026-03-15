# 关系代数

## 基本运算
- select: 从关系中筛选出满足条件的元组（行）
	- $\sigma_{\text{Dept} = \text{"计算机系"}}(\text{Student})$
- project: 从关系中选取指定的属性（列），并自动去重。
	- $\Pi_{\text{Name}, \text{Age}}(\text{Student})$ - 取出学生表 `Student` 中的姓名和年龄列。
- union: 将两个兼容且结构完全相同的关系合并，去除重复元组。
	- $\text{Course\_A} \cup \text{Course\_B}$ - 合并 `Course_A` 和 `Course_B` 两门课的选课学生名单。
- set difference: 
	- $\text{Course\_A} - \text{Course\_B}$ - 找出选了课 A 但没选课 B 的学生。
- cartesian product: 将两个关系的元组所有可能组合连接起来，属性列数相乘。
	- $r×s={{t,q} ∣ t∈r\,and\,q∈s}r×s={{t,q} ∣ t∈r\,and\,q∈s}$
- rename: 给关系或属性重新命名，用于结构运算或简化表达。
	- $\rho_{S}(\text{Student})$ - 将关系 `Student` 重命名为 `S`。

## 附加运算
- intersection
- natural join
- assignment
- outer join
- division

## 扩展运算
- Generalized Projection
- Aggregated Functions