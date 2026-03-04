---
title: hexo部署报错Error Spawn failed
date: 2026-03-04 21:23:11
tags:
 - github
 - hexo
categories:
 - blog
---

执行hexo d命令之后报错：

> FATAL Something's wrong. Maybe you can find the solution here: https://hexo.io/docs/troubleshooting.html
>
> Error: Spawn failed
>
>   at ChildProcess.<anonymous> (/Users/cy/Documents/Younger0911.github.io/node_modules/hexo-deployer-git/node_modules/hexo-util/lib/spawn.js:51:21)
>
>   at ChildProcess.emit (node:events:508:20)
>
>   at ChildProcess._handle.onexit (node:internal/child_process:293:12)



<!-- more -->



生成 SSH 密钥（如果还没有）：

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

将公钥（~/.ssh/id_ed25519.pub）添加到 GitHub SSH keys。

测试连接：

```bash
ssh -T git@github.com
```

看到欢迎信息即成功。

修改 _config.yml：

**注意是git@**

```yaml
deploy:
  type: git
  repository: git@github.com:Younger0911/Younger0911.github.io.git
  branch: master   # 与远程默认分支一致
```

再次部署：

```bash
hexo clean && hexo generate && hexo deploy
```
