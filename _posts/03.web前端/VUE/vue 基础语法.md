---
title: vue 基础语法
date: "2019-07-10 11:40:00"
categories:
- web前端
- VUE
tags:
- 前端
- VUE
toc: true
typora-root-url: ..\..\..
---

## 1. vue的基础语法介绍

#### **1-1基本数据绑定**

```javascript
<div id="app">
  {{ msg }}
</div>
//script
new Vue({
  el:"#app",//代表vue的范围
  data:{
    msg:'hello Vue' //数据
  }
})
```

在这个例子中我们可以进行赋值

```javascript
var app = new Vue(...);
app.msg = '初探vue';
//那么页面的值就被改变了
```

#### **1-2 v-html命令**

```javascript
<div id="app" v-html="msg">
 
</div>
//script
new Vue({
  el:"#app",//代表vue的范围
  data:{
    msg:'<h1>你好，世界</h1>' //这里就不会是文本了 而是会被当成是html标签了
  }
})
```

#### **1-3 v-on:click||@click指令**

```javascript
<div id="app">
<button v-on:clikc="clickHead">事件</button>
<button @click="clickHead">事件</button>
</div>

//js
new Vue({
  el:"#app",
  methods:{
    clickHead:functoin(){
      alert('vue的事件绑定')
     }

  }
})

//在es6语法中
对象中的函数可以
const json={
  clickHead(){
    //do something
  }
}
json.clickHead()调用方法

//和一样的
const json={
  clickHead:function(){
    //do something
  }
}
```

#### **1-4 v-bind 属性绑定指令**
 例如绑定class  和id  已经已经存在的属性 和自定义属性

> 绑定类名

```javascript
<div id="app">
  <p v-bind:class="className">{{msg}}</p> 
</div>
/*
v-bind 自定义名字
v-bind:id="..."  绑定id名字
v-bind:title="..."绑定title属性
v-bind:style="..." 绑定样式属性 
v-bind:...="..."绑定自定义属性
、、、
*/

//js
new Vue({
    el:"#app",
    data:{
        msg:'这是v-bind绑定数据',
        className:'contatiner'
    },
})
const Name = document.querySelector('.contatiner');
console.log(Name) //能正常的获取这个元素
```

#### **1-5 v-show**

> 根据值的真假 控制元素的display的属性

```javascript
<div id="app">
 <p v-show="msg"> 可以看到啊 </p>
 <p v-show="msg1"> 不可以看到啊 </p> 
</div>

//js
new Vue({
  el:"#app",
  data:{
    msg:1+1==2,
    msg1:1+1!=2
  }
})
```

#### **1-6 v-text**

> 赋予文本值

```javascript
<div id="app">
 <p v-text="msg"> 可以看到啊 </p> <!-- 最终会被替换掉 1+1==2 -->
</div>

//js
new Vue({
  el:"#app",
  data:{
    msg:'你好哈 v-text'
  }
})
```

#### **1-7 v-for**

> 循环

```javascript
 <div id="app">
   <ol>
       <li v-for="module in modules">{{module.msg}}</li>
   </ol>
</div>

//js
new Vue({
   el:"#app",
   data:{
       modules:[
           {msg:'第一个'},
           {msg:'第二个'},
           {msg:'第三个'},
           {msg:'第四个'}
       ]
   }
})
```

#### **1-8 v-model 双向数据绑定**

```javascript
<div id="app">
    <input type="text" v-model="msg">
    <p>{{msg}}</p>
</div>

//js
new Vue({
    el:"#app",
    data:{
        msg:''
    }
})
```

#### **1-9 template属性**

```javascript
<div id="app">
    <span>你好啊</span>
</div>

//js
new Vue({
  el:"#app",
  data:{
    msg:'这是数据'
  },
  template:"<div>模板div</div>"
})
```

最终的效果是把id为app的div直接替换成template里面的元素

![1562730177845](/img/1562730177845.png)

注意在template的值中不能含有兄弟根节点

```javascript
new Vue({
  el:"#app",
  template:"<div>1<div><div>2<div>"
})
```

这样是错误的  , 在template 可以把团苏片段放在script标签内

```javascript
<div id="app">
        <span>你好啊</span>
    </div>
    <script type="x-template" id="temp">  //这个地方就是模板片段
        <div id="tpl">
            <p> 这是模板啊,{{msg}} </p>
            <input type="text" v-model="msg" />
        </div>
    </script>
    <script src="js/vue.min.js"></script>
    <script>
        new Vue({
            el:"#app",
            data:{
                msg:''
            },
            template:"#temp"
        })
    </script>
```

## 2. vue实例中的属性

#### **2-1 el**

> el表示在vue的实例中的作用范围

```javascript
new Vue({
  el:"#app" //作用在id名为app的div上
})
```

#### **2-2 data**

> data的属性就是数据绑定

```javascript
new Vue({
  data:{
    msg:'数据的值'    
    arrayData:[
      {title:'这是1'}
    ]
  }
})
```

#### **2-3 methods**

> methods绑定事件

```javascript
new Vue({
  el:"#app",
  methods:{
    mouseClick(){
      alert('绑定事件')
    }
  }
})
```

#### **2-3 template**

> template模板的绑定

```javascript
new Vue({
  el:"#app",
  template:'<div>这是模板属性</div>'
})
```

#### **2-4 render**

> render模板的绑定

```javascript
new Vue({
    el:"#app",
    render(createElement){
        return createElement(
            "ol",
            {  //新创建的元素身上绑定属性
                style:{
                    fontSize:'30px',
                    border:'1px solid red',
                    fontWeight:'bold'
                },
                attrs:{
                    title:'你好啊',
                    coundNum:'01',
                    id:'ls',
                    class:'bg'
                }
            },
            [
                createElement("li",'这是第一个文本'),
                createElement("li",'这是第二个文本'),
                createElement("li",'这是第三个文本'),
            ]
        )
    }
})
```

![1562730261599](/img/1562730261599.png)

## 3. vue的自定义指令

> 在vue中除了内置的指令如v-for、v-if...,用户可以自定义指令官网

```javascript
//这里定义v-focus
directives: {
  focus: {
    // 指令的定义
    inserted: function (el) {
      el.focus()
    }
  }
}
```

> 一个指令定义对象可以提供如下几个钩子函数 (均为可选)：

-  **bind**：只调用一次，指令第一次绑定到元素时调用。在这里可以进行一次性的初始化设置。
-  **inserted**：被绑定元素插入父节点时调用 (仅保证父节点存在，但不一定已被插入文档中)。
-  **update**：所在组件的 VNode 更新时调用，但是可能发生在其子 VNode 更新之前。指令的值可能发生了改变，也可能没有。但是你可以通过比较更新前后的值来忽略不必要的模板更新 (详细的钩子函数参数见下)。
-  **componentUpdated**：指令所在组件的 VNode 及其子 VNode 全部更新后调用。
-  **unbind**：只调用一次，指令与元素解绑时调用。

## 4. vue的扩展

#### **4-1 v-bind根据条件绑定类名**

> 案例 比如现在在true的情况下绑定red类名

```javascript
<div id="app">
    <span :class="{red:addClass}">{{msg}}</span> <!--可以利用简单的表达式-->
<!--这是v-bind指令可以省略-->
</div>
<script src="js/vue.min.js"></script>
<script>
    new Vue({
        el:"#app",
        data:{
            msg:'条件绑定类名',
            addClass:true
        }
    })
</script>
```

#### **4-2 v-on || @eventName 事件绑定 有一个事件修饰符**

```javascript
//阻止事件冒泡
<div v-on:click.stop="eventHadles"></div>
//阻止默认事件
<div v-on:click.prevent="eventHadles"></div>
//事件只能触发一次
<div v-on:click.once="eventHadles"></div>
//只能回车触发事件
<div @keyup.enter="eventHadles"></div>
//只能指定keyCode的值触发事件
<div @keyup.13="eventHadles"></div>
```

## 5. vue的计算属性 -computed

> 例如一个案例需要过滤一些列表
>
> ![1562730348815](/img/1562730348815.png)
>
> 而我们需要利用v-for进行循环出来列表

**需要用到我们的实例的属性**
 computed 说透点就是过滤你的数据根据你的条件进行过滤

```javascript
//js
new Vue({
  el:"#app",
  data:{
    list:[
     {
        title:'你好啊',
        isChecked:true
     },
      {
        title:'你好啊2',
        isChecked:false
     }
    ]，
    hash:'all' //过滤条件
  },
  computed:{ //重头戏 
      filterData(){ 
      //这个根据你的条件进行过滤 记住这个函数最终返回的是数据需要return 数据出来 不需要调用此函数
        let filterDatas = {
          all(list){
              return list
          },
          unfinshed(list){
            return list.filter(function(item){
                return !item.isChecked
            })
          },
          finshed(list){
              return list.filter(function(item){
                return item.isChecked
            })
          }
        }
        return filterDatas[你的条件]?filterDatas[你的条件](this.list):this.list
        //这里的你的条件可以使hash值 然后很久hash值的不同进行过滤 不需要调用这个函数
      }
  }
})
//然后在v-for的指令中
<div v-for="item in filterData"></div> <!--注意不是list了 而是刚刚的computed中的计算属性的函数名字-->
```

## 6. vue的组件

#### **6-1 底层学习组件**

> 组件 (Component) 是 Vue.js 最强大的功能之一。组件可以扩展 HTML 元素，封装可重用的代码。在较高层面上，组件是自定义元素，Vue.js 的编译器为它添加特殊功能。在有些情况下，组件也可以表现为用 is 特性进行了扩展的原生 HTML 元素。

```javascript
//html
<div id="app">
  <my-test></my-test> <!--自定义标签-->
</div>

//js
Vue.component('my-test',{  //注册组件
  template:'<div>初学组件</div>'       
});

new Vue({
  el:"#app"
})
```

#### **6-2 父子组件通信**

> 利用props进行传值

```javascript
//局部组件
<div id="app">
  <my-test msg="你好"></my-test>
  <my-test msg="传值2"></my-test>
</div>
//js
new Vue({
  el:"#app",
  components:{
    'my-test':{
      props:['msg'],
      template:'<div>{{msg}}</div>'
    }
  }
})
//全局组件
<div id="app">
  <my-test msg="你好"></my-test>
  <my-test msg="传值2"></my-test>
</div>

//js
Vue.component('my-test',{
  props:['msg'],
  template:'<div>{{msg}}</div>'
})
```

> 如果需要传的值在vue的实例中

```javascript
//html
<div id="app">
  <my-test v-bind:listData="list"></my-test>
</div>

//js
new Vue({
  el:"#app",
  data:{
    list:[
      {title:'这是数据'},
      {title:'这是数据22'}
    ]
  },
  components:{
    'my-test':{
      props:['listData'],
      template:`
        <select name="" id="">
          <option v-for="item in listData">{{item.title}}</option>
        </select>
      `
    }
  }
})
```

## 7. vue获取dom元素

```javascript
在想获取的元素身上
<div class="container" rel="getDom">
//js
new Vue({
  el:"#app",
  methods:{
     _someDo(){
      this.dom = this.$refs.getDom;
    }
  },
  mounted(){
    this._someDo(); //vue完成挂载 调用函数
  }
})
```

## 8. vue渲染dom是异步$.nextTick()函数

> 因为vue渲染dom是异步的所以直接操作dom是会出错的
>  案例 例如 创建vue实例的时候请求接口数据，然后要进行dom元素操作

```javascript
new Vue({
  data:{
    result:''
  },
  created(){
    axios.get('/data')
    .then(data=>{
      this.result = data.data  //如果在dom中用到了v-for这些元素 而我们乡操作这些元素
      this.$nextTick(()=>{
        //这个函数的意义就是等待dom渲染结束后执行
      })
    })
    .catch(err=>{
      //错误处理
    })
  }
})
```

