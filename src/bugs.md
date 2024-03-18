# bugs
- 闭包问题：
  在 onMounted 注册事件后没有注销，导致再次进入页面时事件处理函数引用的ref为null