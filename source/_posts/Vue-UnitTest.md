title: Vuex 的單元測試（基於 Jest 框架）
tags:

	- Vue.js
	- Front-end
	- Unit Test

categories:

- Programming

date: 2019-08-22 14:17:52
---
# 前言

暑假在 UCLA 做項目時基本成為了全棧工程師（帶我的 Ph.D. 學長是搞機器學習的，所以除了 ML 之外系統中這樣那樣雜七雜八前端後端數據庫的東西都變我的工作了），在項目接近尾聲時覺得應該記錄下一些過程中的經驗。一來以免以後做項目時想不起來又重新在Google 的海洋里打撈，二來希望能多少加快其他遇到同樣問題的中文使用者搞定它的時間。之後應該還會更新其他的一些東西（吧

# 什麼是 Vuex？

> Vuex 是一个专为 Vue.js 应用程序开发的**状态管理模式**。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。Vuex 也集成到 Vue 的官方调试工具 [devtools extension](https://github.com/vuejs/vue-devtools)，提供了诸如零配置的 time-travel 调试、状态快照导入导出等高级调试功能。
>
> -- 官網說明

[Vuex](https://vuex.vuejs.org/zh/) 是一個用在 Vue 的全局狀態管理模塊，可以把Vuex 看成前端的數據庫。雖然前端的數據存儲很大一部分都是部件相關的（在 Vue.js 裡面對應的就是每個 Vue 文件里的 data），但也有一些數據是需要全局維護的，在不是很複雜的情況下可以維護一個.ts 文件里的字典，可是當交互越來越複雜的時候一個可追蹤狀態的模塊對於代碼的質量就很重要了。

Vuex 的一個重要思想是狀態和操作分離，即一個典型 Vuex 對象中的 state, mutation 以及 action。前端所需要的全局信息維護在 state 中，而對 state 的所有操作只能透過 mutation 來完成（action 是為了異步調用對 mutation 的封裝，此處不展開）以確保操作的合法性和可預期性。

我在這篇文章並不試圖從頭解釋如何存取 state 或編寫 mutation，這在官方文檔中已經有比較詳細的[說明](https://vuex.vuejs.org/zh/guide/mutations.html)。我的主要目的是記錄如何在一個 Vue 項目中有效地進行關於 Vuex 的單元測試（這點在官方文檔中沒有那麼完整），從而保證前端全局信息的可靠性，因此假定你已經對 state, mutation, commit 等概念有了基本的認識，如果沒有可以先看文檔。

對於 Vuex 的測試可以分為兩種，一種是測試對 Vuex 執行 mutation/action 時 state 有沒有預期的變化，一種是測試組件中有沒有執行期望的 mutation。

# 如何測試 Vuex 本身

以下考慮一個典型 Vuex 對象的定義，其中有一個簡單的 state 和兩個簡單的 mutation。

> 相比于純 JavaScript ，我是一個 TypeScript 愛好者。

```typescript
export const store = new Vuex.Store({
  state: {
    count: 0 as number
  },
  mutations: {
    increase: function(state) {
      state.count ++;
    },
    increaseNum: function(state, payload) {
      // 一個更一般性的情況，payload 是參數對象，假定其中有 num 這個變量
      state.count += payload.num;
    }
  }
  
export default store
```

那麼對於 vuex 本身的測試其實相當簡單，以上面的代碼為例：

```typescript
// store.spec.ts
import store from '../store';
import Vuex from 'vuex';
import Vue from 'vue';
Vue.use(Vuex);

describe('test vuex', () => {
  it('test init', () => {
    expect(store.state.count).toBe(0);
  });
  it('test mutation', () => {
    store.commit('increase');
    expect(store.state.count).toBe(1);
  });
});
```

其實就是對於 store 允許的每個 commit 操作一下，測試看看執行前後 state 的變化是否符合預期。由於 Vuex 操作與狀態分離的良好性質，這樣就基本可以確保運行如同預期。

# 如何在調用處測試 Vuex？

在某個`.vue`或`.ts`中我們通常會這樣來使用 vuex 對象：

```typescript
import store from './store';

// A common .ts file
const someFunction = function(){
  store.commit('increase');
  store.commit('increase', {
    num: 2
  });
}

// A function in vue componenet
export default Vue.extend({
  data: function() {
    return {
      // Some data.
    }
  },
  methods: {
    someFunction: function() {
      this.$store.commit('increase');
    }
  }
});
```

其中 Vue 組件的`$store`寫法需要在根組件注入，`vue-cli`生成的典型情況下可以在入口 `main.ts` 這麼寫：

```typescript
import MyVuexStore from './store';

new Vue({
  store: MyVuexStore,
  router: MyRouter,
  render: h => h(App)
}).$mount("#app");
```

那麼測試代碼應該怎麼寫呢？我在搜索 Stack Overflow 并結合 Jest 和 Vue/Vuex 官方教程后覺得比較好的寫法是這樣：

```typescript
import { shallowMount, createLocalVue } from "@vue/test-utils";
import VueRouter from "vue-router";
import Vuex from "vuex";
import ElementUI from "element-ui";
const localVue = createLocalVue();
localVue.use(ElementUI);
localVue.use(Vuex);

import TestComponent from "../test_component.vue";
describe("Component test: test_component.vue", () => {
  let store;
  let wrapper;
  let mutations;
  beforeEach(() => {
    mutations =  {
      increase: jest.fn(),
    };
    store = new Vuex.Store({
      // state 的內容在這裡其實可有可無，只是做個範例
      state: {
        count: 0 as number
      },
      // 這是 vue 的語法糖，表示 mutations: mutations
      mutations
    });
    // 測試 Vue 組件的時候會使用 wrapper 掛載，可見官方文檔說明
    wrapper = shallowMount(SketchPad, {
      localVue,
      store
    });
  });

  it("test: when increase mutation is called.", () => {
    const {increase} = mutations;
    wrapper.find("[id='test-btn']").trigger("click");
    expect(increase.mock.calls.length).toBe(1);
  });
});

```

這個測試代碼里有幾個重點：

1. 使用`createLocalVue`來創建一個測試用的 vue 對象，可以避免在本測試里掛載或注入的操作對全局 Vue 造成影響。
2. 使用 `shallowMount`表示只掛載待測試組件，而不掛載、渲染子組件，符合單元測試的思想且可以加速測試時間。
3. `beforeEach` 的 hook 確保每個子測試之間的隔離。
4. 把這個組件里會用到的所有 mutation 寫出來，函數內容使用`jest.fn()`，因為我們在這裡測試時並不關心 store 實際的狀態變化（這個變化應該由前面提到的對 Vuex 對象本身的單元測試確保），我們只需要保證組件中操作時有如同我們預期的呼叫 mutation 即可，`jest.fn()`就是這樣用來記錄被調用信息的 mock 函數。
5. 前端測試實際上就是要測試用戶交互時的行為，從測試代碼里可以看到一種典型的操作：點擊組件中的某個按鈕（`id='test-btn'`）時我們希望 Vuex 做出相應變化，這種情況就可以用上面這種寫法。通過`jest.fn()`捕獲到的呼叫次數就可以判斷效果是否如同預期。

總結而言，Vuex 的測試核心就兩點：

1. store 的每個 mutation 被調用時狀態的變化是否符合預期。
2. 組件中特定操作觸發時，是否會向 Vuex 對象提交預期的 commit。

得益于 Vuex 的設計思想，這兩點被好好測試時就基本可以保證全局狀態的可靠性了，action 較為複雜，但本質上也是如此。