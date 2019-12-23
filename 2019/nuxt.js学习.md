##1.nuxt.js简介
nuxt.js是一个基于vue.js的通用框架，集成了Vue 2、Vue-Router、Vuex、Vue-ssr(服务端渲染)、Vue-Meta，最常用的是用来作ssr（服务端渲染）。这里，我们先来科普一下服务端渲染跟客户端渲染的区别.
####(1)服务端渲染与客户端渲染
![服务端渲染与客户端渲染过程对比图.png](https://upload-images.jianshu.io/upload_images/13541244-4634f4ddea06b8d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最大的区别，简单点来说就是责任越大压力越大(谁渲染谁压力大)
1)服务端渲染优缺点：
优点：前端耗时少、有利于SEO、无需占用客户端资源、后端生成静态文件
缺点：不利于前后端分离、占用服务器资源
适用于应用交互不太复杂需要良好的SEO且服务器性能好
2)客户端渲染优缺点：
优点：有利于前后端分离、服务器压力小
缺点：前端响应慢、不利于SEO
适用于应用交互复杂且不需要良好的SEO，例如企业内部系统
####(2)Vue ssr
既然nuxt.js最常用是用来做ssr，那就更有必要提一下Vue SSR了。一目了然，它是基于vue.js的服务端渲染。主要渲染插件是：vue-server-renderer，官网给出的流程图如下：
![vue-server-renderer流程图.png](https://upload-images.jianshu.io/upload_images/13541244-3d8ef5729011b4ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看出vue的后端渲染分三个部分组成：页面的源码（source），node层的渲染部分和浏览器端的渲染部分。
source分为两种entry point,一个是前端页面的入口client entry,主要是实例化Vue对象，将其挂载到页面中；另外一个是后端渲染服务入口server entry,主要是控服务端渲染模块回调，返回一个Promise对象，最终返回一个Vue对象。
关于vue-ssr的配置使用，不是本篇文章的重点，就不一一详述了。具体参见[vue-ssr官网](https://ssr.vuejs.org/zh/guide/)

####(3)nuxt.js的使用
前面科普了一大堆，只是为了让我们能更好的上手nuxt.js。
1)使用npx或者yarn两种方式下载
`npx create-nuxt-app <项目名>`
`yarn create nuxt-app <项目名>`
2)接着需要你做一些选择，下载方式、UI框架、集成的服务端框架、语法校验等等，做完选择后会帮你安装相关依赖。
3)完成后，进入文件夹，运行`npm run dev`即可
4)nuxt.js概述：
![nuxt.js.png](https://upload-images.jianshu.io/upload_images/13541244-4212d977471bec70.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
5）一些常用配置
js、css引入
```
css: [
    'iview/dist/styles/iview.css'
  ],
 plugins: [
    '@/plugins/iview'
  ],
```
修改网站icon
icon.png文件存放在static文件夹下，nuxt.config.js中配置head属性
```
head: {
    title: process.env.npm_package_name || '',
    meta: [
      { charset: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { hid: 'description', name: 'description', content: process.env.npm_package_description || '' }
    ],
    link: [
      { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' }
    ]
  },
```
中间件的使用
```
//全局使用
module.exports = {
  router: {
    middleware: '中间件名称'
  }  
}

//页面单独使用
export default {
    middleware: '中间件名称'
}
```
关于nuxt.config.js的其他配置，官网说得很详细，可以直接在[官网](https://zh.nuxtjs.org/)学习使用。
##2.nuxt.js一些常用操作
####1.关于路由
只要我们在pages创建新的vue页面时，运行时.nuxt包中的router.js就会自动生成路由地址。
![自动生成路由地址](https://upload-images.jianshu.io/upload_images/13541244-397e541a98b32c10.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
####(1)路由跳转
路由跳转有两种方式，router-link与nuxt-link,使用方式跟原来vue一样
```<nuxt-link to="/user/9">页面跳转</nuxt-link> 
<nuxt-link :to="{name:'user-id',params:{id:2}}">页面跳转2</nuxt-link>
```
也可以使用
`
this.$router.push({name:'user-id'})
`
####(2)关于带参数的路由地址校验
```
export default {
       validate ({ params }) {
       // 路由参数校验，必须是一个数字,如果校验失败，抛出异常
       if(/^\d+$/.test(params.id)){
        return true
       }
       throw new Error('Under Construction!')
     }
}
```
在跳转页面的js中加上validate校验，这样在页面预渲染时就会自动校验路由参数是否符合规范，否则抛出异常
####(3)页面是否必须已登录才能访问
在没有使用nuxt.js框架时，我们可以采取的办法一般有：在配置路由时，给路由加个meta:{isHadLogin: true},再通过router.beforeEach判断,简单例子如下：
```import Vue from 'vue'
import Router from 'vue-router'
import Store from '../vuex/store'
Vue.use(Router)
const route = [{
    path: "/login",
    component:()=>import('../pages/login'),
    name: "login"
  }, {
    path: "/user/One",
    component: ()=>import('../pages/user/one'),
    name: "user-One",
    meta:{
      isHadLogin:true
    }
  }, {
    path: "/user/:id",
    component: ()=>import('../pages/user/_id'),
    name: "user-id",
    meta:{
      isHadLogin:true
    }
  }, {
    path: "/",
    component:()=>import('../pages/index'),
    name: "index",
    meta:{
      isHadLogin:true
    }
  }, {
    path: "/:slug/component",
    component: ()=>import('../pages/_slug/component'),
    name: "slug-component",
    meta:{
      isHadLogin:true
    }
  }
)]
const router = new Router({
  mode:'history',
  route
})
router.beforeEach((to,from,next)=>{
  store.dispatch('getCurrentUser').then(() => {
    // 未登录，需要登录跳转到登录页面
    if (to.meta.isHadLogin && !Store.state.authUser) {
      window.location.href = 'localhost:1000/login'
    }
    next()
  })
})
```
使用nuxt.js框架的话，需要在middleware(中间件)中加个控制，需要登录才能访问的页面再加个middleware配置即可。例如：
文件地址：middleware/auth.js
```$xslt
export default function ({ store, redirect }) {
  if (!store.state.authUser) {
    // error({
    //   message: 'You are not connected',
    //   statusCode: 403
    // })
    return redirect('/login')
  }
}
```
需要登录才能访问的页面配置：
```$xslt
export default {
  middleware: 'auth'
}
```
这样配置之后，在跳转到该页面时，如果没有登录，就会跳到登录页面。

####2.axios跨域请求
在nuxt项目中，是默认安装axios，但跨域请求，有点小区别于非nuxt.js的项目。
在没有使用nuxt.js时，我们在vue.config.js文件中配置proxy代理，如下：
```module.exports = { 
        devServer: {
           port: 8085, // 端口号
            host: '127.0.0.1',
            https: false ,
            proxy: {
              '/api': {
                target: 'http://localhost:9094', // 对应自己的接口
                changeOrigin: true,
                ws: true,
                pathRewrite: {
                  '^/api/': ''
                }
              }
          }
      }
}
```
而在nuxt.js的项目中，我们是在nuxt.config.js中配置proxy代理，代码如下：
```
modules: [
    '@nuxtjs/axios',
    '@nuxtjs/proxy'
  ],
  proxy: {
    '/api': {
      target: 'http:www.xxx.com',
      changeOrigin: true,
      pathRewrite: {
        '^/api ': ''
      }
    }
  }
```
####3.为项目配置固定的ip端口号
在nuxt.js项目中中，我们是在pagekage.json文件中配置如下设置：
```
"config": {
    "nuxt": {
      "host": "0.0.0.0",
      "port": "3000"
    }
},
```
而本人之前在普通vue项目中，遇到过在vue.config.js设置了固定端口号，可是项目运行起来并不是我原来想要的端口号，在网上找到的方法是：
```
npm install portfinder@1.0.21
```
运行这个命令就可
####4.跨域身份验证
1）package.json配置：
```
{
  "name": "example-auth-jwt",
  "dependencies": {
    "cookieparser": "^0.1.0",
    "js-cookie": "^2.2.0",
    "nuxt": "latest"
  },
  "scripts": {
    "dev": "nuxt",
    "build": "nuxt build",
    "start": "nuxt start",
    "post-update": "yarn upgrade --latest"
  }
}
```
2)vuex配置：即store文件夹中的index.js配置
```
const cookieparser = process.server ? require('cookieparser') : undefined

export const state = () => {
  return {
    auth: null
  }
}
export const mutations = {
  setAuth (state, auth) {
    state.auth = auth
  }
}
export const actions = {
  nuxtServerInit ({ commit }, { req }) {
    let auth = null
    if (req.headers.cookie) {
      const parsed = cookieparser.parse(req.headers.cookie)
      try {
        auth = JSON.parse(parsed.auth)
      } catch (err) {
        // No valid cookie found
      }
    }
    commit('setAuth', auth)
  }
}
```
登录页配置：
```
<script>
const Cookie = process.client ? require('js-cookie') : undefined

export default {
  middleware: 'notAuthenticated',
  methods: {
    postLogin () {
      setTimeout(() => { // we simulate the async request with timeout.
        const auth = {
          accessToken: 'someStringGotFromApiServiceWithAjax'
        }
        this.$store.commit('setAuth', auth) // mutating to store for client rendering
        Cookie.set('auth', auth) // saving token in cookie for server rendering
        this.$router.push('/')
      }, 1000)
    }
  }
}
</script>
```
这样，我们就能做出一个跨域带身份认证的登陆了，登录后的信息存在store里面，只要调用this.$store.state.auth即可拿到。


##3.结束语
nuxt.js还有很多优秀的功能供我们使用，但小编的体验还不够深入，所以暂时只能介绍这么多了。以后对nuxt.js有了更深入的体验，再来进一步更新。感恩观看！



@by 曾晓霞