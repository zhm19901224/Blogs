

##### TDD(Test Driven Development) 测试驱动开发
step 1. 先编写测试用例无法通过
step 2. 编写测试用例，此刻用例不能通过
step 3. 编写代码使测试用例通过
step 4. 优化代码，完成开发
特点:
1. 先写测试再写代码，测试重点在代码
2. 结合单元测试，速度快(shallow)
3. 安全感低，只能保证每个模块没问题，无法保证整体没问题
4. TDD适合组件库或者公用util、通用组件的测试
优点：开发周期长，可减少回归bug，代码维护性更好。先写用例，测试覆盖率比较高。


##### BDD(Behavior Driven Development) 行为驱动开发
step 1. 先正常开发;
step 2. 根据用户行为，编写测试用例
特点:
1. 先写代码再测试，测试重点在UI
2. 结合集成测试，白盒测试
3. 安全感高，速度慢(mount)
4. BDD集成测试适合做业务代码的自动化测试
##### 1. jest基本语法
```
/*
package.json 配置
  "scripts": {
    "test": "jest --watch"
  }
jest会执行test/*.test.js中的测试脚本，监听用例改变，如果用例改变了，会测试变更的用例，相当于o模式
*/
const { add } = require('./math')

test('test add', () => {
    expect(add(1, 2)).toBe(3)
})
```
所有测试的方法必须导出，为了方便单元测试(单个模块)和集成测试(多个模块)


##### 初始化Jest
npx jest --init， 生成jest.config.js

##### 生成代码覆盖率报告
npx jest --covarage
在命令行中显示代码覆盖率，也可以在covarage/icon-report/index.html中查看报告

##### 2. Jest匹配器machers
> 都可加not.前缀
1. toBe严格相等 **toEqual内容相等，如两个地址不一样，其余一样的对象**
2. toBeCloseTo **浮点数必须用它，可以处理舍入误差**
3. not.toBe 不等于
4. toBeNull
5. toBeUndefined
6. toBeDefined
7. toBeTruthy 隐士转换为true的
8. toBeFalsy
9. toBeGreaterThan 大于
10. toBeGreaterThanOrEqual 大于等于
11. toBeLessThan 小于
12. toBeLessThanOrEqual 小于等于
13. toContain **数组包含此成员、 字符串包含子串**
14. toMatch **是否匹配正则**


##### 3. 在VSCODE中使用code指令
按f 只测试刚刚没有通过的用例
按o 只测试修改过的相关的用例，**前提必须配置了git**



##### 4. 测试异步方法

> 基于Promise，在测试方法中必须将promise return
```
// request.test.js

import { getUserInfo } from '../src/api'

test('getUserInfo接口数据返回', () => {
    return getUserInfo().then(res => {
        expect(res.code).toBe(0)
    })
})

// api.js
import requeset from './utils/server';
export const getUserInfo = () => requeset('/api/getUserInfo', 'get', {aaa: 123});
```

> 利用async await，最佳方案
```
import { getData } from '../src/api'

test('getUserInfo接口数据返回', async () => {
    let data = await getData()
    expect(data).toBe('success')
})
```


##### 5. Jest hooks
1. beforeAll(() => {}) 所有用例执行前执行
2. afterAll(() => {}) 所有用例执行后执行
3. beforeEach(() => {}) 每个用例执行前执行
4. afterEach(() => {}) 每个用例执行后执行
5. describe('分组描述', () => { // ...test }) 对用例进行分组

> hook具有作用域，一个describe下的hook只对其describe内部的test有效果

##### 6. Jest Mock
> 用于生成mock函数，来捕获用例函数的调用情况
```
// 检测函数内部的回调函数是否被调用，调用了几次
function sleep(fn) {
    fn()
}

test('测试函数调用', () => {
    let mock = jest.fn();
    sleep(mock)
    expect(mock).toBeCalled()
    expect(mock.calls.length).toBe(1)
})
```

> 真正的项目中不会真的测试接口请求的数据，因为非常慢影响开发效率。只要测试这个请求正常发送了就可以了，至于接口返回那是后端测试的事情了
```
// 实际项目中异步测试这么测就可以了
import { getUserInfo } from '../src/api'

jest.mock('../src/api.js')

test('测试服务配置信息', async () => {
    getUserInfo.mockResolvedValue('success')
    let res =  await getUserInfo();
    expect(res).toBe('success')
})
```

> 假如api.js中有一个getParams方法不需要mock模拟, 但也需要测试
```
import { getUserInfo } from '../src/api'

jest.mock('../src/api.js')
const { getParams } = jest.requireActual('../src/api.js')
test('测试服务配置信息', async () => {
    getUserInfo.mockResolvedValue('success')
    let res =  await getUserInfo();
    expect(res).toBe('success')
})
```

> mock timers对js定时器的模拟
```
// useFakeTimers + runAllTimers + mock， mock会污染代码

function runTimer(cb){
    setTimeout(() => {
        // ... 业务代码
        cb() // 触发调用统计mock函数
    }, 1000)
}


jest.useFakeTimers()

test('测试定时器', () => {
    let mock = jest.fn()
    runTimer(mock)

    jest.runAllTimers()   // 让所有定时器不用等时间到，直接启动
    expect(mock).toHaveBeenCalledTimes(1)
})

```
##### 7. snapshot 快照测试
> 对于配置文件的测试，会生成缓存，当次通过测试，下次再次测试只要配置项内容不变，就会通过，如果发生改变不通过，可以按u，更新快照缓存。这种方式会让开发者进一步确认配置文件变化
```
const serverConfig = {
    hostname: 'localhost',
    port: 8081
}


test('测试服务配置信息', () => {
    expect(serverConfig).toMatchSnapshot()
})
```

> 如果有多个快照，要按i，一个一个进行交互提示，针对性的更新

> 如果遇到一直变化的配置，比如time
```
const serverConfig = {
    hostname: 'localhost',
    port: 8081,
    time: new Date()
}


test('测试服务配置信息', () => {
    expect(serverConfig).toMatchSnapshot({
        time: expect.any(Date)
    })
})
```


##### 8. 测试es6 class
> 测试复杂的类，**无需把每个方法跑一边，只需测试其实例中的所有方法可以正常调用就可以**

```
jest.mock('./util') // 将utils下的class进行模拟
import Util from './util'
test('测试类', () => {
    let u = new Util() // 没有真正实例化原来的类，只是实例化mock
    u.a();
    u.b();
    expect(Util).toHaveBeenCalled()

    expect(Util.mock.instances[0].a).toHaveBeenCalled()
    expect(Util.mock.instances[0].b).toHaveBeenCalled()
})

```

##### 9. React项目使用Enzyme进行TDD测试

1. jest.config.js通用配置。参考create-react-app, 先需要安装jest依赖
```
npm i jest-watch-typeahead babel-jest camelcase @testing-library/jest-dom
babel-preset-react-app
```
> jest配置文件解读
```
module.exports =  {
  "roots": [
    "<rootDir>/src"
  ],
  // 分析src下这四类文件生成代码覆盖率报告，忽略ts的类型定义文件
  "collectCoverageFrom": [
    "src/**/*.{js,jsx,ts,tsx}",         
    "!src/**/*.d.ts"                 
  ],
   // 测试之前，用jsdom做补偿
  "setupFiles": [
    "react-app-polyfill/jsdom"         
  ],
  // 在这个文件内可以做一些额外配置
  "setupFilesAfterEnv": [
    "<rootDir>/jestConfig/setupTests.js" 
  ],
  // npm run test时，会被认为是测试文件的文件名
  "testMatch": [                      
    "<rootDir>/src/**/__tests__/**/*.{js,jsx,ts,tsx}",  
    "<rootDir>/src/**/*.{spec,test}.{js,jsx,ts,tsx}"
  ],
  "testEnvironment": "jsdom",           
  
  "testRunner": "/Users/zhanghaomin/Desktop/jest-test-tdd/node_modules/jest-circus/runner.js",
  // 用babel-jest对脚本文件做转化，对css做忽略操作, 对静态资源，就当作字符串静态资源处理
  "transform": {
    "^.+\\.(js|jsx|mjs|cjs|ts|tsx)$": "<rootDir>/jestConfig/babelTransform.js",
    "^.+\\.css$": "<rootDir>/jestConfig/cssTransform.js",
    "^(?!.*\\.(js|jsx|mjs|cjs|ts|tsx|css|json)$)": "<rootDir>/jestConfig/fileTransform.js"
  },
  
  // transform忽略文件配置，node_modules的无需转化，因为都是打包好转化过的内容
  "transformIgnorePatterns": [
    "[/\\\\]node_modules[/\\\\].+\\.(js|jsx|mjs|cjs|ts|tsx)$",
    "^.+\\.module\\.(css|sass|scss)$"
  ],
  // 测试文件引入的模块默认在node_module中找，如果需要在其他文件夹中找，在此配置
  "modulePaths": [],
  // css使用css module时，类名有映射，方便测试
  "moduleNameMapper": {
    "^react-native$": "react-native-web",
    "^.+\\.module\\.(css|sass|scss)$": "identity-obj-proxy"
  },
  "moduleFileExtensions": [
    "js",
    "jsx",
    "ts",
    "tsx",
    "json",
    "node"
  ],
  // jest测试时，模式的插件
  "watchPlugins": [
    "jest-watch-typeahead/filename",
    "jest-watch-typeahead/testname"
  ],
  "resetMocks": true
};
```

2. Enzyme(因赞姆)的配置
> Enzyme提供了对ReactDom的包装，不仅可以测试页面dom，还可以测试组件内部的props
> 文档：https://github.com/enzymejs/enzyme

step1 安装相关包
```
// jest-enzyme是对jest匹配器的扩展
npm i --save-dev enzyme enzyme-adapter-react-16 jest-enzyme
```

step2 修改配置文件，扩展jest对jest-enzyme匹配器的支持
```
// jest.test.js
  "setupFilesAfterEnv": [
    "./node_modules/jest-enzyme/lib/index.js"
  ],
```

step3 在每个测试文件引入相关库
```
测试文件
import Enzyme, { shallow } from 'enzyme';
import Adapter from 'enzyme-adapter-react-16';
import React from 'react'
Enzyme.configure({ adapter: new Adapter() });
```
> 可以将部分代码抽离到setUpTests.js中去，在初始化的时候统一执行，这样不用没此时文件都写了
```
// setupTests.js
import Enzyme from 'enzyme';
import Adapter from 'enzyme-adapter-react-16';
Enzyme.configure({ adapter: new Adapter() });
```

3. Enzyme常用方法
三种渲染方式:
shallow, mount, render。

    shallow渲染叫浅渲染，仅仅对当前jsx结构内的顶级组件进行渲染，而不对这些组件的内部子组件进行渲染，因此，它的性能上最快的，大部分情况下，如果不深入组件内部测试，那么可以使用shallow渲染。

    mount则会进行完整渲染，而且完全依赖DOM API，也就是说mount渲染的结果和浏览器渲染结果说一样的，结合jsdom这个工具，可以对上面提到的有内部子组件实现复杂交互功能的组件进行测试。

    render也会进行完整渲染，但不依赖DOM API，而是渲染成HTML结构，并利用cheerio实现html节点的选择，它相当于只调用了组件的render方法，得到jsx并转码为html，所以组件的生命周期方法内的逻辑都测试不到，所以render常常只用来测试一些数据（结构）一致性对比的场景。在这里还提到，shallow实际上也测试不到componentDidMount/componentDidUpdate这两个方法内的逻辑


思考：
1. 写UT这么麻烦，付出大量人力物力成本，为何还要去写？开发周期长的大型项目进行回归测试的时候会节省很多人力成本。而且当一个新员工接触别人写的代码的时候，在做修改时不知道会不会对其他模块产生副作用，这时候通过UT去检验单个公用模块，安全保险！

2. 在日常业务线的开发中，我觉得测试金字塔中，ui前端处于最顶端，越是底层越需要进行UT，顶层前端业务代码其实没那么大必要。反倒集成测试是十分有必要的，自动模拟用户行为，很大程度比自己各种瞎操作测试要安全的多。