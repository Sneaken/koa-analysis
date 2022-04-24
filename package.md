# package.json

> [node 使用的一些字段](https://nodejs.org/api/packages.html#packages_exports)
>

## exports

> 这个字段的优先级比 main 高

exports 字段声明了一个对应关系，用 import "package" 和 import "package/sub/path" 会返回不同的模块。

这替换了默认返回 main 字段文件的行为。

当指定了 exports 字段时，只有声明了那些模块是可用的，其他的模块会抛出 ModuleNotFound Error。