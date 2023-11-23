---
icon: simple/git
---

#### 与远程仓库保持一致

```bash
git fetch --all && git reset --hard origin/master && git pull
```

#### 按需克隆(部分克隆)

当仓库体积较大时，就可以考虑使用部分克隆来提升开发过程中的效率及体验.

使用`blob:none`选项
```bash
git clone --filter=blob:none --no-checkout git@codeup.aliyun.com:6125fa3a03f23adfbed12b8f/linux.git
```

#### 微服务单根代码仓

近年来，越来越多项目选择了使用微服务的架构，将大单体服务拆分为若干个内聚化的微型服务，每一个服务由一个微型团队进行维护，团队间开发可以并行、互不干扰，团队间协同复杂度大幅降低。
单根代码仓，公用代码更易于共享，项目文档、流程规范可以集中于一处，也更加易于实施持续集成。配合部分克隆配合稀疏检出特性，可以解决开发人员只关注项目中某一部分，他也不得不克隆整个仓库的问题。

!!! example
    ```bash
        monorepo
        ├── README
        ├── backend
        │   └── command
        │       └── command.go
        ├── docs
        │   └── api_specification
        ├── frontend
        │   ├── README.md
        │   └── src
        │       └── main.js
        └── libraries
            └── common.lib
    ```

1. 首先进行部分克隆
    ```bash
    git clone --filter=blob:none --no-checkout https://codeup.aliyun.com/61234c2d1bd96aa110f27b9c/monorepo.git
    ```
2. 开启稀疏检出
   ```bash
    cd monorepo
    git config core.sparsecheckout true
    echo "backend/*" > .git/info/sparse-checkout
   ```

3. 检出指定目录
   ```bash
    git checkout
   ```

更详细的关于部分检出可参考 [部分检出](https://help.aliyun.com/document_detail/309002.html?spm=a2c4g.324161.0.i1)