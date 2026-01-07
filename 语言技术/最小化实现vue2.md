# 最小化实现Vue2

DEMO: [jsPlayGround](https://playcode.io/html-playground--019b8d26-ebec-7188-91c4-116c7025efd9)

## 第一步：构建 Vue 入口类与数据代理 (Data Proxy)

在 Vue 中，我们通常这样初始化：
```javascript
const vm = new Vue({
    data: { msg: 'Hello' }
});
console.log(vm.msg); // 我们直接通过 vm.msg 取值，而不是 vm.$data.msg
```
这一步的目标是建立 `Vue` 类，并实现让实例直接访问 `data` 中的属性（即数据代理）。

### 核心要点
1.  **保存配置**：将 `el` 和 `data` 保存到实例上（`$el`, `$data`）。
2.  **数据代理**：使用 `Object.defineProperty` 将访问 `this.xxx` 转发到 `this.$data.xxx`。

### 代码实现

创建一个 JS 文件（或直接在 HTML 的 script 标签中）：

```javascript
class Vue {
    constructor(options) {
        // 1. 保存选项
        this.$el = document.querySelector(options.el);
        this.$data = options.data;

        // 2. 数据代理：把 data 里的属性挂载到 Vue 实例(this)上
        this._proxyData(this.$data);
    }

    _proxyData(data) {
        // 遍历 data 中的所有 key
        Object.keys(data).forEach(key => {
            // 给 Vue 实例定义属性
            Object.defineProperty(this, key, {
                enumerable: true,
                configurable: true,
                // 当读取 vm.msg 时，从 vm.$data.msg 读取
                get() {
                    return data[key];
                },
                // 当修改 vm.msg 时，修改 vm.$data.msg
                set(newValue) {
                    data[key] = newValue;
                }
            });
        });
    }
}
```

### 验证测试 (期望产出)

请创建一个 HTML 文件运行以下代码：

```html
<div id="app"></div>
<script>
    // 这里粘贴上面的 Vue 类代码 //

    const vm = new Vue({
        el: '#app',
        data: {
            msg: 'Hello World',
            count: 100
        }
    });

    console.log('原始 data:', vm.$data.msg); // 输出: Hello World
    console.log('代理访问:', vm.msg);       // 输出: Hello World
    
    vm.msg = 'New Value';
    console.log('修改后 data:', vm.$data.msg); // 输出: New Value
</script>
```

**期望产出：**
控制台应该能正常打印出数据。当你修改 `vm.msg` 时，`vm.$data.msg` 应该同步变化。这说明最外层的壳子和便利访问机制已经建立好了。



## 第二步：数据劫持 (Observer)

**这一步的理由：**
第一步我们只是做了“搬运工”（代理），方便我们访问数据。但 Vue 的核心在于：**当数据变化时，视图要自动更新**。
为了做到这一点，我们需要给 `data` 中的每一个属性都安装一个“监控器”。在 ES5 中，必须使用 `Object.defineProperty` 来重写对象的 getter 和 setter。

这一步的目标是：**把普通的 data 对象，变成“响应式”的对象**。

### 核心要点
1.  **递归遍历**：因为 data 可能是嵌套对象（如 `{ user: { name: 'jack' } }`），我们需要递归到底。
2.  **重写 Getter/Setter**：
    *   `get`：后续将在这里进行“依赖收集”（记录谁用了这个数据）。
    *   `set`：后续将在这里进行“派发更新”（通知视图刷新）。目前我们先用 `console.log` 代替。
3.  **集成**：在 Vue 初始化时调用这个逻辑。

### 代码实现

请在之前的 `Vue` 类上方，添加一个 `Observer` 类，并更新 `Vue` 类。

```javascript
// 1. 新增 Observer 类
class Observer {
    constructor(data) {
        this.observe(data);
    }

    // 遍历对象处理
    observe(data) {
        // 如果 data 不存在或者不是对象，就停止递归
        if (!data || typeof data !== 'object') {
            return;
        }

        // 获取对象的所有 key，遍历劫持
        Object.keys(data).forEach(key => {
            this.defineReactive(data, key, data[key]);
        });
    }

    // 定义响应式 (核心方法)
    defineReactive(obj, key, value) {
        // 递归：如果 value 还是个对象 (例如 user: {name: '...'})，需要继续劫持
        this.observe(value);

        const self = this; // 保存上下文

        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            
            get() {
                // 目前暂时只返回原值
                // 【下一阶段目标：在这里收集依赖】
                return value;
            },
            
            set(newValue) {
                if (newValue !== value) {
                    // 如果新设置的值是对象，也需要把它变成响应式的
                    self.observe(newValue); 
                    
                    value = newValue;
                    
                    // 【下一阶段目标：在这里通知视图更新】
                    console.log(`监测到属性 [${key}] 发生了变化，新值为:`, newValue);
                }
            }
        });
    }
}

// 2. 更新 Vue 类
class Vue {
    constructor(options) {
        this.$el = document.querySelector(options.el);
        this.$data = options.data;

        // 这里顺序很重要：
        // 先把数据变成响应式的 (Observer)
        new Observer(this.$data);

        // 再做代理 (Proxy)，这样访问 this.msg 触发 set 时，
        // 实际上会触发 Observer 里定义的 set
        this._proxyData(this.$data);
    }

    _proxyData(data) {
        Object.keys(data).forEach(key => {
            Object.defineProperty(this, key, {
                enumerable: true,
                configurable: true,
                get() { return data[key]; },
                set(newValue) { data[key] = newValue; }
            });
        });
    }
}
```

### 验证测试 (期望产出)

请更新你的测试代码：

```javascript
const vm = new Vue({
    el: '#app',
    data: {
        msg: 'Hello',
        user: {
            name: 'Jack' // 测试嵌套对象
        }
    }
});

console.log('--- 测试开始 ---');
// 1. 修改基本属性
vm.msg = 'World'; 
// 期望控制台输出: 监测到属性 [msg] 发生了变化，新值为: World

// 2. 修改嵌套属性
vm.user.name = 'Tom';
// 期望控制台输出: 监测到属性 [name] 发生了变化，新值为: Tom

// 3. 修改值为新对象
vm.user = { name: 'Jerry' };
// 期望控制台输出: 监测到属性 [user] 发生了变化...

// 4. 验证新对象是否也是响应式的 (Observer 里的 self.observe(newValue) 起作用)
vm.user.name = 'Spike';
// 期望控制台输出: 监测到属性 [name] 发生了变化...
```

**期望产出：**
当你修改数据时，控制台能打印出日志。这证明我们已经成功“劫持”了数据的修改操作。

此时,你发现这两步都是使用`defineProperty`来劫持对象,但是两者的用处不同.

第一步

*   **Observer (仓库管理员)**：
    *   操作对象是：`data` 对象（以及内部嵌套的对象）。
    *   它的工作是：深入仓库内部，给每一个货物（属性）都安装报警器。
    *   **它是核心**：没有它，数据变了 Vue 根本不知道。
第二步

*   **_proxyData (前台接待)**：
    *   操作对象是：`vm` (Vue 实例本身)。
    *   它的工作是：当你找前台要 `msg` 时，前台转身去仓库 `data` 里把 `msg` 拿给你。
    *   **它是语法糖**：没有它，Vue 也能跑，但是你写代码会很痛苦。
*   
## 第三步：模板编译 (Compiler) —— 显示数据

**这一步的理由：**
现在数据已经是响应式的了（改数据会打印 log），但页面上还是空的，或者是写死的 HTML。
我们需要一个“翻译官”，把 HTML 里的 `{{ msg }}` 和 `v-model="msg"` 替换成 `data` 里真实的数据。

这一步的目标是：**解析 DOM 节点，初始化页面的显示（暂时还不做自动更新，只做第一次渲染）。**

### 核心要点
1.  **文档碎片 (DocumentFragment)**：直接操作 DOM 会频繁引起页面回流，性能很差。我们会把 DOM 节点全部移入内存中的“文档碎片”里，处理完再一次性贴回页面。
2.  **递归遍历 DOM**：节点也是树状结构，需要递归检查每一个节点。
3.  **区分节点类型**：
    *   **元素节点** (nodeType === 1)：检查有没有 `v-model` 这种指令。
    *   **文本节点** (nodeType === 3)：检查有没有 `{{ }}` 这种插值。

### 代码实现

新增一个 `Compiler` 类。

```javascript
class Compiler {
    constructor(el, vm) {
        this.el = typeof el === 'string' ? document.querySelector(el) : el;
        this.vm = vm;

        // 1. 把 DOM 节点移入内存片段
        if (this.el) {
            let fragment = this.nodeToFragment(this.el);
            
            // 2. 编译片段 (处理 {{}} 和 v-model)
            this.compile(fragment);
            
            // 3. 把编译好的片段放回页面
            this.el.appendChild(fragment);
        }
    }

    // --- 核心逻辑 1: 移动节点 ---
    nodeToFragment(el) {
        let fragment = document.createDocumentFragment();
        let firstChild;
        // 循环把 el 的第一个子节点吸走，直到 el 空了为止
        while (firstChild = el.firstChild) {
            fragment.appendChild(firstChild);
        }
        return fragment;
    }

    // --- 核心逻辑 2: 编译 ---
    compile(node) {
        let childNodes = node.childNodes; // 获取当前层级的所有子节点
        
        // 类数组转数组后遍历
        [...childNodes].forEach(child => {
            if (this.isElementNode(child)) {
                // 如果是元素 (div, input, p)，编译指令
                this.compileElement(child);
                // 递归深入
                this.compile(child);
            } else {
                // 如果是文本，编译插值 {{}}
                this.compileText(child);
            }
        });
    }

    // 编译元素节点 (处理 v-model)
    compileElement(node) {
        let attributes = node.attributes;
        [...attributes].forEach(attr => {
            let { name, value } = attr; // name="v-model", value="msg"
            
            // 判断是否是 v- 开头的指令
            if (this.isDirective(name)) {
                let [, directive] = name.split('-'); // 拿到 model
                
                if (directive === 'model') {
                    // 从 vm.$data 中取值，放到 node.value 中
                    node.value = this.getVMValue(this.vm, value);
                }
            }
        });
    }

    // 编译文本节点 (处理 {{ msg }})
    compileText(node) {
        let content = node.textContent;
        // 正则匹配 {{ xxx }}
        if (/\{\{(.+?)\}\}/.test(content)) {
            // RegExp.$1 是匹配到的内容 (例如 "msg" 或 "user.name")
            let expr = RegExp.$1.trim();
            
            // 从 vm.$data 中取值，替换文本内容
            // 注意：这里简化处理，假设文本节点里只有 {{}}，没有其他字符
            node.textContent = this.getVMValue(this.vm, expr);
        }
    }

    // --- 辅助工具 ---
    
    // 判断是否元素节点
    isElementNode(node) { return node.nodeType === 1; }
    
    // 判断是否指令
    isDirective(name) { return name.startsWith('v-'); }
    
    // 获取 data 值 (解决 user.name 这种嵌套取值)
    getVMValue(vm, expr) {
        let data = vm.$data;
        expr.split('.').forEach(key => {
            data = data[key];
        });
        return data;
    }
}
```

**别忘了更新 Vue 类：**
在 `Vue` 的构造函数最后一行，启动编译器。

```javascript
class Vue {
    constructor(options) {
        // ... 前面的代码不变 ...
        this.$el = document.querySelector(options.el);
        this.$data = options.data;
        new Observer(this.$data);
        this._proxyData(this.$data);

        // 新增：编译模板
        new Compiler(this.$el, this);
    }
    // ...
}
```

### 验证测试 (期望产出)

请更新你的 HTML：

```html
<div id="app">
    <!-- 测试文本插值 -->
    <p>{{ msg }}</p> 
    <!-- 测试嵌套插值 -->
    <p>{{ user.name }}</p>
    <!-- 测试指令 -->
    <input type="text" v-model="msg">
</div>

<script>
    // ... 你的 JS 代码 ...
    const vm = new Vue({
        el: '#app',
        data: {
            msg: 'Hello Compiler',
            user: { name: 'Jack' }
        }
    });
</script>
```

**期望产出：**
1.  打开页面，你应该不再看到 `{{ msg }}` 这种花括号，而是看到了真实的文字 "Hello Compiler"。
2.  Input 输入框里应该自动填入了 "Hello Compiler"。
3.  **注意：** 此时你修改输入框，或者在控制台修改 `vm.msg`，页面**不会**发生变化（因为还没做连接），这是正常的。

简化过程,第三步其实就是这三个步骤:

1. 搬家: 将真实DOM全部转为文档片段
2. 遍历文档片段,解析并放入真实数据
3. 将文档片段塞回原本的真实DOM中


## 第四步：订阅发布模式 (Watcher & Dep) —— 连接数据与视图

**这一步的理由：**
目前我们有了“能监控数据的 Observer”和“能修改视图的 Compiler”，但它们俩是割裂的。
*   数据变了，Observer 知道，但不知道该去改页面上的哪个节点。
*   Compiler 知道节点在哪里，但只在初始化时执行了一次。

我们需要一个 **中间人** 和一个 **通讯录**：
*   **Watcher (订阅者/中间人)**：每一个 `{{ msg }}` 或 `v-model` 对应一个 Watcher。它知道怎么去更新那一个具体的 DOM 节点。
*   **Dep (通讯录/发布者)**：每一个 data 属性（如 `msg`）都有一个专属的 Dep。它记录了所有依赖这个属性的 Watcher。

这一步的目标是：**建立“数据 -> Dep -> Watcher -> 视图”的更新链路。**

### 核心要点 (最抽象的部分)

1.  **Dep 类**：就是一个数组，用来存 Watcher。
2.  **Watcher 类**：
    *   在初始化时，要把自己“塞进”对应属性的 Dep 里。**怎么塞？**
    *   **技巧**：Watcher 初始化时会故意去读取一次数据（触发 `get`）。
    *   在 `get` 触发前，把当前 Watcher 挂在一个全局变量 `Dep.target` 上。
    *   Observer 的 `get` 只要看到 `Dep.target` 有人，就把他加到通讯录里。

### 代码实现

先定义 **Dep** 和 **Watcher** 类，然后修改 **Observer** 来使用它们。

**1. Dep 类 (通讯录)**

```javascript
class Dep {
    constructor() {
        this.subs = []; // 存放 Watcher 的数组
    }

    // 收集订阅者
    addSub(watcher) {
        this.subs.push(watcher);
    }

    // 通知更新
    notify() {
        this.subs.forEach(watcher => watcher.update());
    }
}
```

**2. Watcher 类 (订阅者)**

```javascript
class Watcher {
    // vm: Vue实例, expr: 数据key(如'msg'), cb: 回调函数(更新DOM用)
    constructor(vm, expr, cb) {
        this.vm = vm;
        this.expr = expr;
        this.cb = cb;

        // --- 核心逻辑：触发依赖收集 ---
        // 1. 把当前 watcher 挂载到 Dep 类的静态属性上
        Dep.target = this;
        
        // 2. 读取一下数据。这一步会强行触发 Observer 里的 get()
        // 因为 get() 里我们会写逻辑：如果你是 Dep.target，我就把你存起来
        this.oldValue = this.getVMValue(vm, expr);
        
        // 3. 收集完毕，清空 target，防止不相干的数据也收集了这个 watcher
        Dep.target = null;
    }

    // 当数据变化时，Dep 会调用这个方法
    update() {
        let newValue = this.getVMValue(this.vm, this.expr);
        let oldValue = this.oldValue;

        if (newValue !== oldValue) {
            this.cb(newValue); // 调用 Compiler 传进来的回调，更新视图
        }
    }

    getVMValue(vm, expr) {
        let data = vm.$data;
        expr.split('.').forEach(key => data = data[key]);
        return data;
    }
}
```

**3. 修改 Observer (安装通讯录)**

回到 `Observer` 类，我们需要在 `defineReactive` 里实例化 `Dep`，并在 `get` 和 `set` 里使用它。

```javascript
// ... Observer 类内部 ...
    defineReactive(obj, key, value) {
        this.observe(value);
        
        const dep = new Dep(); // 【新增】每个属性都有一个专属的 dep

        Object.defineProperty(obj, key, {
            enumerable: true,
            configurable: true,
            get() {
                // 【新增】如果 Dep.target 有值，说明是 Watcher 正在读取，赶紧把它收集起来
                if (Dep.target) {
                    dep.addSub(Dep.target);
                }
                return value;
            },
            set(newValue) {
                if (newValue !== value) {
                    // self.observe(newValue); // 注意上下文
                    // 这里为了简化代码展示，省略了 observe 调用和 self 缓存
                    // 实际代码中请保留之前的 observe 逻辑
                    
                    value = newValue;
                    
                    // 【新增】数据变了，通知 dep 里所有的 watcher 干活
                    console.log(`属性 ${key} 更新，通知 Watcher 更新视图`);
                    dep.notify();
                }
            }
        });
    }
```

**4. 修改 Compiler (创建订阅者)**

现在“机关”都设好了，我们需要在编译模板时，给每一个节点注册 Watcher。

```javascript
// ... Compiler 类内部 ...

    // 1. 修改 compileText (处理 {{ }})
    compileText(node) {
        let content = node.textContent;
        if (/\{\{(.+?)\}\}/.test(content)) {
            let expr = RegExp.$1.trim();
            
            // A. 初始化显示 (保留)
            node.textContent = this.getVMValue(this.vm, expr);

            // B. 【新增】创建一个 Watcher
            // 当数据变了，这个回调函数会被执行
            new Watcher(this.vm, expr, (newValue) => {
                node.textContent = newValue;
            });
        }
    }

    // 2. 修改 compileElement (处理 v-model)
    compileElement(node) {
        // ... 前面的属性遍历逻辑 ...
        if (directive === 'model') {
             // A. 初始化显示 (保留)
             node.value = this.getVMValue(this.vm, value);

             // B. 【新增】创建一个 Watcher
             new Watcher(this.vm, value, (newValue) => {
                 node.value = newValue;
             });
             
             // ... C. 双向绑定监听 input 事件 (下一步再做) ...
        }
        // ...
    }
```

### 验证测试 (期望产出)

请运行之前的 HTML 页面。

1.  打开控制台 (Console)。
2.  输入 `vm.msg = 'Vue is awesome'`。
3.  **期望产出**：
    *   控制台打印："属性 msg 更新，通知 Watcher 更新视图"。
    *   **页面变化**：页面上的文字 `<p>Hello Compiler</p>` 瞬间变成了 `<p>Vue is awesome</p>`。
    *   **输入框变化**：输入框里的值也变成了 `Vue is awesome`。

## 第五步：双向绑定 (v-model) 与最终完善

**这一步的理由：**
目前我们的链路是单向的：`Data` -> `View`。
当我们手动修改 `vm.msg` 时，页面会变。但是，当我们在页面的 `<input>` 输入框里打字时，`vm.msg` 数据并不会跟着变。

Vue 的 `v-model` 其实就是 **单向绑定 + 事件监听** 的语法糖。
这一步的目标是：**监听输入框的 `input` 事件，当用户输入时，反向更新 `data` 中的数据。**

### 核心要点
1.  **监听事件**：在 Compiler 解析 `v-model` 时，给 DOM 节点添加 `input` 事件监听器。
2.  **设置数据**：在事件回调中，把 input 的 `value` 赋值给 `vm` 的数据。
3.  **闭环**：赋值操作触发 `set` -> 触发 `dep.notify()` -> 触发其他依赖该数据的 `Watcher` 更新（比如页面上的文本节点）。

### 代码实现

我们需要修改 **Compiler** 类的 `compileElement` 方法。

```javascript
// ... Compiler 类内部 ...

    compileElement(node) {
        let attributes = node.attributes;
        [...attributes].forEach(attr => {
            let { name, value: expr } = attr; // name="v-model", expr="msg"
            
            if (this.isDirective(name)) {
                let [, directive] = name.split('-'); 
                
                if (directive === 'model') {
                    // 1. (已有的) 数据 -> 视图
                    node.value = this.getVMValue(this.vm, expr);
                    new Watcher(this.vm, expr, (newValue) => {
                        node.value = newValue;
                    });

                    // 2. 【新增】视图 -> 数据
                    node.addEventListener('input', (e) => {
                        let newValue = e.target.value;
                        // 将界面产生的新值，写回 data
                        // 注意：这里需要一个辅助方法 setVMValue 来处理 user.name 这种层级
                        this.setVMValue(this.vm, value, newValue);
                    });
                }
            }
        });
    }

    // 【新增】辅助方法：设置 data 值 (处理 user.name)
    setVMValue(vm, expr, value) {
        let data = vm.$data;
        let arr = expr.split('.');
        
        arr.forEach((key, index) => {
            // 如果还没到最后一层 (例如 user.name 的 user 层)
            if (index < arr.length - 1) {
                data = data[key];
            } else {
                // 到了最后一层，赋值
                data[key] = value;
            }
        });
    }
```

### 最终验证测试 (期望产出)

请运行完整的 HTML 页面。

1.  **测试双向绑定**：
    *   在页面上的输入框里输入 "123"。
    *   **期望**：下方的文本 `<p>{{ msg }}</p>` 会实时变成 "123"。
    *   **原理**：Input -> Data (set触发) -> Dep.notify -> Watcher -> TextNode Update。

2.  **测试多处联动**：
    *   如果你在 HTML 里写两个输入框 `<input v-model="msg">`。
    *   **期望**：修改任意一个，另一个输入框和文本都会同步变化。
