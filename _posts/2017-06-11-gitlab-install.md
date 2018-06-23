---
layout: post
title: "Centos 7 安装 gitlab"
description: "gitlab 安装教程以及使用已安装的 nginx"
date: 2017-06-11
tags: [工具,gitlab]
comments: true
share: true
---

## 安装 gitlab

### 安装配置依赖项

```bash
sudo yum install curl policycoreutils openssh-server openssh-clients
sudo systemctl enable sshd
sudo systemctl start sshd
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
sudo firewall-cmd --permanent --add-service=http
sudo systemctl reload firewalld
```

### 安装 gitlab

```
curl -sS http://packages.gitlab.cc/install/gitlab-ce/script.rpm.sh | sudo bash
sudo yum install gitlab-ce
```

### 启动 gitlab

```bash
sudo gitlab-ctl reconfigure
```

## 使用现有 nginx

### 禁用 gitlab 自带的 nginx

修改 `/etc/gitlab/gitlab.rb` 文件，将 `# nginx['enable']=true` 修改为 `nginx['enable']=false`

### 将 gitlab 生成的 nginx 配置复制到已有 nginx 的配置目录下

```bash
cp /var/opt/gitlab/nginx/conf/gitlab-http.conf /ect/nginx/conf.d/gitlab-http.conf
```

### 启动 nginx

```bash
nginx -t // 语法检查
nginx: [emerg] unknown log format "gitlab_access" in /etc/nginx/conf.d/gitlab-test.conf:58
nginx: configuration file /etc/nginx/nginx.conf test failed //报错的话直接删掉 gitlab_access 即可
nginx //启动 nginx
```

## 遇到的问题

* 安装完成启动 gitlab 后无法访问。提示连接被拒绝

  **原因:** 因为是新机器 `firewall` 服务未启动，`sudo firewall-cmd --permanent --add-service=http` 设置未生效。

  **解决方法:**

  ```bash
  sudo systemctl start firewalld
  sudo firewall-cmd --permanent --add-service=http
  sudo systemctl reload firewalld
  ```

* 使用已有 nginx 后访问 gitlab 502

  **原因:** 文件访问权限问题.

  **解决方法:**

  ```bash
  sudo chmod -R o+x /var/opt/gitlab/gitlab-rails //简单粗暴
  ```
