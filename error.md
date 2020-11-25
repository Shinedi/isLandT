### 接口参数的携带和获取
* 链接： 参数前加`:`,例如`/v1/:id/classic/latest`
* 链接后加参数
* header
* body(发送post请求时，参数会带在body里)
参数获取：
```
router.post('/v1/:id/classic/latest', (ctx, next)=>{
    const path = ctx.params; // 获取链接里的参数
    const query = ctx.request.query; // 获取链接后带的参数
    const headers = ctx.request.header; // 获取header
    const body = ctx.request.body; // // 获取body
    console.log('path', JSON.stringify(path))
    console.log('query',JSON.stringify(query))
    console.log('headers',JSON.stringify(headers))
    console.log('body',JSON.stringify(body))
    ctx.body = {
        key: 'classic1'
    }
})
```
### 异步异常处理
---
调用异步函数时，前面加await,await可以对表达式求值
将await包裹在try...catch中
* 全局异常处理中间件编写
新建`middlewares/exceptionns.js`
```
const catchError = async (ctx, next) => {
    try {
        await next()
    }catch (error) {
        ctx.body = '服务器有点问题，稍等一下'
    }
}
module.exports = catchError
```
`app.js`中,引入并注册这个异常处理函数
```
const Koa = require('koa');
const parser = require('koa-bodyparser')
const InitManger = require('./core/init')
const catchError = require('./middlewares/exception.js')

const app = new Koa()
app.use(parser())
app.use(catchError)
InitManger.initCore(app)
```
错误监听步骤：
* 监听错误
* 输出一段有意义的信息（message/error_code/request_url）
// 已知型错误: 处理错误,明确/try...catch
// 未知型错误: try...catch

### 定义异常返回格式
如何快速定位哪里的调用出现异常，可以在出现异常的时候返回`message、errorCode、requestUrl(当前调用接口)、status`,在全局异常处理中间件中赋值给ctx.body *status可以直接赋值给`ctx.status`*
```
const error = new Error('我出错了')
error.errorCode = 10001
error.status = 400
error.requestUrl = `${ctx.method} ${ctx.path}`
throw error
```
`middlewares/exception.js`
```
try {
    await next()
}catch(error) {
    // 监听错误，输出一段有意义的信息
    if (error.errorCode) {
        ctx.body = {
            msg: error.message,
            error_code: error.errorCode,
            request: error.requestUrl
        }
        ctx.status = error.status
    }

    
}
```
我们发现每个接口处理异常都要写一大堆代码来进行错误信息描述，有点繁琐，况且错误信息结构都是相同的，所以我们可以定义一个错误对象

### 定义http-exception异常基类
---
在`core/http-exception.js`中定义这个类
```
class HttpException extends Error {
    constructor(msg='服务器异常', errorCode = 10000 , code = 400) {
        super()
        this.errorCode = errorCode;
        this.msg = msg;
        this.code = code;
    }
}

module.exports = {
    HttpException
}
```
`app/api/v1/classic.js`中
```
const Router = require('koa-router')
const {HttpException} = require('../../../core/http-exception')
const router = new Router()
router.post('/v1/:id/classic/latest', (ctx, next)=>{
    const path = ctx.params;
    const query = ctx.request.query;
    const headers = ctx.request.header;
    const body = ctx.request.body;
    console.log(query)
    if (true) {
        // const error = new Error('我出错了')
        // error.errorCode = 10001
        // error.status = 400
        // error.requestUrl = `${ctx.method} ${ctx.path}`
        throw new HttpException('我出错了', 10001, 400)
    }
    ctx.body = {
        key: 'classic1'
    }
    throw new Error('出错了')
})

module.exports = router

```
相对简化了好多,这里不需要再定义`requestUrl`,在全局异常处理函数里进行处理就可以
`middlewares/exception.js`中进行修改
```
const {HttpException} = require('../core/http-exception')
const cacheError = async (ctx, next) => {
    try {
        await next()
    }catch(error) {
        // 监听错误，输出一段有意义的信息
        if (error instanceof HttpException) {
            ctx.body = {
                msg: error.msg,
                error_code: error.errorCode,
                request: `${ctx.method} ${ctx.path}`
            }
            ctx.status = error.code
        }
    
        
    }
}
module.exports = cacheError
```
这里的判断条件修改为`error instanceof HttpException`,并且获取`request`

### 特定异常类
出现某个异常时，提示文案、code都相同，不必每次都要写这些文案，可以创造一个特定异常类
```
class parameterException extends HttpException {
    constructor(msg = '参数错误', errorCode = 10000) {
        super();
        this.code = 400;
        this.msg = msg;
        this.errorCode = errorCode;
    }
}
```
以后出现参数错误的异常时，可以引入这个类，new就可以了`new parameterException()`

       