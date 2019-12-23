#一、了解mockjs
前言：mockjs是什么

生成随机数据，拦截 Ajax 请求。

通过随机数据，模拟各种场景；不需要修改既有代码，就可以拦截 Ajax 请求，返回模拟的响应数据；支持生成随机的文本、数字、布尔值、日期、邮箱、链接、图片、颜色等；支持支持扩展更多数据类型，支持自定义函数和正则。

优点是非常简单方便, 无侵入性, 基本覆盖常用的接口数据类型.
####1.安装

```
npm install mockjs
```
####2.引用
```
var Mock = require('mockjs')    //普通引用方式

import Mock from 'mockjs'      //在vue中可以这样引用
```
####3.使用
```
testMockJs(){
    let data = Mock.mock({ 
        'list|1-10':[{ 'id|+1':1,
        'engName|2-4':'Hello',
        'chinaName|5':'Chinese',
        'number|1-6':3,
        'aNumber|1-6.5':4,
        'first: '@FIRST',    //指随机生成英语中的first name
        middle: '@FIRST',
        last: '@LAST',      //指随机生成英语中的last name
        full: '@first @middle @last' }]  //@+变量名，指引用该变量的属性值
    })
        console.log(`data:${JSON.stringify(data, null, 4)}`)
}
```
生成的结果如下：

![生成随机json数据](https://upload-images.jianshu.io/upload_images/13541244-8591909d7ad643ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####4.语法规范
4.1数据模板定义规范（Data Template Definition，DTD）

**数据模板中的属性构成：“name|rule”:value**

(1).属性值是字符串String

**'name|min-max': string**

通过重复 string 生成一个字符串，重复次数大于等于 min，小于等于 max。

**'name|count': string**

通过重复 string 生成一个字符串，重复次数等于 count。

'engName|2-4':'Hello',

'chinaName|5':'Chinese',

![生成规则1](https://upload-images.jianshu.io/upload_images/13541244-df9abe4b0670dbc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（2）属性值是数字Number

**'name|+1': number**

属性值自动加 1，初始值为 number。

**'name|min-max': number**

生成一个大于等于 min、小于等于 max 的整数，属性值 number 只是用来确定类型。

**'name|min-max.dmin-dmax': number**

生成一个浮点数，整数部分大于等于 min、小于等于 max，小数部分保留 dmin 到 dmax 位。
```
data = Mock.mock({
                'list|1-10':[{
                'number1|1-100.1-10': 1,
                'number2|123.1-10': 1,
                'number3|123.3': 1,
                'number4|123.10': 1.123 }]
            })  
```

===>生成结果：

```
{ "list": [ 
                { "number1": 2.73803408, "number2": 123.46, "number3": 123.748, "number4": 123.1237616335 },
                { "number1": 67.0562188234,"number2": 123.5817686348,"number3": 123.541,"number4": 123.1238281167 }]
            }
```

（3）属性值是布尔型Boolean

**'name|1': boolean**

随机生成一个布尔值，值为 true 的概率是 1/2，值为 false 的概率同样是 1/2。

**'name|min-max': value**

随机生成一个布尔值，值为 value 的概率是 min / (min + max)，值为 !value 的概率是 max / (min + max)。

```
data = Mock.mock({
                'list|1-10':[{
                'boolean1|1': false,
                'boolean2|6-5': true }]
                })
```

===》生成结果：

```
data:{ "list": [ 
                { "boolean1": true, "boolean2": false }, 
                { "boolean1": true, "boolean2": true }, 
                { "boolean1": true, "boolean2": false }]
            }
```

(4)属性值是对象Object

**'name|count': object**

从属性值 object 中随机选取 count 个属性。

**'name|min-max': object**

从属性值 object 中随机选取 min 到 max 个属性。
```
const mockdata = Mock.mock({
        'people|3':{name:'Marry',age:14,gender:'girl',eat:'apple'},
        'peoples|1-2':{name:'Marry',age:14,gender:'girl',eat:'apple'}
      })
```
===》生成结果
![image.png](https://upload-images.jianshu.io/upload_images/13541244-306d615522225b9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 5.方法

5.1Mock.mock()

**rurl** :可选。表示需要拦截的 URL，可以是 URL 字符串或 URL 正则。
例如: /\/domain\/list\.json/、'/domian/list.json'。

**rtype**：可选。表示需要拦截的 Ajax 请求类型。
例如 GET、POST、PUT、DELETE 等。

**template**：可选。表示数据模板，可以是对象或字符串。
例如 { 'data|1-10':[{}] }、'@EMAIL'。

**function(options)**：可选。表示用于生成响应数据的函数。

(1)**Mock.mock(template)**

根据模板生成模拟数据

(2)**Mock.mock( rurl, template )**

记录数据模板。当拦截到匹配 rurl 的 Ajax 请求时，将根据数据模板 template 生成模拟数据，并作为响应数据返回。

(3)**Mock.mock( rurl, function( options ) )**

记录用于生成响应数据的函数。当拦截到匹配 rurl 的 Ajax 请求时，函数 function(options) 将被执行，并把执行结果作为响应数据返回。

(4)**Mock.mock( rurl, rtype, template )**

记录数据模板。当拦截到匹配 rurl 和 rtype 的 Ajax 请求时，将根据数据模板 template 生成模拟数据，并作为响应数据返回。

(5)**Mock.mock( rurl, rtype, function( options ) )**

记录用于生成响应数据的函数。当拦截到匹配 rurl 和 rtype 的 Ajax 请求时，函数 function(options) 将被执行，并把执行结果作为响应数据返回。

5.2 Mock.setup()

配置拦截 Ajax 请求时的行为。支持的配置项有：timeout。

5.3 Mock.Random()

是一个工具类，用于生成各种随机数据。(在数据模板中成为占位符，书写格式为**@占位符(参数 [, 参数])**)

5.4 Mock.valid()

校验真实数据 data 是否与数据模板 template 匹配。

5.5 Mock.toJSONSchema()

把 Mock.js 风格的数据模板 template 转换成[JSON Schema](http://json-schema.org/)。

# 二、在polymer项目中

前言：因为在polymer中，js文件不能直接引用外部文件，所以我们需要新建一个html,把我们需要引用的文件放在HTML中使用。

1.安装mockjs

```
npm install -g bower
bower install --save mockjs
```

2.在项目中新建mockData文件夹

![mockData文件夹](https://upload-images.jianshu.io/upload_images/13541244-9d6b0aebdf33f105.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.mockData.js中定义模拟数据以及需要拦截的接口

```
(function () {
  const userInfo = Mock.mock({
    "rows|10":[{ //"name|number":[]生成number个数组
      "id|+1":1, //"name|+number":number 生成的每个id比前面一个大number
      "name":"@name()",//"@name()"随机生成name
      "email": "@email()", //"@email"随机生成邮箱
      "groupId|35-47": 35, //"groupId|35-47"随机生成35-47中的某个整数
      "setGroupId|1": true,//"setGroupId|1"随机生成boolean值
      "setId|1": true,
      "setName|1": true,
      "setSort|1": true,
      "setUrl|1": true,
      "sort|1-10": 1,
      "url": "@url()" //"@url()"随机生成url
    }]
  })

  //定义你需要拦截的接口对应的方法,可重写拦截后返回的操作方法
  Mock.mock(RegExp(getMockUrl('/management/link/groupsList.do') + ".*"),'get',function (options) { //get请求使用正则匹配是为了拦截带参数的get请求
    return userInfo
  })
  Mock.mock(getMockUrl('/management/link/add.do'),'post',function (options) {
    addMockData(options,userInfo)
  })
  Mock.mock(getMockUrl('/management/link/update.do'),'post',function (options) {
    updateMockData(options,userInfo)
  })
  Mock.mock(getMockUrl('/management/link/delete.do'),'delete',function (options) {
    deleteMockData(options,userInfo)
  })
})()
//把url统一改成'/mock'+url
function getMockUrl(url){
  return `/mock${url}`
}
//把请求的参数转换成js
function getOptions(options) {
  return JSON.parse(options.body)
}
//add method
function addMockData(options,mockData) {
  const obj = this.getOptions(options)
  obj.id = mockData.rows.length+1
  mockData.rows.push(obj)
}
//update method
function updateMockData(options,mockData) {
  const obj = this.getOptions(options)
  mockData.rows = mockData.rows.map(item=>{
    return item.id === obj.id?obj:item
  })
}
//delete method
function deleteMockData(options,mockData) {
  const id = parseInt(this.getOptions(options).id)
  mockData.rows = mockData.rows.filter(item=>{
    return item.id !== id
  })
}
```

4.在mockData.html中引用mockjs以及mockData.js两个脚本文件

```
<script src="../../../bower_components2/mockjs/dist/mock.js"></script>
<script src="./mock-data.js"></script>
```

5.需要使用模拟数据的页面，引用mockData.html

![image](https://upload-images.jianshu.io/upload_images/13541244-db20e6a7a516fd21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在已有的项目中加入mockjs框架，已封装http请求接口，且不影响原有的使用。

![image](https://upload-images.jianshu.io/upload_images/13541244-40cbae3116f762b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在该封装接口的脚本中引用mockData.html,在原来的基础上加一个参数，判断是否使用模拟数据.如果是，在url上做文章。这样可以保证不会影响到其他模块的使用。

# 三、在vue项目中

1.先在vue工程文件中下载mockjs

一般是：

```
npm install mockjs
```

2.在src中创建 mock.js,在mock.js中导入mockjs,即可使用Mock的各种方法

```
const Mock = require('mockjs')     //导入mockjs
            const userInfo =Mock.mock({       //生成随机数
                'data|10':[
                    { 'id|+1':1,
                    name:'@ctitle(2,10)',
                    avatar:'@image(\'600x600\',@color)',
                    'gender|1':true,
                    'age|18-25':20 }]
                 })
Mock.mock('/api/userInfo/queryUser','get',function () { return userInfo.data}) //获取模拟
数据的接口
```

3.在main.js中导入刚刚新建的mock.js

```
require('./mock.js')
```

至此，我们就可以访问我们定义的模拟数据接口了。

以下是鄙人自己写的实现数据增删改查接口：

```
import Mock from 'mockjs'
export default {
  mockData(){
    const userInfo = Mock.mock(
      {
        'data|10':[{
          'id|+1':1,
          name:'@ctitle(2,10)',
          avatar:'@image(\'600x600\',@color)',
          'gender|1':true,
          'age|18-25':20
        }]
      }
    )
//模拟删除数据
    Mock.mock('/api/userInfo/deleteUser','delete',function (options) {
      const id = parseInt(JSON.parse(options.body).id)
      userInfo.data = userInfo.data.filter(item=>{
        return item.id !== id
      })
      return userInfo.data
    })
//模拟查询数据
    Mock.mock('/api/web/itemPrice/findFinalPriceOfferPage','get',function (options) {
      return userInfo.data
    })
//模拟更新数据
    Mock.mock('/api/userInfo/updateUser','post',function (options) {
      const obj = JSON.parse(options.body)
      userInfo.data = userInfo.data.map(item=>{
        return item.id === obj.id?obj:item
      })
      return userInfo.data
    })
//模拟添加数据
    Mock.mock('/api/userInfo/addUser','post',function (options) {
      const obj = JSON.parse(options.body)
      obj.id = userInfo.data.length+1
      userInfo.data.push(obj)
      return userInfo.data
    })
  }
}
```

在vue.config.js中设置代理服务，即可以测试接口是否被拦截

```
devServer: {
    proxy: {
      '/api': {
        target: 'http://localhost:9094', // 请求的接口
        changeOrigin: true,
        ws: true,
        pathRewrite: {
          '^/api/': ''
        }
      }
    }
  }
```

在我们的实际项目中，很多时候都是把get post请求封装好，以不用多次自己手写ajax或者axios请求。所以我们可以这样做：

在封装请求的方法里加一个isMock判断是否使用模拟数据，是的话改变它的url，如：

```
get(url,params={},isMock){
    url = isMock?`/mock${url}`:url
    return new Promise((resolve,reject)=>{
      axios.get(url,params)
        .then(res=>{
          resolve(res)
        })
        .catch(error=>{
          reject(error)
        })
    })
  }
```

在mockUrl中，在url前缀加上/mock，以防止在不使用mock数据模拟的时候，影响了原有的url请求

```
Mock.mock(RegExp('/mock/api/web/itemPrice/findFinalPriceOfferPage' +'.*'), 'get', function () { 
                 return data.itemPriceInfo
                })
```
#四、在vue项目中的应用（升级版）
**适用对象：** 项目已开发了一部分，后续想引入mockjs
**优点：** 不影响原来请求的使用，其他成员想使用模拟数据时只需要新建按要求data数据即可，不需要修改原有的代码，不担心错提交或漏提交导致测试环境甚至生产环境跑不起来问题
前面操作一样，先install mockjs，在封装好的http请求中，若使用模拟数据，给url加个前缀识别,如：。
```
get (url, params, isMock = false) {
    url = isMock ? `/mock${url}` : url
    return new Promise((resolve, reject) => {
      axios.get(url, {
        params: params
      }).then(res => {
        resolve(res.data)
      }).catch(err => {
        reject(err)
      })
    })
  }
```
####不同的是：
在目录中,新建mockData文件夹，mockData的组成有：
**data文件夹**：用于存放其他成员使用模拟数据时其数据文件；
**getMockData.js**：自动读取data中新增的数据脚本文件；
**mockBaseFunction**:封装一些基础的增删改查方法，方便其他成员按需引用。
![mockData目录](https://upload-images.jianshu.io/upload_images/13541244-56b1a21840629977.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
data中的数据文件，需要用export 封装起来，方便读取。
```
import mockBaseFunction from '../mockBaseFunction'
const successData = { data: null, meta: { success: true, message: 'ok' } }
const itemPriceInfo = mockBaseFunction.mock({
  data: {
    'rows|5': [{
      category: '@name()', // '@name()随机生成英文名字'
      createdByName: '@cname()', // '@name()随机生成中文名字'
      'id|+1': 63, // "id|+number":number 生成的每个id比前面一个大number
      manufacturerName: '@ctitle()',
      'status|1-3': 3, // "status1-3"随机生成1-3中的某个整数
      supplierCompanyId: 61,
      supplierCompanyName: '@ctitle()', // '@ctitle()'随机生成中文标题
    }],
    pageResponse: { start: 0, limit: 5, results: 60 }
  },
  meta: {
    message: 'ok',
    success: true
  }
})

export default {
  mockDemo () {
    // 拦截的url：fun.getMockUrl('需要拦截的url'),请求方法，可为空:'',拦截后的操作方法
    // 拦截get请求，返回模拟数据
    mockBaseFunction.mock(RegExp(mockBaseFunction.getMockUrl('/api/web/itemPrice/findFinalPriceOfferPage') + '.*'), 'get', function (options) {
      return mockBaseFunction.queryMockData(options, itemPriceInfo)
    })
    // 拦截post请求，增加数据，拦截后返回增加模拟数据方法
    mockBaseFunction.mock(mockBaseFunction.getMockUrl('/api/userInfo/addUser'), 'post', function (options) {
      mockBaseFunction.addMockData(options, itemPriceInfo, successData)
    })
    // 拦截post请求，修改数据，拦截后返回修改模拟数据方法
    mockBaseFunction.mock(mockBaseFunction.getMockUrl('/api/userInfo/updateUser'), 'post', function (options) {
      mockBaseFunction.updateMockData(options, itemPriceInfo, successData)
    })
    // 拦截post请求，删除数据，拦截后返回删除模拟数据方法
    mockBaseFunction.mock(mockBaseFunction.getMockUrl('/api/userInfo/deleteUser'), 'post', function (options) {
      mockBaseFunction.deleteMockData(options, itemPriceInfo, successData)
    })
  }
}
```
**getMockData.js**
```
(function () {
  const mockData = require.context('../mockData/data', false, /[A-Za-z0-9-_,\s]+\.js$/i)
  mockData.keys().reduce((modules, modulePath) => {
    // set './app.js' => 'app'
    const moduleName = modulePath.replace(/^\.\/(.*)\.\w+$/, '$1')
    const value = mockData(modulePath)
      modules[moduleName] = value.default
      return modules[moduleName][moduleName]()
  }, {})
})()
```
###重点在于使用require.context（）读取data中的数据文件
**mockBaseFunction.js**
```
import Mock from 'mockjs'

export default {
  mock: Mock.mock,  //这里定义mock变量，是为了方便其他成员使用mockjs，不需要再自己引用mockjs,直接引用mockBaseFunction.js即可
// 将url转换成/mock+url格式
  getMockUrl (url) {
    return `/mock${url}`
  },
// 将请求的传参转换成js对象
  getOptions (options) {
    return JSON.parse(options.body)
  },
// 查询模拟数据
  queryMockData (option, mockData, selfFunction) {
    if (typeof (selfFunction) !== 'function') {
      return mockData
    }
    return selfFunction()
  },
// 模拟新增数据
  addMockData (options, mockData, successData = {}, selfFunction) {
    const params = this.getOptions(options)
    if (typeof (selfFunction) !== 'function') {
      params.id = mockData.data.rows.length + 1
      mockData.data.rows.push(params)
      return successData || ''
    }
    return selfFunction()
  },
// 模拟编辑数据
  updateMockData (options, mockData, successData = {}, selfFunction) {
    if (typeof (selfFunction) !== 'function') {
      const obj = this.getOptions(options)
      mockData.data.rows = mockData.data.rows.map(item => {
        return item.id === obj.id ? obj : item
      })
      return successData || ''
    }
    return selfFunction()
  },
// 模拟删除数据
  deleteMockData (options, mockData, successData = {}, selfFunction) {
    if (typeof (selfFunction) !== 'function') {
      const id = parseInt(this.getOptions(options).id)
      mockData.data.rows = mockData.data.rows.filter(item => {
        return item.id !== id
      })
      return successData || ''
    }
    selfFunction()
  }
}
```
**突破点：**读取data中的数据文件并且调用封装好的方法。

@ by 曾晓霞



