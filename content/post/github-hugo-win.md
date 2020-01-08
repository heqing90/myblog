---
title: "Windows使用Github Pages + Hugo 搭建个人博客"
date: 2020-01-05T22:43:26+08:00
draft: true
tags: 
    - "windows"
    - "github"
    - "hugo"
categories: 
    - "starter"
---

## 环境准备

1. 安装Hugo
    - 下载Hugo Release 对应的Windows 版本 [hugo.exe](https://github.com/gohugoio/hugo/releases)
    - 解压Zip, 将*hugo.exe*路径添加到环境变量**Path**中
    - 检查 hugo 是否能正常运行
       >hugo version  
        Hugo Static Site Generator v0.62.2-83E50184 windows/amd64 BuildDate: 2020-01-05T18:53:21Z 
2. 配置 Github Pages
    - 登录/注册 Github
    - 新建Repo， 名称为\<username>.github.io
    - 在Settings->Custom Domain 中可选配置用户域名

## 使用 Hugo

1. 使用 *hugo* 创建网站模板
    - 进入“\<yourpath>\<username>.github.io” 目录
    - 运行 *“hugo new site .”* 命令， 在当前目录下创建模板
2. 添加 *hugo* 主题（themes）
    - 在 [hugo themes](https://themes.gohugo.io/) 上选择一个主题
    - 进入“\<yourpath>\\\<username>.github.io\themes” 目录使用git clone 下载主题
    - 在项目文件 ___config.toml___ 中将themes配置成对应主题名称
3. 新建一个新页面
    - 运行 *hugo new \<filename>.md*
    - 使用 __markdown__ 编辑该文件
4. 本地启动 *hugo*
    - 进入 “\<yourpath>/\<username>.github.io” 目录
    - 运行 *hugo server -D*
    - 访问 http://localhost:1313
      ![Demo](http://q3o8nhuc8.bkt.clouddn.com/demo.jpg)

## 部署 hugo 到 Github [host on github](https://gohugo.io/hosting-and-deployment/hosting-on-github/)

1. 使用 ***git submodule***
    - 删除旧的 *pulic* 目录
    - 运行 *git submodule add yourproject.git public*
    - 提交 *public*

        ```shell
        #!/bin/bash
        # If a command fails then the deploy stops
        set -e

        printf "\033[0;32mDeploying updates to GitHub...\033[0m\n"

        # Build the project.
        hugo # if using a theme, replace with `hugo -t <YOURTHEME>`

        # Go To Public folder
        cd public

        # Add changes to git.
        git add .

        # Commit changes.
        msg="rebuilding site $(date)"
        if [ -n "$*" ]; then
            msg="$*"
        fi
        git commit -m "$msg"

        # Push source and build repos.
        git push origin master
        ```

2. 使用 *github.io* 域名访问
    - 将本地工程push到github
    - 访问 *http://username.github.io*