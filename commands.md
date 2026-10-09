
# git commands

```bash
git tag v0.4.2
git push origin v0.4.2
git push origin --delete v0.4.2

git tag -f v0.4.2 # 强制把 tag 移到最新提交 
git push -f origin v0.4.2 # 覆盖远端 tag → 触发 workflow 重跑

git clone --depth 1 https://github.com/hexiongwu1995/theoframe .
git fetch --deepen 5
```
