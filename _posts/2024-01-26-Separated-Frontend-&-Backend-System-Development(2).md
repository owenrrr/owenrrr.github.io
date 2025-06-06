---
layout: post
title:  "Separated Frontend & Backend System Development(2)"
date:   2024-01-26 09:04:02 +0800
categories: Demo
tags: Tech-Share
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
# 前後端分離系統開發介紹(2)

## 2. 前端
### 2.1 前置工作
- Vue需要使用到Node.js和npm；前者是一個Web Application framework；後者是Node.js預設的Dependency Manager；兩者都需要安裝，詳情見官網
- 創建Vue項目不需要任何IDE，只需要在安裝完npm后執行命令`npm install -g vue-cli`先安裝Vue的輔助程序，這個輔助程序可以生成空項目
- 然後執行`vue create xxx`就可生成項目，其中`xxx`是項目名
- 如果需要可見 [Vue官網](https://vuejs.org/) 查詢各種API和框架介紹

### 2.2 整體框架

```
-node_modules
-public
-src
    -assets
    -views
    -components
    -store
        -index.js
    -router
        -index.js
    -App.vue
    -main.js
-package.json
-package-lock.json
-vue.config.js
...
```

- `node_modules`是所有依賴存放目錄
- `package.json`是npm用來管理這個vue項目依賴的配置文件；借由`package.json`來生成更加全面的依賴樹`package-lock.json`。若要更改或添加依賴可通過手動加入`package.json`或執行`npm install xxx`
- `vue.config.js`是一個可選的配置文件，會被`vue-cli`加載
- `src/views`和`src/components`并不是强制要求的目錄，只是規範: `views/`存放整個頁面的組件；`components/`存放小的或可組合的組件
- `src/store`負責全局數據的定義
- `src/router`負責組件路由的定義
- `public`存放靜態資源

### 2.3 模塊/文件介紹
#### 2.3.1 .vue 文件基本介紹

```
<template>
    <div class="outer">
        <SW nav="login"></SW>
        </div>
    </div>
</template>

<script>
import SW from './SwitchLoginRegister.vue'

export default {
  name: 'LoginPage',
  components: {
    SW
  },
  props: {
    msg: String
  },
  mounted(){
  },
  data(){
    return{
        username: '',
        ...
    }
  },
  methods:{
    function() {
        ...
    },
  }
}
</script>

<style scoped>
...
</style>
```
- 在Vue項目中，一個.vue文件就默認是一個組件，而所有的組件都是挂載在 `App.vue` 這個根組件上；組件與組件之間並沒有聯係，可以通過在一個組件中導入另一個組件來實現父子組件。
- 一個.vue文件基本由三個部分組成: `<template></template>`、`<script></script>` 和 `<style></style>`。`<template>` 是聲明最基礎的html標簽和整體佈局，`<template>`標簽内至少需要一個標簽作爲根標簽，因爲前面提到的組件導入就是從這個根標簽作爲嵌入點。例如上面的A組件導入B組件最終就會變爲A(下面):

```                                    
<template>                                  
    <div class="A">                                         
        <B></B>                                     
    </div>      
</template>

<template>
    <div class="B">
    </div>
</template>

<template>                                  
    <div class="A">                                         
        <div class="B">
        </div>                                   
    </div>      
</template>                        
```

- `<script>`是為整個組件提供脚本的地方，最外層包含了一個組件的聲明，同時裏面還有很多實用的屬性: `components`聲明從外部導入組件；`props`允許父組件傳遞數據給子組件；`mounted()`在每個組件加載時會調用的方法(類似JAVA中的構造函數，詳情見[Vue Lifecycle](https://vuejs.org/guide/essentials/lifecycle))；`data`聲明這個組件内使用到的數據；`methods`聲明組件内使用到的方法
- `<style>`與基礎的寫法相同
#### 2.3.2 main.js

```
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import store from './store'
import VueRouter from 'vue-router'

Vue.config.productionTip = false

Vue.use(VueRouter)

new Vue({
  el: "#app",
  router: router,
  store: store,
  render: h => h(App),
}).$mount('#app')
```

- 負責導入Vue.js庫和其他使用到的全局庫和組件
- 創建Vue實例，並將其掛載到HTML文檔中的一個DOM元素上，通常是一個div元素。可由下列例子(App.vue)中看出，`main.js`把整個Vue實例掛載到`App.vue`中的`<div id="app">`上，把這個`<div>`當作最外層的HTML元素:
- 
```
<template>
  <div id="app">
    <router-view/>
  </div>
</template>
```
- 配置Vue實例的選項，例如路由器（Vue Router）、狀態管理器（Vuex）以及其他插件。
- 最後，啟動Vue應用程序。

#### 2.3.3 Vuex(狀態管理)
`Vuex` 是Vue項目中最重要的觀念之一，它管理項目中所有的狀態。由於Vue本身設計原因，數據流通僅能通過父組件單方面通知子組件，而不能實現雙向流通。因此Vuex能掌控並響應所有數據變化就變得尤爲重要，所有 `Vuex` 的觀念和使用可在 [官網](https://v3.vuex.vuejs.org/) 查詢。首先要瞭解Vue中的`data`和`computed`有何區別；`data`沒辦法做到當其他組件改變值，這個組件也跟著改變；而`computed`則是如果這個方法裏的值變動並會改變計算結果就會更新，這個特性可以結合 `Vuex-store` 來使得當一個全局值變動則所有組件一起變動

- store/index.js
```
import Vue from 'vue'
import Vuex from 'vuex'

Vue.use(Vuex)

const store = new Vuex.Store({
    state: {
        username: '',
        password: '',
        mail: ''
    },
    mutations: {
        set_username(state, newVal) {
            state.username = newVal
        },
        set_password(state, newVal){
            state.password = newVal
        },
        set_mail(state, newVal) {
            state.mail = newVal
        }
    }
})
export default store
```

- `Vue.use(Vuex)` 是讓 `Vuex` 挂載在 `Vue` 實例上，在 `main.js`上導入即可使得全局都可以使用這個配置
- `state` 是Vuex的基本狀態，通過定義在 `state` 裏可以在各個組件内部通過 `this.$store.state` 調用，非常方便
- `mutations` 是爲了解決很多組件若要調用相同的方法都需要再寫一遍，若是現在 `store/index.js` 中定義就能在各個組件中直接調用方法
- 另外 `Vuex` 還有其他觀念如 Getters/Actions 本次案例中沒有使用到，可查閲 [文檔](https://v3.vuex.vuejs.org/) 學習

#### 2.3.4 Router
路由功能是Vue提供的基本跳轉功能,使用方法也很簡單
- router/index.js
```
import Router from 'vue-router'
import LoginPage from '../components/LoginPage.vue'

const router = new Router({
    routes: [
        {
            path: '/',
            name: 'mainPage',
            redirect: '/login'
        },
        {
            path: '/login',
            name: 'login',
            component: LoginPage
        }
    ]
})
export default router
```
-創建一個Router實例並將其挂載到全局Vue實例上(main.js)

## 3. 演示
```
client: http://localhost:3000/
server: http://localhost:8080/demo/
```
- startup
```
cd %NGINX_HOME%
start nginx

cd %TOMCAT_HOME%\bin
./startup
```
- shutdown
```
cd %NGINX_HOME%
nginx.exe -s quit

cd %TOMCAT_HOME%\bin
./shutdown
```

</div>
</body>
</html>
