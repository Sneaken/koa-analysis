# delegates

> 把target上的部分内容挂载到 proto 上

## 1. 源码

```js

/**
 * Expose `Delegator`.
 */

module.exports = Delegator;

/**
 * Initialize a delegator.
 *
 * @param {Object} proto
 * @param {String} target
 * @api public
 */

function Delegator (proto, target) {
  // 必须实例化 其实
  if (!(this instanceof Delegator)) return new Delegator(proto, target);
  // 原对象
  this.proto = proto;
  // 目标键
  this.target = target;
  // methods
  this.methods = [];
  // getters
  this.getters = [];
  // setters
  this.setters = [];
  // TODO: ?
  this.fluents = [];
}

/**
 * Automatically delegate properties
 * from a target prototype
 *
 * @param {Object} proto
 * @param {object} targetProto
 * @param {String} targetProp
 * @api public
 */

Delegator.auto = function (proto, targetProto, targetProp) {
  // 把 proto[targetProto] 里面的东西全部绑到 proto 上
  var delegator = Delegator(proto, targetProp);
  // 获取 targetProto 上面的属性
  var properties = Object.getOwnPropertyNames(targetProto);
  for (var i = 0; i < properties.length; i++) {
    var property = properties[i];
    var descriptor = Object.getOwnPropertyDescriptor(targetProto, property);
    if (descriptor.get) {
      // 有 getter 定义 gatter
      delegator.getter(property);
    }
    if (descriptor.set) {
      // 有 setter 定义 setter
      delegator.setter(property);
    }
    if (descriptor.hasOwnProperty('value')) { // could be undefined but writable
      var value = descriptor.value;
      if (value instanceof Function) {
        // 如果是函数的话
        delegator.method(property);
      } else {
        // 如果是其他内容的话，设置 getter
        delegator.getter(property);
      }
      if (descriptor.writable) {
        // 如果是可写的 那么设置 setter
        delegator.setter(property);
      }
    }
  }
};

/**
 * Delegate method `name`.
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.method = function (name) {
  var proto = this.proto;
  var target = this.target;
  this.methods.push(name);

  // proto 上面 挂载 [name] 函数
  proto[name] = function () {
    // this  实际上指的就是 这个 proto
    return this[target][name].apply(this[target], arguments);
  };

  return this;
};

/**
 * Delegator accessor `name`.
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.access = function (name) {
  // 同时挂载 getter 和 setter
  return this.getter(name).setter(name);
};

/**
 * Delegator getter `name`.
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.getter = function (name) {
  var proto = this.proto;
  var target = this.target;
  this.getters.push(name);

  // 非标写法 定义 getter
  proto.__defineGetter__(name, function () {
    return this[target][name];
  });

  return this;
};

/**
 * Delegator setter `name`.
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.setter = function (name) {
  var proto = this.proto;
  var target = this.target;
  this.setters.push(name);

  // 非标写法 定义 setter
  proto.__defineSetter__(name, function (val) {
    return this[target][name] = val;
  });

  return this;
};

/**
 * Delegator fluent accessor
 *
 * @param {String} name
 * @return {Delegator} self
 * @api public
 */

Delegator.prototype.fluent = function (name) {
  var proto = this.proto;
  var target = this.target;
  this.fluents.push(name);

  proto[name] = function (val) {
    if ('undefined' != typeof val) {
      // val 存在的时候就是值
      this[target][name] = val;
      return this;
    } else {
      // 不存在的时候就是读取值
      return this[target][name];
    }
  };

  return this;
};
```