---
title: "Activity生命周期,md"
date: 2022-02-24T20:38:51+08:00
draft: true
---

### 一般情况

1. onCreate:表示Activity正在被创建。
2. onRestart:表示Activity正在被重新启动。
3. onStart:表示Activity正在被启动，这时**已经可见，但没有出现在前台无法进行交互**。
4. onResume:表示Activity已经可见，并且处于前台。
5. onPause:表示Activity正在停止(可做一次保存状态停止动画等非耗时操作)。
6. onStop:表示Activity即将停止(可进行重量级回收工作)。
7. onDestroy:表示Activity即将被销毁。