---
title: github 
tags: github,git
grammar_cjkRuby: true
---

### 初始化一个项目


网页创建项目

git init

设置改项目提交的用户信息

git config user.name "boying"
git config user.email "boyinnju@gmail.com"

git add .
git commit -m ''

git remote add origin git@github.com:yourName/yourRepo.git
(失败的话，用 先git remote rm origin)

git push --set-upstream origin master

git branch --set-upstream-to=origin/master master

git push