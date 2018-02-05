---
title: 写ansible playbook脚本批量取回服务器上的文件
date: 2018-01-25 07:03:03
categories: 杂项
tags: ansible
---

> fetch.yaml

```
- hosts:  all
  remote_user: root

  tasks:
  - name: fucking
    find:
      paths: /path/
      patterns: "*.log"
      recurse: yes
    register: file_2_fetch

  - name: fuck your bitch
    fetch:
      src: "{{ item.path }}"
      dest: /dest_path/
      flat: yes
    with_items: "{{ file_2_fetch.files }}"
```

