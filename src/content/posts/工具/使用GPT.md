---
title: 使用ChatGPT
published: 2025-08-23
category: 工具
draft: false
---

- 设置代理地址为支持 ChatGPT 的国家或地区。
- 清除浏览器 cookie。
- 地址栏输入

  ```js
  javascript:window.localStorage.removeItem(Object.keys(window.localStorage).find(i=>i.startsWith('@@auth0spajs')))
  ```
  其中javascript:需要手动输入。
  以上方法根据情况自由选择搭配使用（代理必须要做）。清除 cookie 需要重启浏览器。清除缓存不需要。