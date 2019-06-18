---
title: '[github][2][github相关]'
date: 2019-05-28 17:00:51
tags:
    - GitHub
categories:
    - GitHub
---
## 2 github相关

### 2.2 Issue

开发者为了跟踪bug并进行软件相关讨论，进而方便管理，创建了Issue

- GitHub的Issue和评论可以使用GFM语法进行描述

- Issue可以通过添加标签(Label)来进行整理，标签可以自由创建

- 可以通过添加里程碑来管理Issue，设置里程碑就可以通过Issue来管理任务

- 只要按照特定的格式描述提交信息，就可以像一般BTS带有的功能那样对Issue进行操作
    - 每个Issue标题下面都分配了一个诸如"#24"一样的序号，只要在提交信息的描述中加入"#24"，就可以在Issue中显示相关信息
    - 当一个处于`open`状态的`Issue`处理完毕，只要在该提交中以下列任意一种格式描述提交信息，对应的`Issue`就会被`close`
        ````
        fix #24
        fixes #24
        fixed #24
        close #24
        closes #24
        closed #24
        resolve #24
        resolves #24
        resolved #24
        ````

### 2.2 发送 Pull Request

- Pull Request是自己修改源代码后，请求对方仓库采纳该修改时采取的一种行为

- 在Github上发送Pull Request后，接收方的仓库会创建一个附带源代码的Issue，我们在这个Issue中记录详细内容。这就是Pull Request

- 发送过来的Pull Request是否被采纳，要由接收方仓库的管理者进行判断

- 一般说来，在Github上修改对方的代码，需要先将仓库fork到本地，然后再修改代码，发送Pull Request。但是，如果用户对该仓库有编辑权限，则可以直接创建分支，从分支上发送Pull Request。利用这一设计，团队开发时不妨为每一个成员赋予编辑权限，免去`fork`仓库的麻烦。这样，成员在有需要时就可以创建自己的分支，然后直接向master分支等发送Pull Request

### 2.3 接收 Pull Request

有两种Pull Request方式:
1. 直接在github上接受此次Pull Request
2. 拉取到本地，然后校验，校验之后，执行`git merge`，解决冲突后，push到远端


### 2.4 团队开发的步骤

自己总结的开发流程
1. git pull 更新本地
2. 从master新建分支并切换到该分支，以开发新功能
3. 开发新功能
4. 将此分支推送到远程仓库
5. 在github上发送pull request请求
6. 管理员将此分支拉取到本地
7. 在本地将此分支合并到master分支并消除冲突

Github Flow的开发流程
1. 令master分支时常保持可以部署的状态
2. 进行新的工作时要从master分支创建新分支，新分支名称要具有描述性
3. 在`2`新建的本地仓库分支中进行提交
4. 在Github端仓库创建同名分支，定期push
5. 需要帮助或者反馈时创建Pull Request，以Pull Request进行交流和讨论
6. 让其他开发者进行审查，确定作业完成后与master分支合并
7. 与master分支合并后立即部署