### API
---
如果浏览器访问某个接口，使用koa怎么返回数据给前端?
通过ctx.path和ctx.method,拿到当前访问的链接，再通过ctx.body把数据返回给前端
```
app.use(async (ctx, next)=> {
    console.log(ctx.path)
    console.log(ctx.method)
    if (ctx.path == '/classic/latest' && ctx.method == 'GET') {
        ctx.body = {
            key: 'classic'
        }
    }
})
```
如果有多个api,这样if判断实在太繁琐，可以通过路由来简化
引入路由后还需要3个步骤
* 实例化路由对象
* 编写路由函数，第一个参数是api名称，第二个参数是一个中间件函数
* 注册到应用对象上
```
const Koa = require('koa');
const Router = require('koa-router')

const app = new Koa()
// 实例化
const router = new Router()
// 编写路由函数 第二个参数是一个中间件
router.get('/classic/latest', (ctx, next)=>{
    ctx.body = {
        key: 'classic'
    }
})

// 注册
app.use(router.routes())
app.listen(3000)
```
### 多Router拆分路由
显然一个文件里定义多个路由，后期维护也会很困难，我们可以新建一个文件夹api,因为要兼容线上好几个版本，根据开闭原则（只扩展，不修改），所以不同版本单独建一个目录，里面存放该版本下的所有api,一般会划分多个js文件存放不同的api
`api/v1/book.js`
```
router.get('/v1/book/latest', (ctx, next)=> {
    ctx.body = {
        key: 'book'
    }
})
```
但是怎么与app.js（父）建立联系呢
* 引入router,创建实例，创建路由函数，并且export出去
```
const Router = require('koa-router')
const router = new Router()
router.get('/v1/book/latest', (ctx, next)=> {
    ctx.body = {
        key: 'book'
    }
})
module.exports = router
```
* app.js中，引入这个api的js,并注册
```
const Koa = require('koa');
const book = require('./api/v1/book')
const classic = require('./api/v1/classic')
const app = new Koa()
// 注册
app.use(book.routes())
app.use(classic.routes())
```
*一个项目里只能有一个router吗*
* 当然不是，每个api文件里都可以实例化一个router对象

### nodemon
`npm i nodemon -g`
`nodemon app.js`运行这个js

### requireDirectory实现路由自动加载
当接口越来越多的时候，每新增一个接口，`app.js`中都要引入这个文件并注册，实在太麻烦，可以通过require-directory来实现路由自动加载
```
const Koa = require('koa');
const requireDirectory = require('require-directory')
const Router = require('koa-router')

const app = new Koa()
requireDirectory(module, './api', {visit: whenLoadModule})
function whenLoadModule (obj) {
    if (obj instanceof Router) {
        console.log('haha')
        app.use(obj.routes())
    }
}
```
引入`require-directory`,引入一个目录下所有的文件，代码中引入的是`api`这个目录，requireDirectory提供了第三个参数，`visit`可以是一个函数，这个函数的参数就是引入的这个模块，在这个函数里就可以对这个模块做相应的操作，这里是对这个模块做了注册
*多思考，精简代码*

### 初始化管理器与process.cwd()
app.js中最好不要放过多的代码，所以我们对目录结构进行调整，新建`app`目录，将`api`放入`app`中,新建`core`目录，这个目录放着工具方法，将路由自动加载的功能放在这个目录下，新建`init.js`
```
const requireDirectory = require('require-directory')
const Router = require('koa-router')
class InitManger {
    static initCore(app){
        // 入口方法
        // InitManger.app = app
        InitManger.initLoadRouters(app)
    }
    static initLoadRouters(app){
        const apiDirectory = `${process.cwd()}/app/api`
        requireDirectory(module, apiDirectory, {visit: whenLoadModule})

        function whenLoadModule (obj) {
            if (obj instanceof Router) {
                console.log('haha')
                // InitManger.app.use(obj.routes())
                app.use(obj.routes())
            }
        }
    }
}
module.exports = InitManger
```

在`app.js`中
```
const Koa = require('koa');
const InitManger = require('./core/init')

const app = new Koa()

InitManger.initCore(app)

app.listen(3000)

```
