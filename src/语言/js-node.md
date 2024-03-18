1. node 怎么处理并发




2. Map 存储
   ```js
    let obj = { name: 'Alice' };
    const map = new Map();
    map.set(obj, 123);

    console.log(map.get(obj)); // 输出 123

    // 修改 obj 的引用
    const newObj = { name: 'Bob' };
    obj = newObj;

    console.log(map.get(obj)); // 输出 undefined，因为 obj 的引用已经变化了
   ```

