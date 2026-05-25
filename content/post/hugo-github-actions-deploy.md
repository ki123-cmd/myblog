---
title: "Hugo 博客自动部署：GitHub Actions + 宝塔 Nginx"
date: 2026-05-25
tags: ["Hugo", "GitHub Actions", "DevOps", "Nginx"]
categories: ["技术"]
toc: true
---

## 背景

博客使用 Hugo 静态生成，部署在一台阿里云服务器上，通过宝塔面板管理 Nginx。每次修改后手动 `hugo` 构建再上传太麻烦，于是用 GitHub Actions 实现推送即部署。

## 整体流程

```
本地写文章 → git push → GitHub Actions 触发
    → 拉取代码（含子模块）→ 安装 Hugo → hugo --minify
    → rsync 同步 public/ 到服务器 Nginx 目录
```

## 服务器端配置

### 1. 宝塔建站

在宝塔面板添加纯静态站点：

| 配置项 | 值 |
|--------|-----|
| 域名 | code-review.top |
| 根目录 | /www/wwwroot/code-review.top |
| PHP 版本 | 纯静态 |

### 2. 生成 SSH 密钥

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/deploy_key -N ""
cat ~/.ssh/deploy_key.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/deploy_key  # 复制私钥，添加到 GitHub Secrets
```

## GitHub 配置

### 3. Secrets

在仓库 Settings → Secrets and variables → Actions 中添加：

| Secret | 值 |
|--------|-----|
| `SSH_PRIVATE_KEY` | 服务器私钥内容 |
| `REMOTE_HOST` | 服务器 IP |
| `REMOTE_USER` | SSH 用户名 |
| `REMOTE_PATH` | Nginx 根目录路径 |

### 4. Workflow 文件

`.github/workflows/deploy.yml`：

```yaml
name: Deploy to Server

on:
  push:
    branches: [master]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: true

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.146.0"
          extended: true

      - name: Build
        run: hugo --minify

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Deploy
        run: |
          rsync -avz --delete --no-perms --no-owner --no-group \
            --exclude='.user.ini' \
            -e "ssh -o StrictHostKeyChecking=no" \
            public/ ${{ secrets.REMOTE_USER }}@${{ secrets.REMOTE_HOST }}:${{ secrets.REMOTE_PATH }}
```

## 关键步骤解释

| 步骤 | 说明 |
|------|------|
| `submodules: true` | 主题是 git 子模块，必须一并拉取 |
| `hugo --minify` | 构建并压缩静态文件 |
| `webfactory/ssh-agent` | 将 GitHub Secret 中的私钥注入 SSH 会话 |
| `rsync -avz --delete` | 增量同步，`--delete` 删除服务器上已移除的文件 |
| `--no-perms --no-owner --no-group` | 跳过权限同步，避免权限报错 |
| `--exclude='.user.ini'` | 排除宝塔生成的文件，避免删除报错 |

## 注意事项

### 子模块修改问题

主题目录 `themes/hugo-theme-next/` 是 git 子模块，对其中文件的修改**不会被推送到仓库**。GitHub Actions 拉取的是原始主题代码。

如需覆盖主题配置，在项目根目录的对应目录下创建 Hugo 查找顺序更高的文件：

- **i18n 覆盖**：`i18n/zh-cn.yaml`
- **模板覆盖**：`layouts/` 下的同名文件
- **样式覆盖**：`static/css/custom_style.css`

### rsync 常见错误

| 退出码 | 原因 | 解决 |
|--------|------|------|
| 12 | 目标目录不存在 | `mkdir -p` 创建目录 |
| 23 | 权限/文件无法删除 | 添加 `--no-perms --exclude='.user.ini'` |
| 255 | SSH 认证失败 | 检查私钥是否正确加入 `authorized_keys` 和 GitHub Secret |

## 总结

配置完成后，日常流程只有一句：

```bash
git add . && git commit -m "update" && git push origin master
```

推送后去 [Actions](https://github.com/ki123-cmd/myblog/actions) 查看构建进度，绿色即部署成功。
