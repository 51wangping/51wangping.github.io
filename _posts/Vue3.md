---
layout: p关于vue3的知识"
date: 2020-11-10
description: "这个是个人总结的vue3的知识点"
tag: hexo
---

# Vue3

```js
<template>
    <ul>
    	<li
			v-for="item of hits"
			:key="item.objectId"
			>
    		<a :href="item.url">{{item.title}}</a>
    	</li>
    </ul>
</template>
<script>
    import { reactive } from 'vue'
	export default {
        setup() {
            const sate = reactive({
                hits: []
            })
            return state
        }
    }
</script>
```

组合 API 需要通过`setup()`方法来启动，`setup()`返回的数据可以在模板内使用，可以理解为 vue2 里面的`data()`方法返回的数据，不同的是，返回的数据需要先经过`reactive()`方法进行包裹，将数据变成相应式

**`setup`是围绕`beforeCreate`和`created`生命周期钩子运行的，所以不需要显示的定义，意思就是在这些钩子中编写的任何代码都应该直接在`setup`函数中编写，在`setup`中执行就相当雨在`created`阶段执行**

```js
import { reactive, onMounted } from "vue";

export default {
  setup() {
    const state = reactive({
      hits: [],
    });
    onMounted(async () => {
      const data = await fetch(
        "https://hn.algolia.com/api/v1/search?query=vue"
      ).then((res) => res.json());
      state.hits = data.hits;
    });
    return state;
  },
};
```

### h

返回一个返回的虚拟 node，通常缩写为`VNode`一个普通对象，其中包含向 Vue 描述它应在页面上旭冉然哪种节点的信息，包括任何子节点的信息，它用于手动编写的渲染函数

```js
render(){
    return Vue.h('h1', {}, 'Some title')
}
```

递归监听存在的问题 非常消耗性能

每一个数据都被包装成了 proxy

```js
// 专门用来创建递归监听的数据的
// triggerRef 根据传入的数据主动的去更新界面
import { shallowReactive, shallowRef, triggerRef } from "vue";

export default {
  name: "App",
  setup() {
    let state = shallowReactive({
      a: "a",
      gf: {
        b: "b",
        f: {
          c: "c",
          s: {
            d: "d",
          },
        },
      },
    });
    function myFn() {
      (state.a = "1"),
        (state.gf.b = "2"),
        (state.gf.f.c = "3"),
        (state.gf.f.s.d = "4");

      console.log(state);
      console.log(state.gf);
      console.log(state.gf.f);
      console.log(state.gf.f.s);
      // 打印之后发现只有第一层被包装成了proxy,其余都没有
      // 所以只要第一层发生改变 就回去更新UI
      // 如果不修改第一层下面的是监听不到的 不会更新UI
    }
  },
};

export default {
  name: "App",
  setup() {
    let state = shallowRef({
      a: "a",
      gf: {
        b: "b",
        f: {
          c: "c",
          s: {
            d: "d",
          },
        },
      },
    });
    function myFn() {
      // 这样不会发生变化 因为是监听的.value的变化
      // state.value.a = '1',
      // state.value.gf.b = '2',
      // state.value.gf.f.c = '3',
      state.value.gf.f.s.d = "4";
      // 注意点： Vue3只提供了triggerRef方法，没有提供triggerReactive方法
      // 所以如果是reactive类型的数据，那么是无法主动触发界面更新的
      triggerRef(state); // 这样就能更新界面

      state.value = {
        a: "1",
        gf: {
          b: "2",
          f: {
            c: "3",
            s: {
              d: "4",
            },
          },
        },
      };

      console.log(state.value);
      console.log(state.value.gf);
      console.log(state.value.gf.f);
      console.log(state.value.f.s);

      // 注意点：如果是通过shallowRef创建的数据
      // 那么Vue监听的是.value的变化，并不是第一层的变化
    }
  },
};
```

引用场景：

一般情况下我们使用`ref`和`reactive`即可

只有子啊需要监听的数据量比较大 的时候，我们才使用 shallowRef/shallowReactive

#### shallowRef 的本质

```js
export default {
  setup() {
    // ref -> reactive
    // ref(10) -> reactive({value: 10})
    // shallowRef -> shallowReactive
    // shallowRef(10) -> shallowReactive({value: 10})
    // 所以如果是通过shallowRef创建的数据，它监听的是.value的变化
    // 因为底层本质上value才是第一层

    let state = shallowRef({
      a: "a",
      gf: {
        b: "b",
        f: {
          c: "c",
          s: {
            d: "d",
          },
        },
      },
    });
    let state2 = shallowReactive({
      value: {
        a: "a",
        gf: {
          b: "b",
          f: {
            c: "c",
            s: {
              d: "d",
            },
          },
        },
      },
    });
  },
};
```

#### toRaw

```js
import { reactive } from "vue";
export default {
  setup() {
    let obj = { name: "lnj", aage: 18 };
    // ref/reactive数据类型的特点
    // 每次修改都会被追踪，都会更新UI界面，但是这样其实非常消耗性能
    // 所以如果我们有一些操作不需要追踪数据和更新UI界面，我们就可以通过toRaw方法拿到它的原始数据，
    // 对原始数据进行修改，这样不会被追踪不会更新UI界面，性能就好了
    let state = reactive(obj);
    let obj2 = toRaw(state);
    let state2 = ref(obj);
    // 注意点： 如果想通过toRaw拿到ref类型的原始数据（创建时传入的数据）
    //         那么久必须明确的告诉toRaw方法，要获取的是.value的值
    //         因为经过Vue处理后，.value中保持的才是当初创建时传入的原始数据
    let obj2 = toRaw(state.value);
    console.log(obj === obj2); // true
    console.log(obj === state); // false
    // 引用关系，state的本质是一个proxy对象，在这个proxy对象中引用了obj
    function myFn() {
      // 如果直接修改obj，那么是不发触发界面更新的
      // 只有通过包装之后的对象修改，才会触发界面的更新
      obj.name = "zs";
      console.log(obj);
      console.log(state);
    }
    return { state, myFn };
  },
};
```

toRaw 从 Reactive 或 Ref 中得到原始数据

##### toRaw 的作用

做一些不想被监听的事情(提升性能)

#### markRaw 让数据永远都不要被追踪，永远不变成相应式数据

```js
import { markRaw } from 'vue'
export default {
    setup() {
        let obj = { name: 'lnj', age: 18}
        obj = markRaw(obj)
        // 这下再追踪都没有用了
        let state = reactive(obj)

        function myFn() {
            state.name = 'zs'
        }
        retuen { state, myFn }
    }
}
```

#### toRef

```js
import  { ref， toRef } from 'vue'
export default {
    name: 'App',
    setup() {
        let obj = { name: 'lnj'}
        /*
        	ref(obj.name) -> ref(lnj) -> reactive(value: lnj)
        */
        // ref 复制
        //let state = ref(obj.name)
        // toRef 引用
        /*
        	ref和toRef的区别：
        	ref -> 复制，修改响应式数据不会影响之前的数据
        	toRef -> 引用，修改响应式数据会影响之前的数据
        	ref -> 数据发生改变， 界面就会自动更新
        	toRef -> 数据发生改变，界面也不会自动更新
        */
        // toRef应用场景：
        // 如果想让响应式数据和以前的数据关联起来，并且更新响应式数据之后还不想更新UI界面，那么就可以使用
        let state = toRef(obj, 'name')

        function myFn() {
            state.value = 'zs'
            /*
            	结论：如果利用ref将某一个对象中的属性变成响应式的数据
            		 修改响应式的数据是不会影响到原始数据的变化
            */

             /*
            	结论：如果利用toRef将某一个对象中的属性变成响应式的数据
            		 修改响应式的数据是会影响到原始数据的变化
            		 但是如果响应式的数据是通过toRef创建的，那么修改了数据并不会触发UI界面的更新
            */
            console.log(obj)    // {name: 'lnj'}
            console.log(state)  // RefImpl{_rawValue: 'zs', _shallow: false, __v_isRef: true,_value: 'zs'}
        }
        return { state, myFn }
    }
}
```

#### toRefs

```js
import { toRefs } from "vue";
export default {
  name: "App",
  setup() {
    let obj = { name: "lnj", age: 18 };
    // let name = toRef(obj,'name')
    // let age = toRef(obj, 'age')

    let state = toRefs(obj);

    function myFn() {
      state.name.value = "zs";
      state.age.value = 666;
    }
    return { state, myFn };
  },
};
```

#### customRef

返回一个 ref 对象，可以显示地控制追踪和触发响应

```js
import { customRef } from "vue";

function myRef(value) {
  return customRef((track, trigger) => {
    return {
      get() {
        track(); // 告诉vue这个数据是需要追踪变化的
        return value;
      },
      set(newValue) {
        console.log("set", value);
        value = newValue;
        trigger(); //告诉vue触发界面更新 不加这个不会自动更新界面
      },
    };
  });
}
export default {
  name: "App",
  setup() {
    let age = myRef(18);
    function myFn() {
      age.value += 1;
    }
    return { age, myFn };
  },
};
```

```js
function myRef(value) {
	return customRef((track, trigger) => {
        fetch(value)
            .then(res => {
                return res.json()
            })
            .then(data => {
                value = data
                trigger()
            })
            .catch(err => {
                console.log(err)
            })
        return {
            get() {
                track()
                // 注意点：
                // 不能再get方法肿发送网络请求
                // 渲染界面 -> 调用get -> 发送网络请求
                // 保存数据 -> 更新界面 -> 调用get
                return value
            }
            set(newValue) {
                value = newValue
                trigger()
            }
        }
    })
}

export default {
    name: 'App',
    setup() {
        let state = ref([])
        // 这样写回调函数太多 不好
        //fetch('../public/data.json')
        //	.then(res => {
        //       return res.josn()
        //    })
        //	.then(data => {
        //    	state.value = data
        //	})
        //	.catch(err => {
        //    	console.log(err)
        //	})
        let state = myRef('../public/data.json')
        return { state }
    }
}
```

#### 获取元素

在 vue2.x 中我们可以通过给元素添加 ref='xxxx'

然后在代码中通过 refs.xxx 的方式来获取元素

在 vue3.x 中我们也可以通过 ref 来获取元素

```js
<template>
  <div ref="box">我是div</div>
</template>;
import { ref, onMounted } from "vue";
export default {
  name: "App",
  // 先赋值然后暴露出去就能在onMounted生命周期中获取到
  setup() {
    let box = ref(null); // reactive({value: null})
    console.log(box.value); //null
    onMounted(() => {
      console.log(box.value);
    });
    return { box };
  },
};
```

#### readonly

```js
import { readonly, isReadonly, shallowReadonlu } from 'vue'
setup() {
    // readonly用于创建一个只读的数据，并且是递归只读（每一层都是只读的）
    let state = readonly({name: 'lnj',attr:{age: 18,height: 1.88}})
    funtion myFn() {
        state.name = 'zs'
        state.attr.age = 666
        console.log(state)
    }

    // shallowReadonly只有第一层是只读的
    // const和readonly的区别：
    // const：赋值保护，不能给变量重新赋值
    // readonly: 属性保护，不能给属性重新赋值
    let state = shallowReadonly({name: 'lnj',attr:{age: 18,height: 1.88}})
    const value = 123
    const value1 = {name: 'zs', age: 124}
    funtion myFn() {
        state.name = 'zs'
        state.attr.age = 666
        state.attr.height: 1.66
        console.log(state)
        // 判断数据是否是只读的
        console.log(isReadonly(state))
        value = 456
        value1.name = 'ls'
        value1.age = 456
        console.log(value)  // 直接报错
        console.log(value1) // 更改成功
    }

```

#### vue3 中是通过 Proxy 来实现响应式数据的

```
let obj = { name: 'zs', age: 18 }
let state = new Proxy(obj, {
   get(obj, key) {
    	console.log(obj,key) // {name: 'zs', age: 18} name
    	return obj[key]
	}
	set(obj, key, value) {
        console.log(obj,key,value) // { name: 'zs',age: 18 } name 知播渔
        obj[key] = value // 更新UI界面
    }
})
console.log(state.name) // zs
state.name = '知播渔'
console.log(state) //{ name: '知播渔', age: 18 }
```

#### 注意点： set 方法必须通过返回值告诉 Proxy 此次操作是否成功

```js
let arr = [1,3,5]
let state = new Proxy(arr, {
    get(obj, key) {
        console.log(obj,key) // [1,3,5] 1
        return obj[key] // 3
    }
    set(obj,key,value) {
    	console.log(obj,key,value) // [1,3,5] 3 7
    	obj[key] = value
   		return true // [1,3,5,7] length 4
    	// 注意更改数组必须写上这一步 因为数组需要两步修改 第一步是插入数据 第二步更改数组长度 这一步是告诉它上一步已经完成可以进行下一步操作了 否则就可能会在控制台报错
	}
})
console.log(state[1])
state.push(7)
```

转载请注明原地址，潘柏信的博客：[http://51wangping.github.io](http://51wangping.github.io) 谢谢！
