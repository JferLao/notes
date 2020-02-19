# Node.js的能力与应用
1. 脱离浏览器运行js
2. NodeJs Stream
3. 服务端API
4. 作为中间层

# JS包导入方式
1. commonJs(require)
2. import from(es6)  <!-- 实验特性-->
3. AMD

# await意义
1. 求值关键字await可以对表达式或promise进行求值
2. await可以阻塞线程(对资源的请求都是异步的:读文件/发送http/操作数据库)
3. 保证中间件按洋葱模型执行的先决条件是中间件中next前需要await
4. 保证ctx上下文对象的获取

# KOA2程序
```
// 导入koa
const Koa = require('koa')

// 实例化koa,应用程序对象包含很多中间件
const app = new Koa()

// 发送HTTP KOA接收HTTP
// 使用中间件
// 函数要变成中间件那就需要把函数注册到程序对象上

// 一个应用程序可以注册多个中间件

// ctx参数:上下文对象
// next参数:下一个中间件的函数
// 在中间件函数前要加入async,在next()前加await,强制执行异步操作,从而保证洋葱模型
app.use(async(ctx, next) => {
        // next()执行下一个中间件,如果next()方法后还有函数则会先执行next()再执行后面的方法
        const a = await next();
        // next调用的返回结果是promise
        // 通过then回调获取promise的数据
    })
    // koa只能自动执行第一个中间件,后续需要自己定义执行




// 启动程序
app.listen(3000)
```

# 路由拆分
1. 避免引用循环(NodeJs不会报错)
2. 新建api文件夹保存路由js文件
3. api携带版本号,因为需要兼顾客户端兼容性:A.加载在路径 B.加载在查询参数 C.header里
4. 在对应路有文件中引入koa-router,并且实例化一个路由对象,再对路由对象配置最后导出模块
5.最后在入口文件导入该文件,并使用中间件注册

# 实现路由自动加载
1. 导入require-directory 
2. 设置引入模块的路径const modules = requireDirectory(module, './api',{visit:whenLoadModule})
3. 第一个参数是固定参数module,第二个参数是导入哪个目录下面的路径,第三个参数是一个对象,有一个visit属性,每当加载一个模块则会加载这个回调函数
4. 通过循环遍历进行路由注册for(let r in modules){app.use(classic.routes())}	
5. 需要判断r是不是路由,只能导入路由进行注册
6. 也可以通过第三个函数的回调函数进行判断和遍历 
```
function whenLoadModule(obj){
	if(obj instanceof Router){
		app.use(obj.routes())
	}
}
```

# 抽离路由代码
1. 新建一个文件,然后把要用到的模块先导入.
2. 设置有个InitManager类,把要用到的方法设置成静态方法,最后导出出去
3. 最后在入口文件使用
```
<!-- init.js -->
const requireDirectory = require('require-directory')
const Router = require('koa-router')
class InitManager {
    static initCore(app) {
        // 入口方法
        // 把app保存在类上
        InitManager.app = app

        // 在类里面调用静态方法
        InitManager.initLoadRouters()
    }
    static initLoadRouters() {
        // 使用绝对路径
        const apiDirectory = `${process.cwd()}/app/api`
        requireDirectory(module, apiDirectory, { visit: whenLoadModule })

        // 加载路由时的回调
        function whenLoadModule(obj) {
            if (obj instanceof Router) {
                InitManager.app.use(obj.routes())
            }
        }
    }
}
// 导出
module.exports = InitManager

<!-- app.js -->
//导入
const InitManager = require('./core/init')
// 把路有方法分离到外面的文件
InitManager.initCore(app)
```
# KOA接收前端传参
1. 3种常见的传递方式:A.header传参 B:url{param}变量传参 C.参数拼接传参
2. 获取方式
```
const path = ctx.params				//获取路径的参数
const query = ctx.request.query		//获取?后的参数
const headers = ctx.request.header	//获取header
const body = ctx.request.body		//需要在app.js引入和注册body-parser来获取请求体
```

# 异常处理
1. 函数设计判断异常有两种方法:一.return false或者null 二.throw新的异常然后进行捕获
2. 不建议return false/null 因为会丢失错误信息
3. 实际情况不想try-catch的方法:A.确保代码完全无误 B.全局异常处理
4. 全局异常处理机制:在所有函数调用的方法里侦测错误然后捕获
5. try-catch一般只对同步的操作有效,对异步操作可能捕获不了错误
6. 在函数调用链里有异步操作,其他方法最好也要用异步await
7. 通过koa中间件进行全局异常处理:当接口抛出异常,则需要在全局监听到错误然后输出一段有意义的提示信息
8. 在函数调用链的外层做一个中间件形式的异常处理,集中把所有异常处理的逻辑都放在一起
```
const catchError = async(ctx, next) => {
    try {
        await next()
    } catch (error) {
        // 异常处理的逻辑代码
        ctx.body = '服务器出现问题'
    }
}
module.exports = catchError
```
9. 程序捕捉到的error不能直接返回到客户端去,需要简化清晰明了的信息
10. 已知错误类型和未知错误类型处理手段不同,已知异常的error有error_code属性
11. 错误的message/error_code/request_url可以携带在error中然后抛出异常,然后全局捕捉异常处可以通过error对象进行获取对应内容
12. 封装error信息然后再在全局捕获
```
// 继承内置对象error
<!-- http-exception.js -->
class HttpException extends Error {
    constructor(msg = '服务器异常', errorCode = 10000, code = 400) {
        super()
        this.errorCode = errorCode
        this.code = code
        this.msg = msg
    }
}
module.exports = {
    HttpException
}

<!-- exception.js -->
const { HttpException } = require('../core/http-exception')
const catchError = async(ctx, next) => {
    try {
        await next()
    } catch (error) {
        if (error instanceof HttpException) {
            ctx.body = {
                    msg: error.msg,
                    error_code: error.errorCode,
                    request: `${ctx.method} ${ctx.path}`,
                }
                // 一次请求的http结果可以通过ctx的status属性保存
            ctx.status = error.code
        }

    }
}
module.exports = catchError
```
13. 可以定义多种错误类型信息,然后通过继承HttpException的一些信息.提高复用性
```
class ParameterException extends HttpException {
    constructor(msg, errorCode) {
        super()
        this.code = 400
        this.msg = msg || '参数错误'
        this.errorCode = errorCode || 10000
    }
}

<!-- 调用 -->
const error = new ParameterException()
```
14. 每一次使用Exception而不想导入异常文件可以把异常处理保存在global上然后调用全局global属性
```
// 把所有异常都装载在global下的errs内,然后通过new global.errs.ParameterException()
    static loadHttpException() {
        const errors = require('./http-exception')
        global.errs = errors
    }
```

# 生产环境和开发环境
1. 新建一个文件进行文件配置
2. 把设置保存在global变量上
3. 进行全局初始化
```
static loadConfig(path = '') {
        const configPath = path || process.cwd() + '/config/config.js'
        const config = require(configPath)
        global.config = config
}
```
4. 使用是只需要判断global.config.environment即可

# 用户系统相关
1. 一般用户有账号/密码还有相关附属信息
2. 相关操作一般有登录和注册
3. 一般流程:把用户信息通过http请求发送到koa2的api中,业务的逻辑放在model层进行处理,然后把处理完后的内容存在数据库中,最后把请求结果返回到客户端

# KOA创建数据库表
1. 使用sequelize连接数据库
2. 导入到项目中,创建对应的表继承Model
3. 使用MOdel表继承的方法init()进行数据变量形式初始化
```
class User extends Model {}
User.init({
    // id不显示注册,会自动生成主键
    id: {
        type: Sequelize.INTEGER,
        primaryKey: true,
        autoIncrement: true
    },
    nickname: Sequelize.STRING,
    email: Sequelize.STRING,
    password: Sequelize.STRING,
    
})
```

# 使用LinValidator进行参数验证(以用户为例)
1. 引入文件(这里是直接文件复制到项目目录上)
```
const { LinValidator, Rule } = require("../../core/lin-validator");
```
2. 配置校验规则
```
<!-- 继承校验 -->
class RegisterValidator extends LinValidator {
    constructor() {
        super()
		<!-- 新建Rule,第一个参数为内置校验模式,第二个参数为提示文字,第三个为相关额外校验对象参数设置 -->
        this.email = [
                new Rule('isEmail', '电子邮箱不符合规范，请输入正确的邮箱')
            ],
            this.password1 = [
                new Rule('isLength', '密码至少6个字符，最多32个字符', {
                    min: 6,
                    max: 32
                }),
                new Rule('matches', '密码不符合规范', "^(?![0-9]+$)(?![a-z]+$)(?![A-Z]+$)(?![,\.#%'\+\*\-:;^_`]+$)[,\.#%'\+\*\-:;^_`0-9A-Za-z]{6,20}$")
            ],
            this.password2 = this.password1,
            this.nickname = [
                new Rule('isLength', '昵称不符合长度规范', {
                    min: 4,
                    max: 32
                }),
            ]
    }
	<!-- 对于一些规则字段可以用方法进行配置,有错误则直接抛出异常即可 -->
    validatePassword(vals) {
        const password1 = vals.body.password1
        const password2 = vals.body.password2
        if (password1 !== password2) {
            throw new Error('两个密码不一致')
        }
    }
}
module.exports = {
    RegisterValidator
}
```
3. 使用则导入该模块中的对应类,然后实例化并且把ctx作为对象传给校验器即可
```
const v = new RegisterValidator().validate(ctx)
```

# 密码加密存储到数据库
1. 在model层使用bcrypt模块并且引入
2. 设置加密密码的规则和成本
3. 最后使用scrypt的hashSync进行异步加密
```
const bcrypt = require('bcryptjs')

<!-- User.init -->
password: {
        type: Sequelize.STRING,
        set(val) {
            const salt = bcrypt.genSaltSync(10)
            const psw = bcrypt.hashSync(v.get('body.password2'), salt)
            this.setDataValue(psw)
        }

    },
```

# 用sequelize操作把数据存到数据库
1. sequelize提供了一个便捷接口实现数据导入数据库,只需要把数据在model层定义好,然后在接口处理的层将数据写成一个js对象,然后使用create()方法即可
```
const user = {
        email: v.get('body.email'),
        password: v.get('body.password2'),
        nickname: v.get('body.nickname')
    }
await User.create(user)
```

# 返回成功的处理结果
1. 成功返回也可以看做类似异常一样抛出即可,所以在exception的地方新建一个Success类处理返回的结果
2. 编写一个抛出Success的方法并导出
3. 在所需要的模块导入并且引用方法即可
```
<!-- http-exception.js -->
class Success extends HttpException {
    constructor(msg, errorCode) {
        super()
        this.code = 201
        this.msg = msg || 'ok'
        this.errorCode = errorCode || 0
    }
}

<!-- success.js -->
const success = (msg, errorCode) => {
  throw new global.errs.Success(msg, errorCode)
}

module.exports = {
  success
}
<!-- 使用 -->
const { success } = require('success')
success()//即可
```

# 验证账号密码
1. 首先在校验器里编写验证token对应的账号和密码规则
2. 并且在校验器里加入登录类型的方法判断
```
class TokenValidator extends LinValidator {
    constructor() {
        super()
        this.account = [
                new Rule('isLength', '账号长度不符合规范', {
                    min: 4,
                    max: 32
                })
            ],
            this.secret = [
                new Rule('isOptional'),
                new Rule('isLength', '至少6个字符', {
                    min: 6,
                    max: 128
                })
            ]
    }

    validateLoginType(vals) {
        if (!vals.body.type) {
            throw new Error('type是必传参数')
        }
        if (!LoginType.isThisType(vals.body.type)) {
            throw new Error('type参数不合法')
        }
    }
}
```
3. 创建接口,将ctx传到校验器内
```
 const v = await new TokenValidator().validate(ctx)
 ```
 4. 将获取到的http判断登录类型进行对应的登录方式的方法操作
 5. 将判断账号密码和数据库比对的步骤放在User类里进行操作
 6. 新建实例方法,传入账号密码,然后根据参数在数据库里寻找一次对应数据
 7. 再判断数据库的数据和接收到的数据有没有错误异常,有则抛出无则返回搜索结果
```
static async verifyEmailPassword(email, plainPassword) {
        const user = await User.findOne({
            where: {
                email
            }
        })
        if (!user) {
            throw new global.errs.NotFound('用户不存在')
        }
        // 利用bcrypt解析密码
        const correct = bcrypt.compareSync(plainPassword, user.password)
        if (!correct) {
            throw new global.errs.AuthFailed('密码不正确')
        }
        return user
    }
```
8. 账户密码验证成功返回jwt令牌


# 不公开接口需要token验证(通过中间件形式处理)
1. 先设置好异常信息
```
class Forbbiden extends HttpException {
    constructor(msg, errorCode) {
        super()
        this.code = 403
        this.msg = msg || '禁止访问'
        this.errorCode = errorCode || 10006
    }
}
```
2. 新建一个权限管理的Auth类,进行token控制,并且在类中设置方法get m() 来控制token
3. 用basic-auth模块的basicAuth()方法解析ctx的请求获取ctx.req
4. 如果请求不带token则直接抛出异常
5. 带了token则进行try-catch处理,yongjsonwebtoken的verfgy()方法解析token
6. 如果有异常则抛出异常,比如不合法和token时间已过期
7. 没有异常可以把token设置时的自定义属性保存在ctx.auth里
8. 最后执行next()方法执行下一个中间件


# 编写小程序的登录接口
1. 小程序登录需要前端传过来code码,然后调用微信的接口(微信平台获取)生成openid唯一标识
2. 微信的接口需要3个参数code,appid和appsecret(code需要作为参数传入,appid和appsecret需要申请)
3. 编写一个WXManage类,把参数拼接url地址然后通过axios请求发送出去即可
```
const url = util.format(global.config.wx.loginUrl, global.config.wx.appId, global.config.wx.appSecret, code)
const result = await axios.get(url)
```
4. 需要对openid进行错误解析(每种错误都会返回不同的result,可以用来进行判断)
5. 因为安全问题,存储在数据库中使用时不能直接使用openid,而是用返回的token即可
```
 static async getUserByOpenid(openid) {
        const user = await User.findOne({
            where: {
                openid
            }
        })

        return user
    }

    static async registerByOpenid(openid) {
        return await User.create({
            openid
        })
    }
```
6. 需要在USER表的model层查询openid是否存在,如果不存在则存入,存在则直接将openid和权限作为参数生成token即可
```
class WXManage {
    static async codeToToken(code) {
        const url = util.format(global.config.wx.loginUrl, global.config.wx.appId, global.config.wx.appSecret, code)
        const result = await axios.get(url)
            // 判断
        if (result.status !== 200) {
            throw new global.errs.AuthFailed('openid获取失败')
        }
        const errcode = result.data.errcode
        const errmsg = result.data.errmsg
        if (errcode) {
            throw new global.errs.AuthFailed('openid获取失败:' + errmsg)
        }

        // 根据openid写入用户到数据库的user表内
        // 在model层写上查询数据库的方法
        const user = await User.getUserByOpenid(result.data.id)
        if (!user) {
            user = await User.getUserByOpenid(result.data.id)
        }
        return generateToken(user.id, Auth.USER)
    }
}
```

# 在小程序请求中携带令牌
1. 在请求头中把token存在Authorization中发送
2. Authorization键值为base64的值所以需要额外的方法进行转换
```
import {Base64} from 'js-base64'	//需要引入js-base64的npm包

_encode(){
	const token=wx.getStorageSync('token')	//从缓存中获取token
	const base64=Base64.encode(token+':')
	
	return 'Basic'+ base64
}
```

**或者直接写编码方法**

```
base64_encode(str) { // 编码，配合encodeURIComponent使用
    var c1, c2, c3;
    var base64EncodeChars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=";
    var i = 0, len = str.length, strin = '';
    while (i < len) {
      c1 = str.charCodeAt(i++) & 0xff;
      if (i == len) {
        strin += base64EncodeChars.charAt(c1 >> 2);
        strin += base64EncodeChars.charAt((c1 & 0x3) << 4);
        strin += "==";
        break;
      }
      c2 = str.charCodeAt(i++);
      if (i == len) {
        strin += base64EncodeChars.charAt(c1 >> 2);
        strin += base64EncodeChars.charAt(((c1 & 0x3) << 4) | ((c2 & 0xF0) >> 4));
        strin += base64EncodeChars.charAt((c2 & 0xF) << 2);
        strin += "=";
        break;
      }
      c3 = str.charCodeAt(i++);
      strin += base64EncodeChars.charAt(c1 >> 2);
      strin += base64EncodeChars.charAt(((c1 & 0x3) << 4) | ((c2 & 0xF0) >> 4));
      strin += base64EncodeChars.charAt(((c2 & 0xF) << 2) | ((c3 & 0xC0) >> 6));
      strin += base64EncodeChars.charAt(c3 & 0x3F)
    }
    return strin
  },
  _encode() {
    const token = wx.getStorageSync('token')	//从缓存中获取token
    const base64 = this.base64_encode(token + ':')
    return 'Basic' + ' ' + base64
  }
```

3. 请求头直接调用方法即可
```
wx.request({
	header:{
		Authorization:this._encode()
	}
})
```

# 避免循环查询数据库
1. in查询避免循环查询数据库,因为循环查询数据库的次数不可控,有风险

```
const { Op } = require('sequelize')
const { flatten } = require('lodash')
static async getList(artInfoList) {
            // 每种类型放置
            const artInfoObj = {
                100: [],
                200: [],
                300: []
            }
	    //先遍历分类
            for (let artInfo of artInfoList) {
                artInfoObj[artInfo.type].push(artInfo.art_id)
            }
            const arts = []
	    //然后进行3次的遍历
            for (let key in artInfoObj) {
                const ids = artInfoObj[key]
                if (!ids.length) {
                    continue
                }
                key = parseInt(key)
                arts.push(await Art._getListByType(ids, key))	//处理数据库查询的外部方法
            }
            return flatten(arts)
        }
```

# 避免循环导入
1. A中导入B,B中导入A,则会发生错误
2. 可以通过在方法内导入

# 中间层和微服务
1. 对于大型项目,业务复杂,如果把所有业务都放在后端难以维护.
2. 把业务拆分成各种服务可以方便开发
3. 后端负责数据整合,业务逻辑
4. 服务负责提供数据
5. 中间层可以负责后端部分,整合前端成为大前端


# 全局控制Model模型JSON序列化
1. 把ToJSON方法加载在Model类上,直接通过Model的原型链上添加方法toJSON()
```
Model.prototype.toJSON = function() {
```

2. 使用lodash的unset方法移除不想显示的数据
```
let data = clone(this.dataValues)
unset(data, 'updatedAt')
return data
```

# KOA图片静态资源
1. 在根目录创建一个static文件夹,在static文件夹里放置一个images文件夹存放图片
2. 在入口文件app.js使用koa-static,并设置路径地址
```
const static = require('koa-static')
const path = require('path')

app.use(static(path.join(__dirname, './static')))
```
3. 在model层图片处理时拼接完整地址
```
if (art && art.image) {
    let imgUrl = art.dataValue = image
    art.dataValue.image = global.config.host + imgUrl
}
```
