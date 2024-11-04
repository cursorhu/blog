# BH201 Linux/Android patch代码合并手册

## 代码git管理

BH201 patch git代码的管理方式：

1. master分支管理各种SOC vendor的BH201 patch，跟随客户新需求（Linux/Android版本）持续更新
2. 各客户的需求用独立分支管理，在该分支创建客户提供的原始代码，不断提交客户反馈的问题和issue fix，全部OK后合并此客户sd/mmc host的代码到master。

新客户需求转化成git代码管理：

1.创建新客户需求分支：

从master或者此soc分支创建新分支（命名：AE task ID - SOC - Kernel - Android），再删除.git以外的旧代码，粘贴客户提供的mmc/host代码，作为初始分支代码

```
# 拉取仓库（bh201-android）
git clone http://10.52.1.103/software/bh201-android.git

# 从master或其他分支创建新分支
git checkout -b AE#211-mtk8792-kernel6.1-android15
git push origin AE#211-mtk8792-kernel6.1-android15

# 初始化分支代码和文件
删除bh201-android目录原有的所有代码（保留.git）
复制客户给的mmc/host代码，以及FAE提供的requirement excel(包含SOC的SD host地址，CD#极性信息)

# 推送新分支
git add .
git commit -m "create based on customer source code. AE task#211 [Amazon/Huaqin/BH201][internal] Spinel MTK8792, BH201 patch"
git push --set-upstream origin AE#211-mtk8792-kernel6.1-android15
git push
git log
```

2.合并此SOC的BH201 patch到此客户新分支

参考master的BH201 patch合并手册