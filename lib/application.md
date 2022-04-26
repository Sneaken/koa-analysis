# lib/application.js

> 应用主入口

## 1. 相关依赖

### 1.1 debug

debug exposes a function; simply pass this function the name of your module, and it will return a decorated version of
console.error for you to pass debug statements to. This will allow you to toggle the debug output for different parts of
your module as well as the module as a whole.

### 1.2 on-finished

Execute a callback when a HTTP request closes, finishes, or errors.

### 1.3 statuses

> 状态码

### 1.4 only

> return whitelisted properties of an object

### 1.5 http-errors

> 1.8.1

## 2. 源码

### 2.1 工具函数

#### 2.1.1 respond

```js
/**
 * Response helper.
 */

function respond (ctx) {

  // 中间件里面 设置 ctx.respond = false 的 时候 就可以绕过 koa 的处理 然后自己处理响应逻辑
  // allow bypassing koa
  if (ctx.respond === false) return;

  // writable 是原生的 responese 对象的可写入属性，检查是否是可写流
  if (!ctx.writable) return;

  const res = ctx.res;
  let body = ctx.body;
  const code = ctx.status;

  // ignore body
  if (statuses.empty[code]) {
    // 204, 205, 304
    // 以上响应码 不应该有 body
    // strip headers
    ctx.body = null;
    return res.end();
  }

  if (ctx.method === "HEAD") {
    // headersSent 只读属性. 如果 http 响应头已发送，则为 true，否则为 false。
    // 如果http响应头已经发送， 并且 响应头没有 Content-Length 属性， 那么添加 length 头
    if (!res.headersSent && !ctx.response.has("Content-Length")) {
      const {length} = ctx.response;
      if (Number.isInteger(length)) ctx.length = length;
    }
    return res.end();
  }

  // status body
  if (body == null) {
    if (ctx.response._explicitNullBody) {
      // TODO:  what happened?
      ctx.response.remove("Content-Type");
      ctx.response.remove("Transfer-Encoding");
      ctx.length = 0;
      return res.end();
    }
    if (ctx.req.httpVersionMajor >= 2) {
      // 如果是 http2 以上， 则将body 设置为对应的HTTP状态码
      body = String(code);
    } else {
      // 否则 设置 body 为 ctx.message, 不存在时则为状态码
      body = ctx.message || String(code);
    }
    if (!res.headersSent) {
      // 如果 http响应头 还没有发送 
      ctx.type = "text";
      ctx.length = Buffer.byteLength(body);
    }
    return res.end(body);
  }

  // responses
  // 如果 body 是 Buffer 或者 String 时， 结束请求返回结果
  if (Buffer.isBuffer(body)) return res.end(body);
  if (typeof body === "string") return res.end(body);
  // 如果 body 是 Stream 的时候， 开启管道
  if (body instanceof Stream) return body.pipe(res);

  // body: json
  // body 为 JSON 类型时， 将 body 转成字符串，
  // 并设置 length 后 返回结果
  body = JSON.stringify(body);
  if (!res.headersSent) {
    ctx.length = Buffer.byteLength(body);
  }
  res.end(body);
}
```

#### 2.1.10

```js
const {HttpError} = require("http-errors");
/**
 * Make HttpError available to consumers of the library so that consumers don't
 * have a direct dependency upon `http-errors`
 */

module.exports.HttpError = HttpError;
```

### 2.2 主函数

```js
"use strict";

/**
 * Module dependencies.
 */

const debug = require("debug")("koa:application");
const onFinished = require("on-finished");
const response = require("./response");
const compose = require("koa-compose");
const context = require("./context");
const request = require("./request");
const statuses = require("statuses");
const Emitter = require("events");
const util = require("util");
const Stream = require("stream");
const http = require("http");
const only = require("only");

/**
 * Expose `Application` class.
 * Inherits from `Emitter.prototype`.
 */

module.exports = class Application extends Emitter {
  /**
   * Initialize a new `Application`.
   *
   * @api public
   */

  /**
   *
   * @param {object} [options] Application options
   * @param {string} [options.env='development'] Environment
   * @param {string[]} [options.keys] Signed cookie keys
   * @param {boolean} [options.proxy] Trust proxy headers
   * @param {number} [options.subdomainOffset] Subdomain offset
   * @param {string} [options.proxyIpHeader] Proxy IP header, defaults to X-Forwarded-For
   * @param {number} [options.maxIpsCount] Max IPs read from proxy IP header, default to 0 (means infinity)
   *
   */

  constructor (options) {
    super();
    // 初始化参数
    options = options || {};
    this.proxy = options.proxy || false;
    this.subdomainOffset = options.subdomainOffset || 2;
    this.proxyIpHeader = options.proxyIpHeader || "X-Forwarded-For";
    this.maxIpsCount = options.maxIpsCount || 0;
    this.env = options.env || process.env.NODE_ENV || "development";
    if (options.keys) this.keys = options.keys;

    //
    this.middleware = [];
    this.context = Object.create(context);
    this.request = Object.create(request);
    this.response = Object.create(response);
    // util.inspect.custom support for node 6+
    /* istanbul ignore else */
    if (util.inspect.custom) {
      // 挺有趣的， 能够自定义当前对象的输出，而不暴露其他属性
      this[util.inspect.custom] = this.inspect;
    }
  }

  /**
   * Shorthand for:
   *
   *    http.createServer(app.callback()).listen(...)
   *
   * @param {Mixed} ...
   * @return {Server}
   * @api public
   */

  listen (...args) {
    debug("listen");
    // 所以 其实 listen 是放在最后才执行的，
    // 即 所有东西都初始化完才 listen
    const server = http.createServer(this.callback());
    // 相当于
    // const server = http.createServer();
    // server.on('request', this.callback());

    // listen的参数实际上有很多
    return server.listen(...args);
  }

  /**
   * Return JSON representation.
   * We only bother showing settings.
   *
   * @return {Object}
   * @api public
   */

  toJSON () {
    // 只允许 subdomainOffset proxy env 暴露出去
    return only(this, ["subdomainOffset", "proxy", "env"]);
  }

  /**
   * Inspect implementation.
   *
   * @return {Object}
   * @api public
   */

  inspect () {
    // 自定义需要输出的对象
    return this.toJSON();
  }

  /**
   * Use the given middleware `fn`.
   *
   * Old-style middleware will be converted.
   *
   * @param {Function} fn
   * @return {Application} self
   * @api public
   */

  use (fn) {
    // 中间件必须是一个函数
    if (typeof fn !== "function")
      throw new TypeError("middleware must be a function!");
    debug("use %s", fn._name || fn.name || "-");
    this.middleware.push(fn);
    // 支持链式调用
    return this;
  }

  /**
   * Return a request handler callback
   * for node's native http server.
   *
   * @return {Function}
   * @api public
   */

  callback () {
    // 处理成洋葱模型
    const fn = compose(this.middleware);

    // 如果没有添加-错误监听事件，则添加默认的错误监听事件
    if (!this.listenerCount("error")) this.on("error", this.onerror);

    const handleRequest = (req, res) => {
      // 初始化请求的上下文， 每个请求的上下文都是全新的
      const ctx = this.createContext(req, res);
      //TODO: 为什么要这个return 查看 issue/848 （ps 我是没看明白）
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }

  /**
   * Handle request in callback.
   *
   * @api private
   */

  handleRequest (ctx, fnMiddleware) {
    const res = ctx.res;
    // 状态码 默认 404 ?
    res.statusCode = 404;
    // 兜底的异常处理函数
    const onerror = (err) => ctx.onerror(err);
    // 响应处理回调函数
    const handleResponse = () => respond(ctx);
    // 当 res 关闭, 结束或者出错的 的时候执行回调
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }

  /**
   * Initialize a new context.
   *
   * @api private
   */

  createContext (req, res) {
    // 单一上下文原则
    // 优点
    // ▪  降低复杂度：在中间件中，只有一个ctx，所有信息都在ctx上，使用起来很方便。
    // ▪  便于维护：上下文中的一些必要信息都在ctx上，便于维护。
    // ▪  降低风险：context对象是唯一的，信息是高内聚的，因此改动的风险也会降低很多。

    // 在上下文上挂载一些东西
    const context = Object.create(this.context);
    // 给上下文挂上东西
    const request = (context.request = Object.create(this.request));
    const response = (context.response = Object.create(this.response));
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;

    // 给请求和响应挂上了上下文
    request.ctx = response.ctx = context;

    // 给请求挂上了响应
    request.response = response;
    // 给响应挂上了请求
    response.request = request;

    // 给上下文挂上东西
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};

    // type Content = {
    //   app: this;
    //   req: req;
    //   res: res;
    //   originalUrl:
    //   state：{};
    // }
    return context;
  }

  /**
   * Default error handler.
   *
   * @param {Error} err
   * @api private
   */

  onerror (err) {
    // When dealing with cross-globals a normal `instanceof` check doesn't work properly.
    // See https://github.com/koajs/koa/issues/1466
    // We can probably remove it once jest fixes https://github.com/facebook/jest/issues/2549.

    // 用来兼容 jest 的
    const isNativeError =
      Object.prototype.toString.call(err) === "[object Error]" ||
      err instanceof Error;
    if (!isNativeError)
      throw new TypeError(util.format("non-error thrown: %j", err));

    // 客户端错误就跳过
    if (err.status === 404 || err.expose) return;

    // 显式设置 实例的silent 为 true 的时候, 可以屏蔽服务端的 error 日志
    // 前提是自己没有绑定其他 error 回调
    if (this.silent) return;

    const msg = err.stack || err.toString();
    // 输出错误 日志
    console.error(`\n${msg.replace(/^/gm, "  ")}\n`);
  }

  /**
   * Help TS users comply to CommonJS, ESM, bundler mismatch.
   * @see https://github.com/koajs/koa/issues/1513
   */

  static get default () {
    // 方便 esm ？
    return Application;
  }
};
```
