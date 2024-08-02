## 提交本地文件夹到github

1. Github上新建仓库 ，==不要==勾选创建readme.md文件，否则会创建一个 `main` 主分支

2. 进入 **Git bash** 提交**[Typora文件]**文件夹下的所有内容

   ```bash
   cd F:/Typora文件
   
   git init  # 初始化git
   
   git add .  # . 代表添加当前目录下所有文件
   
   git commit -m '***' # *** 为提交描述
   
   # 远程连接仓库 http 地址
   git remote add origin https://github.com/17351590701/typora_note.git  
   
   git push -u origin master # 提交到master分支（主分支）
   ```

3. 后续提交

   ```bash
   cd F:/Typora
   
   git add . # 将所有修改文件提交到工作区
   
   git commit -m '...'
   
   git push
   ```
