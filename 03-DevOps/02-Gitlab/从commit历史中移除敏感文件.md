# 删除commit历史中的敏感信息

当历史文件中存在敏感信息时（如AccessKey，smtp码），即使修改掉敏感信息/删除文件，这些敏感信息仍旧存在于历史commit中，最好的方法是马上删掉AccessKey，smtp码。这里也提供一些方法删除commit里的敏感信息，但是Github对历史commit有缓存，导致即使项目里面原commit id已经不存在，但还是能通过commit id访问到旧代码

## 方式一：filter-branch

### 备份最新的文件

### 执行git filter-branch

其中，PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA 是你需要删除的敏感信息文件名

```shell
git filter-branch --force --index-filter \
'git rm - --cached --ignore-unmatch ./k8s-practice/03-DevOps/02-Gitlab/docker-compose部署Gitlab.md' \
--prune-empty --tag-name-filter cat -- --all
```

![image-20220313132910687](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203131329757.png)

### 将备份文件导回并add

```shell
# 将之前备份的文件copy回来
git add /k8s-practice/03-DevOps/02-Gitlab/docker-compose部署Gitlab.md
```

### 覆盖commit（作用于所有分支）

```shell
git push origin --force --all
```

