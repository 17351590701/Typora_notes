# Typora跳转

**超链接**跳转标题（可跨文件跳转）

-  [跳转本md内标题](#搭建图床)

- [跳转其他md内标题](F:/Typora文件/其他/。。。.md#标题名)

# 搭建图床

## Typora+Gitee+PicGo

1. 下载typora

2. 注册Gitee https://gitee.com/

3. 新建仓库，记住此**仓库地址**，注意，设置为==开源==（重要），初始化仓库，分支<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061730548.png" alt="image-20240606173012479" style="zoom:67%;" />

4. 创建保存**私人令牌** 

   - ==私人令牌==

     ```shell
     57305bd9e29b851f0d3cd46af97235e3
     ```

     <img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061657639.png" alt="image-20240606162829942" style="zoom:67%;" />

5. 仓库**初始化**readme文件

   1. <img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061657640.png" alt="image-20240606163041608" style="zoom: 67%;" />

6. 下载PicGo https://github.com/Molunerfinn/PicGo/releases

7. 搜索下载插件 **gitee-uploader** ，**重启**PicGo<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061657641.png" alt="image-20240606163513301" style="zoom:67%;" />

8. PicGo->**==图床设置==**->**Gitee**  默认图床

   - ==repo==：仓库地址，截取 https://gitee.com/yurun-zhang/typora-tu-chuang.git 中间部分
   - ==token==:私人令牌
   - <img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061657642.png" alt="image-20240606164132116" style="zoom:67%;" />

9. PicGo->PicGo设置->**Server设置（通过http实现upload）**，同时设置开机自启，**时间戳重命名**<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061657643.png" alt="image-20240606164425562" style="zoom:67%;" />

10. ==图床接入==：

    - 注意PicGo路径选择Typora.exe文件，验证图片上传选项<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406061657645.png" alt="image-20240606165509894" style="zoom:67%;" />

11. 图片压缩插件 `compression` 作者 `Krins` （==设置可能不生效==）<img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406112158478.png" alt="image-20240611215812431" style="zoom: 80%;" /><img src="https://gitee.com/yurun-zhang/typora-tu-chuang/raw/master/202406112157932.png" alt="image-20240611215738886" style="zoom:80%;" />

> - `容许质量下降`：若您能够接受图片质量的轻微下降，可将该属性设置为 true 来获得更大压缩比
> - `图片质量`：取值范围 5 ~ 100（默认为 0）。数字越大，图像质量越好， 但相应能压缩的文件体积也较少（若您不确定具体的数值，请将其设置为 0，我们将根据具体情况帮您决定）
