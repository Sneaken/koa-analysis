# only

## 源码

```js
module.exports = function (obj, keys) {
  // 默认空对象
  obj = obj || {};
  // 如果 keys 是 字符串的话 就按空格分割
  if ("string" == typeof keys) keys = keys.split(/ +/);
  // keys: string[]
  return keys.reduce(function (ret, key) {
    // 取不到值就算了
    if (null == obj[key]) return ret;
    // 能取到值就添加到结果列表里面
    ret[key] = obj[key];
    return ret;
  }, {});
};
```
