# cors.js

> Cross-Origin Resource Sharing(CORS) for koa

## 1. 依赖包

## 2. 源码

```js

'use strict';

const vary = require('vary');

/**
 * CORS middleware
 *
 * @param {Object} [options]
 *  - {String|Function(ctx)} origin `Access-Control-Allow-Origin`, default is request Origin header
 *  - {String|Array} allowMethods `Access-Control-Allow-Methods`, default is 'GET,HEAD,PUT,POST,DELETE,PATCH'
 *  - {String|Array} exposeHeaders `Access-Control-Expose-Headers`
 *  - {String|Array} allowHeaders `Access-Control-Allow-Headers`
 *  - {String|Number} maxAge `Access-Control-Max-Age` in seconds
 *  - {Boolean|Function(ctx)} credentials `Access-Control-Allow-Credentials`
 *  - {Boolean} keepHeadersOnError Add set headers to `err.header` if an error is thrown
 *  - {Boolean} secureContext `Cross-Origin-Opener-Policy` & `Cross-Origin-Embedder-Policy` headers.', default is false
 *    @see https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer/Planned_changes
 *  - {Boolean} privateNetworkAccess handle `Access-Control-Request-Private-Network` request by return `Access-Control-Allow-Private-Network`, default to false
 *    @see https://wicg.github.io/private-network-access/
 * @return {Function} cors middleware
 * @public
 */
module.exports = function (options) {
  const defaults = {
    allowMethods: 'GET,HEAD,PUT,POST,DELETE,PATCH',
    secureContext: false,
  };

  options = {
    ...defaults,
    ...options,
  };

  if (Array.isArray(options.exposeHeaders)) {
    options.exposeHeaders = options.exposeHeaders.join(',');
  }

  if (Array.isArray(options.allowMethods)) {
    options.allowMethods = options.allowMethods.join(',');
  }

  if (Array.isArray(options.allowHeaders)) {
    options.allowHeaders = options.allowHeaders.join(',');
  }

  if (options.maxAge) {
    options.maxAge = String(options.maxAge);
  }

  options.keepHeadersOnError = options.keepHeadersOnError === undefined || !!options.keepHeadersOnError;

  return async function cors (ctx, next) {
    // If the Origin header is not present terminate this set of steps.
    // The request is outside the scope of this specification.
    // Origin 字段 是浏览器在跨域的时候自动添加的
    // for example:
    // Origin: http://example.com
    const requestOrigin = ctx.get('Origin');

    // Always set Vary header
    // https://github.com/rs/cors/issues/10
    ctx.vary('Origin');

    // 表明不是跨域请求的话 就之行下一个中间件
    if (!requestOrigin) return await next();

    let origin;
    if (typeof options.origin === 'function') {
      origin = options.origin(ctx);
      if (origin instanceof Promise) origin = await origin;
      if (!origin) return await next();
    } else {
      origin = options.origin || requestOrigin;
    }

    let credentials;
    if (typeof options.credentials === 'function') {
      credentials = options.credentials(ctx);
      if (credentials instanceof Promise) credentials = await credentials;
    } else {
      credentials = !!options.credentials;
    }

    const headersSet = {};

    function set (key, value) {
      ctx.set(key, value);
      headersSet[key] = value;
    }

    if (ctx.method !== 'OPTIONS') {
      // Simple Cross-Origin Request, Actual Request, and Redirects
      set('Access-Control-Allow-Origin', origin);

      if (credentials === true) {
        set('Access-Control-Allow-Credentials', 'true');
      }

      if (options.exposeHeaders) {
        set('Access-Control-Expose-Headers', options.exposeHeaders);
      }

      if (options.secureContext) {
        set('Cross-Origin-Opener-Policy', 'same-origin');
        set('Cross-Origin-Embedder-Policy', 'require-corp');
      }

      if (!options.keepHeadersOnError) {
        return await next();
      }
      try {
        return await next();
      } catch (err) {
        const errHeadersSet = err.headers || {};
        const varyWithOrigin = vary.append(errHeadersSet.vary || errHeadersSet.Vary || '', 'Origin');
        delete errHeadersSet.Vary;

        err.headers = {
          ...errHeadersSet,
          ...headersSet,
          ...{vary: varyWithOrigin},
        };
        throw err;
      }
    } else {
      // Preflight Request

      // If there is no Access-Control-Request-Method header or if parsing failed,
      // do not set any additional headers and terminate this set of steps.
      // The request is outside the scope of this specification.
      if (!ctx.get('Access-Control-Request-Method')) {
        // this not preflight request, ignore it
        return await next();
      }

      // 这步 是 CORS 的 关键步骤？
      ctx.set('Access-Control-Allow-Origin', origin);

      if (credentials === true) {
        ctx.set('Access-Control-Allow-Credentials', 'true');
      }

      if (options.maxAge) {
        ctx.set('Access-Control-Max-Age', options.maxAge);
      }

      if (options.privateNetworkAccess && ctx.get('Access-Control-Request-Private-Network')) {
        ctx.set('Access-Control-Allow-Private-Network', 'true');
      }

      // 这步 是 CORS 的 关键步骤？
      if (options.allowMethods) {
        ctx.set('Access-Control-Allow-Methods', options.allowMethods);
      }

      if (options.secureContext) {
        // COEP: Cross Origin Embedder Policy：跨源嵌入程序策略
        // COOP: Cross Origin Opener Policy：跨源开放者政策
        // CORP: Cross Origin Resource Policy：跨源资源策略
        // CORS: Cross Origin Resource Sharing：跨源资源共享
        // CORB: Cross Origin Read Blocking：跨源读取阻止

        // 把从该网站打开的其他不同源的窗口隔离在不同的浏览器 Context Group，这样就创建的资源的隔离环境。
        // 如果带有 COOP 的网站打开一个新的跨域弹出页面，则其 window.opener 属性将为 null 。
        set('Cross-Origin-Opener-Policy', 'same-origin');
        // 仅加载明确标记为可共享的跨域资源
        set('Cross-Origin-Embedder-Policy', 'require-corp');
      }

      let allowHeaders = options.allowHeaders;
      if (!allowHeaders) {
        allowHeaders = ctx.get('Access-Control-Request-Headers');
      }
      if (allowHeaders) {
        ctx.set('Access-Control-Allow-Headers', allowHeaders);
      }

      // 204 No Content
      // 此时是没有body的
      ctx.status = 204;
    }
  };
};
```

## 3. how to use

```js
const Koa = require('koa');
const cors = require('@koa/cors');

const app = new Koa();


app.use(cors({
  // 这边可以设置一系列参数的
}));
```