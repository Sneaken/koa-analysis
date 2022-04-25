# http-errors

> 是 koa、express 社区使用比较广泛的基础库，主要用于处理 HTTP Error。
>
> 当前版本 2.0.0 和 koa 内置的不是一个版本

## 1. 相关依赖

### 1.1 setprototypeof

> Polyfill for Object.setPrototypeOf

### 1.2 statuses

> HTTP status utility for node.
> This module provides a list of status codes and messages sourced from a few different projects:
> The IANA Status Code Registry
> The Node.js project
> The NGINX project
> The Apache HTTP Server project

```js
// import { codes, message } from 'statuses'
// console.log(codes)
// [
//   100, 101, 102, 103, 200, 201, 202, 203, 204,
//   205, 206, 207, 208, 226, 300, 301, 302, 303,
//   304, 305, 306, 307, 308, 400, 401, 402, 403,
//   404, 405, 406, 407, 408, 409, 410, 411, 412,
//   413, 414, 415, 416, 417, 418, 421, 422, 423,
//   424, 425, 426, 428, 429, 431, 451, 500, 501,
//   502, 503, 504, 505, 506, 507, 508, 509, 510,
//   511
// ]
// console.log(message)
// {
//   '100': 'Continue',
//   '101': 'Switching Protocols',
//   '102': 'Processing',
//   '103': 'Early Hints',
//   '200': 'OK',
//   '201': 'Created',
//   '202': 'Accepted',
//   '203': 'Non-Authoritative Information',
//   '204': 'No Content',
//   '205': 'Reset Content',
//   '206': 'Partial Content',
//   '207': 'Multi-Status',
//   '208': 'Already Reported',
//   '226': 'IM Used',
//   '300': 'Multiple Choices',
//   '301': 'Moved Permanently',
//   '302': 'Found',
//   '303': 'See Other',
//   '304': 'Not Modified',
//   '305': 'Use Proxy',
//   '306': '(Unused)',
//   '307': 'Temporary Redirect',
//   '308': 'Permanent Redirect',
//   '400': 'Bad Request',
//   '401': 'Unauthorized',
//   '402': 'Payment Required',
//   '403': 'Forbidden',
//   '404': 'Not Found',
//   '405': 'Method Not Allowed',
//   '406': 'Not Acceptable',
//   '407': 'Proxy Authentication Required',
//   '408': 'Request Timeout',
//   '409': 'Conflict',
//   '410': 'Gone',
//   '411': 'Length Required',
//   '412': 'Precondition Failed',
//   '413': 'Payload Too Large',
//   '414': 'URI Too Long',
//   '415': 'Unsupported Media Type',
//   '416': 'Range Not Satisfiable',
//   '417': 'Expectation Failed',
//   '418': "I'm a teapot",
//   '421': 'Misdirected Request',
//   '422': 'Unprocessable Entity',
//   '423': 'Locked',
//   '424': 'Failed Dependency',
//   '425': 'Unordered Collection',
//   '426': 'Upgrade Required',
//   '428': 'Precondition Required',
//   '429': 'Too Many Requests',
//   '431': 'Request Header Fields Too Large',
//   '451': 'Unavailable For Legal Reasons',
//   '500': 'Internal Server Error',
//   '501': 'Not Implemented',
//   '502': 'Bad Gateway',
//   '503': 'Service Unavailable',
//   '504': 'Gateway Timeout',
//   '505': 'HTTP Version Not Supported',
//   '506': 'Variant Also Negotiates',
//   '507': 'Insufficient Storage',
//   '508': 'Loop Detected',
//   '509': 'Bandwidth Limit Exceeded',
//   '510': 'Not Extended',
//   '511': 'Network Authentication Required',
// }
```

### 1.3 inherits

> Browser-friendly inheritance fully compatible with standard node.js inherits.

### 1.4 toidentifier

> Convert a string of words to a JavaScript identifier

## 2. 源码

### 2.1 工具函数

#### 2.1.1 toClassName

```js
/**
 * Get a class name from a name identifier.
 * @private
 */

function toClassName (name) {
  return name.substr(-5) !== "Error" ? name + "Error" : name;
}
```

#### 2.1.2 codeClass

```js
/**
 * Get the code class of a status code.
 * @private
 */

function codeClass (status) {
  return Number(String(status).charAt(0) + "00");
}
```

#### 2.1.3 nameFunc

```js
/**
 * Set the name of a function, if possible.
 * @private
 */

function nameFunc (func, name) {
  var desc = Object.getOwnPropertyDescriptor(func, "name");

  if (desc && desc.configurable) {
    desc.value = name;
    Object.defineProperty(func, "name", desc);
  }
}
```

#### 2.1.4 populateConstructorExports

```js
/**
 * Populate the exports object with constructors for every error class.
 * @private
 */

function populateConstructorExports (exports, codes, HttpError) {
  // HttpError是抽象类

  codes.forEach(function forEachCode (code) {
    var CodeError;
    // 转换成合法变量名
    var name = toIdentifier(statuses.message[code]);

    // 当是 400 或者 500 时， 做一些操作
    switch (codeClass(code)) {
      case 400:
        // 客户端错误
        CodeError = createClientErrorConstructor(HttpError, name, code);
        break;
      case 500:
        // 服务端错误
        CodeError = createServerErrorConstructor(HttpError, name, code);
        break;
    }

    if (CodeError) {
      // 挂在 exports 上面
      // export the constructor
      exports[code] = CodeError;
      exports[name] = CodeError;
    }
  });
}
```

#### 2.1.5 createClientErrorConstructor

> 客户端错误

```js
/**
 * Create a constructor for a client error.
 * @private
 */

function createClientErrorConstructor (HttpError, name, code) {
  // 统一类名末尾带 Error
  var className = toClassName(name);

  function ClientError (message) {
    // create the error object
    // 填充 message
    var msg = message != null ? message : statuses.message[code];
    var err = new Error(msg);

    // capture a stack trace to the construction point
    Error.captureStackTrace(err, ClientError);

    // adjust the [[Prototype]]
    setPrototypeOf(err, ClientError.prototype);

    // 为什么要用 Object.defineProperty ?
    // err.message 原来是 不可枚举的
    // err.name 原来是 Error, 并且想改成不可枚举的

    // redefine the error message
    Object.defineProperty(err, "message", {
      enumerable: true,
      configurable: true,
      value: msg,
      writable: true,
    });


    // redefine the error name
    Object.defineProperty(err, "name", {
      enumerable: false,
      configurable: true,
      value: className,
      writable: true,
    });

    return err;
  }

  // 试 ClientError 继承自 HttpError
  inherits(ClientError, HttpError);

  // 修改函数的name
  nameFunc(ClientError, className);

  ClientError.prototype.status = code;
  ClientError.prototype.statusCode = code;
  // client 错误 需要暴露
  ClientError.prototype.expose = true;

  return ClientError;
}
```

#### 2.1.6 createServerErrorConstructor

> 服务端错误

```js
/**
 * Create a constructor for a server error.
 * @private
 */

function createServerErrorConstructor (HttpError, name, code) {
  var className = toClassName(name);

  function ServerError (message) {
    // create the error object
    var msg = message != null ? message : statuses.message[code];
    var err = new Error(msg);

    // capture a stack trace to the construction point
    Error.captureStackTrace(err, ServerError);

    // adjust the [[Prototype]]
    setPrototypeOf(err, ServerError.prototype);

    // redefine the error message
    Object.defineProperty(err, "message", {
      enumerable: true,
      configurable: true,
      value: msg,
      writable: true,
    });

    // redefine the error name
    Object.defineProperty(err, "name", {
      enumerable: false,
      configurable: true,
      value: className,
      writable: true,
    });

    return err;
  }

  inherits(ServerError, HttpError);
  nameFunc(ServerError, className);

  ServerError.prototype.status = code;
  ServerError.prototype.statusCode = code;
  // server 错误 不需要暴露
  ServerError.prototype.expose = false;

  return ServerError;
}
```

#### 2.1.7 createIsHttpErrorFunction

```js
/**
 * Create function to test is a value is a HttpError.
 * @private
 */

function createIsHttpErrorFunction (HttpError) {
  return function isHttpError (val) {
    if (!val || typeof val !== "object") {
      return false;
    }

    if (val instanceof HttpError) {
      return true;
    }

    // 这里其实也就是自己定义的规则
    return (
      val instanceof Error &&
      typeof val.expose === "boolean" &&
      typeof val.statusCode === "number" &&
      val.status === val.statusCode
    );
  };
}
```

#### 2.1.8 createHttpErrorConstructor

> 用来创建一个抽象类 感觉我们基本上用不上

```js
/**
 * Create HTTP error abstract base class.
 * @private
 */

function createHttpErrorConstructor () {
  function HttpError () {
    throw new TypeError("cannot construct abstract class");
  }

  inherits(HttpError, Error);

  return HttpError;
}
```

### 2.2 主体代码

```js
/**
 * Module dependencies.
 * @private
 */

var deprecate = require("depd")("http-errors");
var setPrototypeOf = require("setprototypeof");
var statuses = require("statuses");
var inherits = require("inherits");
var toIdentifier = require("toidentifier");

/**
 * Module exports.
 * @public
 */

module.exports = createError;
module.exports.HttpError = createHttpErrorConstructor();
module.exports.isHttpError = createIsHttpErrorFunction(
  module.exports.HttpError
);

// 把 Code 和 对应的 Message 挂载到默认导出上
// Populate exports for all constructors
populateConstructorExports(
  module.exports,
  statuses.codes,
  module.exports.HttpError
);

/**
 * Create a new HTTP Error.
 *
 * @returns {Error}
 * @public
 */

function createError () {
  // so much arity going on ~_~
  var err;
  var msg;
  var status = 500;
  var props = {};
  for (var i = 0; i < arguments.length; i++) {
    var arg = arguments[i];
    var type = typeof arg;
    if (type === "object" && arg instanceof Error) {
      err = arg;
      status = err.status || err.statusCode || status;
    } else if (type === "number" && i === 0) {
      status = arg;
    } else if (type === "string") {
      msg = arg;
    } else if (type === "object") {
      props = arg;
    } else {
      // 参数不能是其他类型的
      throw new TypeError("argument #" + (i + 1) + " unsupported type " + type);
    }
  }

  // 仅支持 4xx || 5xx 的错误码 其他的不应该属于 错误的范畴
  if (typeof status === "number" && (status < 400 || status >= 600)) {
    deprecate("non-error status code; use only 4xx or 5xx status codes");
  }

  // status 不正常的时候 调整成 500
  if (
    typeof status !== "number" ||
    (!statuses.message[status] && (status < 400 || status >= 600))
  ) {
    status = 500;
  }

  // constructor
  // 之前挂载到 createError 上面的类
  var HttpError = createError[status] || createError[codeClass(status)];

  if (!err) {
    // create error
    err = HttpError
      ? new HttpError(msg)
      : new Error(msg || statuses.message[status]);
    Error.captureStackTrace(err, createError);
  }

  if (!HttpError || !(err instanceof HttpError) || err.status !== status) {
    // add properties to generic error
    // 当错误不属于 HttpError 的 时候 就把他们包装成 HttpError 这样

    // client 错误 需要暴露
    err.expose = status < 500;
    err.status = err.statusCode = status;
  }

  for (var key in props) {
    // 屏蔽这两个字段
    if (key !== "status" && key !== "statusCode") {
      err[key] = props[key];
    }
  }

  return err;
}
```
