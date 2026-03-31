## Sparse checkout

可以只克隆一个仓库中的部分文件夹，命令为：
```bash
git clone --depth 1 --filter=blob:none --sparse 你的仓库地址 
cd 项目名 
git sparse-checkout set src/ docs/ # 直接指定要的文件夹
```

后续添加 / 删除要下载的文件夹：
```bash
# 添加新文件夹 
git sparse-checkout add utils/ 
# 取消某个文件夹（本地会删除，远程不受影响） 
git sparse-checkout remove docs/ 
# 查看当前配置 
git sparse-checkout list
```