---
title: "Hugo 博客自动部署：GitHub Actions + 宝塔 Nginx"
date: 2026-05-25
tags: ["Hugo", "GitHub Actions", "DevOps", "Nginx"]
categories: ["自动化部署"]
toc: true
---

## 背景

博客使用 Hugo 静态生成，部署在一台阿里云服务器上，通过宝塔面板管理 Nginx。每次修改后手动 `hugo` 构建再上传太麻烦，于是用 GitHub Actions 实现推送即部署。

## 整体流程

```
本地开发、测试（hugo server）→ git push → GitHub Actions 触发
    → 拉取代码 → 安装 Hugo → hugo --minify 生成静态文件
    → rsync 同步 public/ 到服务器 Nginx 目录
```

服务器上不需要安装 Hugo，只需要 Nginx 托管静态文件。构建工作全部在 GitHub Actions 中完成。

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
| `hugo --minify` | 构建并压缩静态文件 |
| `webfactory/ssh-agent` | 将 GitHub Secret 中的私钥注入 SSH 会话 |
| `rsync -avz --delete` | 增量同步，`--delete` 删除服务器上已移除的文件 |
| `--no-perms --no-owner --no-group` | 跳过权限同步，避免权限报错 |
| `--exclude='.user.ini'` | 排除宝塔生成的文件，避免删除报错 |

## 注意事项

### 修改主题的正确方式

不要直接改 `themes/` 下的文件。需要在项目根目录创建同名文件来覆盖：

| 需要修改 | 创建文件 |
|----------|----------|
| 翻译文字 | `i18n/zh-cn.yaml` |
| 页面结构 | `layouts/` 下同名文件 |
| 样式 | `static/css/custom_style.css`

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
