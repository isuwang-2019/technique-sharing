新版网站首页需要做到动画的效果，为了满足产品的需求，我选择使用AOS插件来做动画。
AOS 是一个用于在页面滚动的时候呈现元素动画的工具库，在页面往回滚动时，元素会恢复到原来的状态。可以看下它的demo演示：([http://michalsnik.github.io/aos/](http://michalsnik.github.io/aos/))

以下是具体使用方案：
1、安装AOS
~~~
npm install aos --save
~~~

2、在vue中引入模块
~~~
import 'aos/dist/aos.css'
import AOS from 'aos/dist/aos.js'
~~~

3、初始化
~~~
mounted() {
// 你也可以在这里设置全局配置
  AOS.init({
    offset: 200,   
    duration: 600,   
    easing: 'ease-in-sine',   
    delay: 100
 })
}
~~~

4、使用`data-aos`属性设置动画：

~~~
  <div  data-aos = “fade-in”> </div>
~~~
也可以通过使用`data-aos-*`属性来调整单个动画行为：
~~~
  <div 
     data-aos = “fade-in”
     data-aos-id = “fadeIn”
     data-aos-offset = “ 200 ”
     data-aos-delay = “ 50 ”
     data-aos-duration = “1000 ”
     data-aos-easing = "ease-in-out"
  > 
  </div>
~~~
5、禁用AOS

如果你项在小屏幕设备中禁用AOS，可以：

~~~
AOS.init({   
    disable: 'mobile'  // mobile、phone或tablet。
});
~~~
或者传入一个函数，返回true或false。

~~~
disable: function () {
    var maxWidth = 1024;
    return window.innerWidth < maxWidth;
}
~~~

6、监听AOS动画
AOS提供了两个事件：`aos:in`以及`aos:out`每当任何元素动画化时，你就可以在JS中做一些额外的事情。
也可以通过设置`data-aos-id`属性来告诉AOS在特定元素上触发自定义事件
~~~
<div data-aos="fade-in" data-aos-id="super-duper"> </div>
~~~
然后，能够监听两个自定义事件：

*   `aos:in:super-duper`
*   `aos:out:super-duper`
~~~
// 动画进来时候
document.addEventListener('aos:in:super-duper', ({ detail }) => {
// 如果还有其他动画效果就可以在这儿添加了
  console.log('animated in', detail);
});
// 动画离开时候
document.addEventListener('aos:out:super-duper', ({ detail }) => {
  console.log('animated out', detail);
});
~~~

7、API
AOS对象作为全局变量公开，目前有三种可用方法：

*   `init`\-初始化AOS
*   `refresh`\-重新计算元素的所有偏移量和位置（在窗口调整大小时调用）
*   `refreshHard`\-使用AOS元素和触发器重新初始化数组`refresh`（调用与`aos`元素相关的DOM更改）

更多AOS动画配置可以看下它的github：https://github.com/michalsnik/aos
