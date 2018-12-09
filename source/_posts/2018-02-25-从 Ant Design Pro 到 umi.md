---
urlname: from-ant-design-pro-to-umi
title: 从 Ant Design Pro 到 umi
date: 2018-2-25
author: 空谷
header-img: http://pics.arvinx.com/2018-02-25-133024.jpg
tags:  
  - 前端
  - 框架
---

因为心想着让毕设真正能够落地，过年放假这段时间一直在研究 Ant Design Pro 的源码，力图寻找一种对新人友好、开发体验优良的模式。然后从 pro 研究到 roadhog 与 dva 再从 roadhog 与dva 研究到 umi ，总算有些满意的结果。在此进行一下分析与总结。

本文的顺序便是基于前端开发必会涉及几个环节的来梳理 Ant Design Pro 、 umi 的开发逻辑。

从这几块来讲：组织架构、app 启动、路由、布局、页面、数据模型、mock。

## 组织架构

先来看看 Ant Design Pro （下称 pro ）的官方提供的文件组织结构：

```
├── mock                     # 本地模拟数据
├── public                   # 静态资源
│   └── favicon.png          # Favicon
├── src
│   ├── assets               # 本地静态资源
│   ├── common               # 应用公用配置，如导航信息
│   ├── components           # 业务通用组件
│   ├── e2e                  # 集成测试用例
│   ├── layouts              # 通用布局
│   ├── models               # dva model
│   ├── routes               # 业务页面入口和常用模板
│   ├── services             # 后台接口服务
│   ├── utils                # 工具库
│   ├── g2.js                # 可视化图形配置
│   ├── theme.js             # 主题配置
│   ├── index.ejs            # HTML 入口模板
│   ├── index.js             # 应用入口
│   ├── index.less           # 全局样式
│   └── router.js            # 路由入口
├── tests                    # 测试工具
└── package.json
```

如果将抛开 model 、 components 、services 这些业务组件，组成一个最小可运行的页面，最关键的内容如下：

```
├── src
│   ├── common               # 应用公用配置，如路由信息、菜单信息
│   ├── layouts              # 通用布局
│   ├── routes               # 业务页面入口和常用模板
│   ├── index.ejs            # HTML 入口模板
│   ├── index.js             # 应用入口
│   └── router.js            # 路由入口
└── package.json
```

再来看看 umi 的最核心的目录结构。

```
├── src                           // 源码目录，可选，把里面的内容直接移到外面即可
    ├── layouts/
    │   └── index.js               // 全局布局
    ├── pages/                     // 页面目录，里面的文件即路由
        ├── .umi/                  // dev 临时目录，需添加到 .gitignore
        ├── document.ejs           // HTML 模板
        └── index.js               // 页面 1
    ├── global.css                 // 约定的全局样式文件，自动引入，也可以用 global.less
    ├── (_routes.json)             // 路由配置，和文件路由二选一
├── .umirc.js                      // umi 配置
├── .webpackrc                     // webpack 配置
└── package.json
```

如果大家仔细看的话umi 与 pro 的核心目录结构其实基本一致。

`layout` 文件夹控制全局布局，`routes` 在 pro 中是具体的业务界面，而在 umi 中则是改用 `pages` 这个词。这对于使用者来说会更加直观。（ `routes` 这样的词其实并不易于理解。我一开始用 pro 的时候花了老半天才理解清楚 routes 到底意味着什么。）

但是粗看的话 umi 比 pro 少了不少文件，比如入口的以下这两个文件：

```
├── src
│   ├── index.js             # 应用入口
│   └── router.js            # 路由入口
```

那么这两个文件到底是干什么的？就要进入第二部分「启动方式」细说。

## 启动方式

pro 中 `index.js` 是整个 app 的控制文件，采用 `dva` 的写法。最小可用代码为：

```js
import dva from 'dva';
import createHistory from 'history/createHashHistory';

const app = dva({
  history: createHistory(),
});

app.model(require('./models/global').default);

app.router(require('./router').default);

app.start('#root');
```

而这个启动 app 的重要文件在 umi 中为什么看似会没有呢？实际上，这个文件隐藏在了 `.umi` 这个临时文件夹下

```
// .umi 的目录结构
├── .umi
│   ├── umi.js             
│   └── router.js
│   └── registerServiceWorker.js
```

如果打开 umi.js

```js
// pages/.umi/umi.js

import React from 'react';
import ReactDOM from 'react-dom';
import createHistory from 'umi/_createHistory';
import FastClick from 'umi-fastclick';


document.addEventListener(
  'DOMContentLoaded',
  () => {
    FastClick.attach(document.body);
  },
  false,
);

// create history
window.g_history = createHistory({
  basename: window.routerBase,
});


// render
function render() {
  ReactDOM.render(React.createElement(require('./router').default), document.getElementById('root'));
}
render();

// hot module replacement
if (module.hot) {
  module.hot.accept('./router', () => {
    render();
  });
}

if (process.env.NODE_ENV === 'development') {
  window.g_history.listen(function(location) {
    new Image().src = (window.routerBase + location.pathname).replace(/\/\//g, '/');
  });
}

// Enable service worker
if (process.env.NODE_ENV === 'production') {
  require('./registerServiceWorker');
}
```

可以看到，这两个文件的思路基本保持一致，创建 history、model（后文会有创建 model 的版本） 再引入 router, 最后启动 app。而 umi 采用了更加原生的方式来启动 app ，而不是之前那样直接封死到 dva ，这保证了 app 启动时的扩展性。就如下文会提到的 umi-plugin-dva 。

那 router 是什么？它有什么用？让我们往下看到第三部分：路由。


## 路由

先来看看 pro 的 `src` 目录下的路由是干什么的。删掉个别不大相关的代码后，`router.js`  的代码如下

```jsx
import React from 'react';
import { routerRedux, Route, Switch } from 'dva/router';


import { getRouterData } from './common/router';
import Authorized from './utils/Authorized';
import styles from './index.less';

const { ConnectedRouter } = routerRedux;
const { AuthorizedRoute } = Authorized;


function RouterConfig({ history, app }) {
  const routerData = getRouterData(app);
  const UserLayout = routerData['/user'].component;
  const BasicLayout = routerData['/'].component;
  return (
      <ConnectedRouter history={history}>
        <Switch>
          <Route path="/user" component={UserLayout} />
          <AuthorizedRoute
            path="/"
            render={props => <BasicLayout {...props} />}
            authority={['admin', 'user']}
            redirectPath="/user/login"
          />
        </Switch>
      </ConnectedRouter>
  );
}

export default RouterConfig;
```

总体来说有以下几个特征：

1. 传出一个整体路由控制的配置函数（给 dva 加载路由）；
2. 利用 `getRouterData` 获得目前 app 的路由数据 `routerData`；
3. 利用 `routerData` 中的两个参数得到两种不同的页面布局 `UserLayout` 和 `BasicLayout` ；
4. 利用 `Switch`  匹配不同路由控制跳转；
5. 利用 `ConnectedRouter` 传递 history props 给子组件。

那我们再来看下 .umi 文件夹下面的 `router.js` 是怎么样的。

```jsx
// pages/.umi/router.js
import { Router as DefaultRouter, Route, Switch } from 'react-router-dom';
import dynamic from 'umi/dynamic';
import('E:/Tech/Github/umi/examples/simple/global.css');
import Layout from 'E:/Tech/Github/umi/examples/simple/layouts/index.js';


let Router = DefaultRouter;

export default function() {
  return (
<Router history={window.g_history}>
  <Layout><Switch>
    <Route exact path="/" component={require('../index.js').default} />
    <Route exact path="/list" component={() => <div>Compiling...</div>} />
  </Switch></Layout>
</Router>
  );
}
```

可以看到，umi 的 `router.js` 与 pro当中的基本一致，不一样的就是其中的 `Route` 完全由 umi 基于 `pages` 文件夹或者路由配置文件表生成，而不是 `getRouteData` 。

那 `getRouteData` 里面有什么？这个部分还是看一下官方文档的介绍吧~

> 目前在脚手架中，除了[顶层路由](https://github.com/ant-design/ant-design-pro/blob/master/src/router.js)，其余路由列表都是自动生成，其中最关键的就是中心化配置文件 `src/common/router.js`，它的主要作用有两个：
>
> - 配置路由相关信息。如果只考虑生成路由，你只需要指定每条配置的路径及对应渲染组件。
> - 输出路由数据，并将路由数据（routerData）挂载到每条路由对应的组件上。
>
> 这样我们得到一个基本的路由信息对象，它的结构大致是这样：
>
> ```
> {
>   '/dashboard/analysis': {
>     component: DynamicComponent(),
>     name: '分析页',
>   },
>   '/dashboard/monitor': {
>     component: DynamicComponent(),
>     name: '监控页',
>   },
>   '/dashboard/workplace': {
>     component: DynamicComponent(),
>     name: '工作台',
>   },
> }
> ```

大家看到没有？其实 umi 在这里尝试解决的问题是**路由信息的配置**。如果页面简单还好说，如果页面数量达到几十乃至上百，这么多的路由要怎么去一一配置？比如看下 pro 默认提供的路由。

```js
'/': {
  component: dynamicWrapper(app, ['user', 'login'], () => import('../layouts/BasicLayout')),
},
'/dashboard/analysis': {
  component: dynamicWrapper(app, ['chart'], () => import('../routes/Dashboard/Analysis')),
},
'/dashboard/monitor': {
  component: dynamicWrapper(app, ['monitor'], () => import('../routes/Dashboard/Monitor')),
},
'/dashboard/workplace': {
  component: dynamicWrapper(app, ['project', 'activities', 'chart'], () => import('../routes/Dashboard/Workplace')),
},
'/form/basic-form': {
  component: dynamicWrapper(app, ['form'], () => import('../routes/Forms/BasicForm')),
},
'/form/step-form': {
  component: dynamicWrapper(app, ['form'], () => import('../routes/Forms/StepForm')),
},
'/form/step-form/info': {
  component: dynamicWrapper(app, ['form'], () => import('../routes/Forms/StepForm/Step1')),
},
'/form/step-form/confirm': {
  component: dynamicWrapper(app, ['form'], () => import('../routes/Forms/StepForm/Step2')),
},
'/form/step-form/result': {
  component: dynamicWrapper(app, ['form'], () => import('../routes/Forms/StepForm/Step3')),
},
'/form/advanced-form': {
  component: dynamicWrapper(app, ['form'], () => import('../routes/Forms/AdvancedForm')),
},
'/list/table-list': {
  component: dynamicWrapper(app, ['rule'], () => import('../routes/List/TableList')),
},
'/list/basic-list': {
  component: dynamicWrapper(app, ['list'], () => import('../routes/List/BasicList')),
},
'/list/card-list': {
  component: dynamicWrapper(app, ['list'], () => import('../routes/List/CardList')),
},
'/list/search': {
  component: dynamicWrapper(app, ['list'], () => import('../routes/List/List')),
},
'/list/search/projects': {
  component: dynamicWrapper(app, ['list'], () => import('../routes/List/Projects')),
},
'/list/search/applications': {
  component: dynamicWrapper(app, ['list'], () => import('../routes/List/Applications')),
},
'/list/search/articles': {
  component: dynamicWrapper(app, ['list'], () => import('../routes/List/Articles')),
},
'/profile/basic': {
  component: dynamicWrapper(app, ['profile'], () => import('../routes/Profile/BasicProfile')),
},
'/profile/advanced': {
  component: dynamicWrapper(app, ['profile'], () => import('../routes/Profile/AdvancedProfile')),
},
'/result/success': {
  component: dynamicWrapper(app, [], () => import('../routes/Result/Success')),
},
'/result/fail': {
  component: dynamicWrapper(app, [], () => import('../routes/Result/Error')),
},
'/exception/403': {
  component: dynamicWrapper(app, [], () => import('../routes/Exception/403')),
},
'/exception/404': {
  component: dynamicWrapper(app, [], () => import('../routes/Exception/404')),
},
'/exception/500': {
  component: dynamicWrapper(app, [], () => import('../routes/Exception/500')),
},
'/exception/trigger': {
  component: dynamicWrapper(app, ['error'], () => import('../routes/Exception/triggerException')),
},
'/user': {
  component: dynamicWrapper(app, [], () => import('../layouts/UserLayout')),
},
'/user/login': {
  component: dynamicWrapper(app, ['login'], () => import('../routes/User/Login')),
},
'/user/register': {
  component: dynamicWrapper(app, ['register'], () => import('../routes/User/Register')),
},
'/user/register-result': {
  component: dynamicWrapper(app, [], () => import('../routes/User/RegisterResult')),
},
```

如果按这样的方式一个一个去配置和维护路由，我想头发都要掉光了吧。那既然能够在 `src/common/router.js` 中直接采用数据化的方式来配置路由，何不直接将路由与文件的结构直接一一映射呢？umi 在路由上做的其实就是这件事——基于文件结构自动生成配置路由。（是不是回到了前端最初的模样？hhh）

另外，pro 的 `router.js` 在布局上可以自己引入不同的页面布局，而 umi 默认引入 `layout` 文件夹中的 `index.js` 文件作为整体的布局。

```jsx
//pro - router.js
<Switch>
	<Route path="/user" component={UserLayout} // 布局为 UserLayout 
        /> 
	<AuthorizedRoute
        path="/"
        render={props => <BasicLayout {...props} />} // 布局为 BasicLayout
        authority={['admin', 'user']}
        redirectPath="/user/login"
	/>
</Switch>

//umi - router.js
 <Layout>
  <Switch>
        ...
      	...
  </Switch>
</Layout>
```

这会导致如果需要存在不同的页面布局，这里就很难调整。而这也是未来 umi 需要解决的一个棘手的问题。

我想细心的朋友可能会想：哎，如果这里是配置了页面布局的路由，那在 pro 中具体的业务逻辑的页面组件到底是怎么实现路由的？

至于这个问题，就一起来看下一个章节吧~

## 布局

pro 当中 layout 的文件夹包含了不同种的页面布局。默认有有四种不同的布局。一种是通用布局框架 `BasicLayout`，一种是登录页面框架 `UserLoginLayout`，，还有一种是页头布局框架 `PageHeaderLayout`，一种是空白页面框架`BlankLayout`。

其他布局框架我个人觉得没有太多可以说的，主要是 `BasicLayout` 的需要进行简单的介绍。

```jsx
class BasicLayout extends React.PureComponent {
  // ... 一大堆功能函数 
  
  render() {
    const layout = (
      <Layout>
            // 一堆其他组件
          <Content>
            <Switch>
              {
                redirectData.map(item =>
                  (<Redirect
                    key={item.from}
                    exact
                    from={item.from}
                    to={item.to}
                  />))
              }
              {
                getRoutes(match.path, routerData).map(item =>
                  (
                    <AuthorizedRoute
                      key={item.key}
                      path={item.path}
                      component={item.component}
                      exact={item.exact}
                      authority={item.authority}
                      redirectPath="/exception/403"
                    />
                  ))
              }
              <Redirect exact from="/" to={bashRedirect} />
              <Route render={NotFound} />
            </Switch>
          </Content>
          <Footer />
        </Layout>
      </Layout>
    );
    return (
      <DocumentTitle title={this.getPageTitle()}>
        <ContainerQuery query={query}>
          {params => <div className={classNames(params)}>{layout}</div>}
        </ContainerQuery>
      </DocumentTitle>
    );
  }
}

export default connect(({ user, global, loading }) => ({
  currentUser: user.currentUser,
  collapsed: global.collapsed,
  fetchingNotices: loading.effects['global/fetchNotices'],
  notices: global.notices,
}))(BasicLayout);
```

这个页面中除了布局元素以外，还通过`redirectData.map`、 `getRoutes` 函数功能耦合了路由的跳转。那 `getRoutes` 函数是干嘛的？关于 `getRouteData` 引用官方的一段介绍：

> 为了帮助自动生成路由，在 `src/utils/utils.js` 中提供了工具函数 `getRoutes` ，它接收两个参数：当前路由的 [match](https://reacttraining.com/react-router/web/api/match) 路径及路由信息 routerData，主要完成两个工作：
>
> - 筛选路由信息，筛选的算法为**只保留当前 match.path 下最邻近的路由层级（更深入的层级留到嵌套路由中自行渲染）**，举个例子 (每条为一个 route path)：
> ```
> // 当前 match.path 为 /
> /a                 // 没有更近的层级，保留
> /a/b               // 存在更近层级 /a，去掉
> /c/d               // 没有更近的层级，保留
> /c/e               // 没有更近的层级，保留
> /c/e/f             // 存在更近层级 /c/e，去掉
> ```
>
> - 自动分析路由 [exact](https://reacttraining.com/react-router/web/api/Route/exact-bool) 参数，除了下面还有嵌套路由的路径，其余路径默认设为 exact。
>
> 经过 `getRoutes` 处理之后的路由数据就可直接用于生成路由列表：
>
> ```jsx
> // src/layouts/BasicLayout.js
> getRoutes(match.path, routerData).map(item => (
>   <Route
>     key={item.key}
>     path={item.path}
>     component={item.component}
>     exact={item.exact}
>   />
> ))
> ```

简单的来说就是根据上文提过的 `getRouteData` 在 `BasicLayout` 模板页中自动生成对应的路由，挂载上相应的页面组件。

而这种方式赋予了 pro 嵌套路由的能力，提升了页面类型的灵活性，但是又一方面耦合了页面布局与路由，个人感觉可能在写法上会稍微有点点绕。

而 umi 的页面布局则非常简单粗暴

```Jsx
export default function(props) {
  return (
    <div>
      ...
      {
        props.children
      }
      ...
     </div>
  );
}
```

可以看到，layout 是一个无状态函数，只负责布局状态的呈现，并不再负责路由的跳转。这样就把布局与路由的关系完全解耦了。

（实际上我在这边躺坑躺了好久，因为一开始没有给 layout 传进去路由的 history 参数，直接用上 pro 的 layout 一直报错。后来才知道只有在 Switch 里面的组件才能够获得 history  参数。最后用 withRouter 把 Layout 包起来才解决…）

不过随着而来的一个问题就是 layout 的设置问题，因为与路由解耦之后似乎就很难用一种灵活的方式来配置 Layout ，比如识别 `/user` 跳转到完全不同的登陆页面中。（至少作为一名小萌新并想不到什么好方法…）

## 页面

讲完了布局那么接下来是页面了。在 pro 当中，其实每个 routes 的文件夹就是一个对应的业务界面。因为页面更多的就是一些业务的界面逻辑，其实并没有什么好说的。在这边主要对比一下pro 和 umi 页面的创建和挂载的步骤。

### pro

，在 pro 中如果需要新建一个页面，除了在在这个 routes 下面写好对应的业务代码之外，你还需要把这个页面的路由添加到`common/router.js` 中，添加的参数有两个：一个路由地址，另一个是加载的页面组件。格式我在前面中说过，大概如下图所示：

```Js
'/form/basic-form': { // 路由地址
    component: dynamicWrapper(app, ['form'], () => import('../routes/Forms/BasicForm')), // 加载页面
},
```

如果希望在菜单栏中呈现，还需要在对应的 `menu.js` 添加对应的菜单信息，这里不再细讲。

### umi

 umi 中如官网所述，只需要在 `pages` 文件夹下建立对应的页面文件即可。非常简单方便。



那么接下来开始介绍模型的部分。

## 模型

Pro 数据模型采用了 dva ，数据模型放在 `models` 这个文件夹下。默认情况下会直接引入各个模型文件，因为其中有一个 `index.js` 的文件，能够将其他所有的模型文件全部引入。

因此，只需要在 `models` 里面添加对应的模型文件即可。非常简便，轻松，易于用户理解。

那 umi 如果需要用 dva 呢？很简单。首先安装 `umi-dva-plugin` 插件。

```sh
yarn add umi-dva-plugin	
```

然后在项目的根目录添加`.umirc.js` 添加以下内容：

```Js
export default{
  plugin:['umi-dva-plugin'];
}
```

接下来直接在 `models` 文件夹下写 model 就好啦。

如果你关心这个插件到底做了什么。那么可以往下看一看，不然就跳过吧~

 实际上，采用 `umi-dva-plugin` 会生成一个 `dvaContainer.js` 的临时文件。`dvaContainer` 会自动引入 models 下面的文件。

```Jsx
// .umi/dvaContainer.js
import { Component } from 'react';
import dva from 'dva';
import createLoading from 'dva-loading';

const app = dva({
  history: window.g_history,
});
window.g_app = app;
app.use(createLoading());

app.model({ ...(require('../../models/activities.js').default) });
app.model({ ...(require('../../models/chart.js').default) });
app.model({ ...(require('../../models/error.js').default) });
app.model({ ...(require('../../models/form.js').default) });
app.model({ ...(require('../../models/global.js').default) });
app.model({ ...(require('../../models/interview.js').default) });
app.model({ ...(require('../../models/list.js').default) });
app.model({ ...(require('../../models/login.js').default) });
app.model({ ...(require('../../models/monitor.js').default) });
app.model({ ...(require('../../models/profile.js').default) });
app.model({ ...(require('../../models/project.js').default) });
app.model({ ...(require('../../models/register.js').default) });
app.model({ ...(require('../../models/rule.js').default) });
app.model({ ...(require('../../models/user.js').default) });

class DvaContainer extends Component {
  render() {
    app.router(() => this.props.children);
    return app.start()();
  }
}

export default DvaContainer;
```

同时在 `umi.js` 中，umi 会引入 `dvaContainer.js` 从而实现 dva 的使用。

```js
//.umi/umi.js
import React from 'react';
import ReactDOM from 'react-dom';
import createHistory from 'umi/_createHistory';
import FastClick from 'umi-fastclick';


document.addEventListener(
  'DOMContentLoaded',
  () => {
    FastClick.attach(document.body);
  },
  false,
);

// create history
window.g_history = createHistory({
  basename: window.routerBase,
});


// render
function render() {
  const DvaContainer = require('./DvaContainer').default;
ReactDOM.render(React.createElement(
  DvaContainer,
  null,
  React.createElement(require('./router').default)
), document.getElementById('root'));
}
render();

// hot module replacement
if (module.hot) {
  module.hot.accept('./router', () => {
    render();
  });
}

if (process.env.NODE_ENV === 'development') {
  window.g_history.listen(function(location) {
    new Image().src = (window.routerBase + location.pathname).replace(/\/\//g, '/');
  });
}

// Enable service worker
if (process.env.NODE_ENV === 'production') {
  require('./registerServiceWorker');
}
      
```

可以看到，到目前为止 umi 的启动文件的设计思路和 pro 中 `src/index.js` 就一模一样了。

## mock

基本开发的环节基本说完了， 接下来说说 pro 和 umi 的数据 mock。

在 pro 中采用 `.roadhogrc.mock.js`文件来统一 mock 请求。

```Js
// pro - .roadhogrc.mock.js
import mockjs from 'mockjs';
import { getRule, postRule } from './mock/rule';
import { getActivities, getNotice, getFakeList } from './mock/api';
import { getFakeChartData } from './mock/chart';
import { imgMap } from './mock/utils';
import { getProfileBasicData } from './mock/profile';
import { getProfileAdvancedData } from './mock/profile';
import { getNotices } from './mock/notices';
import { format, delay } from 'roadhog-api-doc';

// 是否禁用代理
const noProxy = process.env.NO_PROXY === 'true';

// 代码中会兼容本地 service mock 以及部署站点的静态数据
const proxy = {
  // 支持值为 Object 和 Array
  'GET /api/currentUser': {
    $desc: "获取当前用户接口",
    $params: {
      pageSize: {
        desc: '分页',
        exp: 2,
      },
    },
    $body: {
      name: 'Serati Ma',
      avatar: 'https://gw.alipayobjects.com/zos/rmsportal/BiazfanxmamNRoxxVxka.png',
      userid: '00000001',
      notifyCount: 12,
    },
  },
  // GET POST 可省略
  'GET /api/users': [{
    key: '1',
    name: 'John Brown',
    age: 32,
    address: 'New York No. 1 Lake Park',
  }, {
    key: '2',
    name: 'Jim Green',
    age: 42,
    address: 'London No. 1 Lake Park',
  }, {
    key: '3',
    name: 'Joe Black',
    age: 32,
    address: 'Sidney No. 1 Lake Park',
  }],
  'GET /api/project/notice': getNotice,
  'GET /api/activities': getActivities,
  'GET /api/rule': getRule,
  'POST /api/rule': {
    $params: {
      pageSize: {
        desc: '分页',
        exp: 2,
      },
    },
    $body: postRule,
  },
  'POST /api/forms': (req, res) => {
    res.send({ message: 'Ok' });
  },
  'GET /api/tags': mockjs.mock({
    'list|100': [{ name: '@city', 'value|1-100': 150, 'type|0-2': 1 }]
  }),
  'GET /api/fake_chart_data': getFakeChartData,
  'GET /api/profile/basic': getProfileBasicData,
  'GET /api/profile/advanced': getProfileAdvancedData,
  'POST /api/login/account': (req, res) => {
    const { password, userName, type } = req.body;
    if(password === '888888' && userName === 'admin'){
      res.send({
        status: 'ok',
        type,
        currentAuthority: 'admin'
      });
      return ;
    }
    if(password === '123456' && userName === 'user'){
      res.send({
        status: 'ok',
        type,
        currentAuthority: 'user'
      });
      return ;
    }
    res.send({
      status: 'error',
      type,
      currentAuthority: 'guest'
    });
  },
  'POST /api/register': (req, res) => {
    res.send({ status: 'ok', currentAuthority: 'user' });
  },
  'GET /api/notices': getNotices,
  'GET /api/500': (req, res) => {
    res.status(500).send({
      "timestamp": 1513932555104,
      "status": 500,
      "error": "error",
      "message": "error",
      "path": "/base/category/list"
    });
  },
  'GET /api/404': (req, res) => {
    res.status(404).send({
      "timestamp": 1513932643431,
      "status": 404,
      "error": "Not Found",
      "message": "No message available",
      "path": "/base/category/list/2121212"
    });
  },
  'GET /api/403': (req, res) => {
    res.status(403).send({
      "timestamp": 1513932555104,
      "status": 403,
      "error": "Unauthorized",
      "message": "Unauthorized",
      "path": "/base/category/list"
    });
  },
  'GET /api/401': (req, res) => {
    res.status(401).send({
      "timestamp": 1513932555104,
      "status": 401,
      "error": "Unauthorized",
      "message": "Unauthorized",
      "path": "/base/category/list"
    });
  },
};

export default noProxy ? {} : delay(proxy, 1000);

```

这里规定的格式为：

```js
{
'方法 /路由': 函数,
'方法 /路由': 函数,
}
```

方法既是 RESTful 那一套请求方法， 路由不再多说，函数可以是`(req,res)=>{...}`这种请求返回函数，也可以直接返回已经准备好的 mock 数据。

在 umi 中，也是采用和 pro 一样的约定方式。也就是说，只要将`.roadhogrc.mock.js` 直接改成`.umirc.mock.js` 就可以在 umi 里面使用了。唯一需要注意下的，便是 `resquest.js` 的下面这句引用

```Js
import store from '../index'; 
```

需要改成：

```Jsx
const store = window.g_app._store;
```

## 如何评价

先说说 pro ，pro 提炼出来的业务开发模式个人认为非常高效。它可以让开发者专注于业务页面的开发，不用关注或者更少去关注中间过程的实现，从而更加聚焦于业务逻辑而不是繁琐的配置。

当理解 pro 之后，再去研究 umi，你会发现 umi 其实是对 pro 的一次升级，意图让开发体验变得更加极致。

如果要我来一句话介绍的话，umi 就是一套零配置，引导开发者进行最佳实践的前端框架。「约定高于配制」的思想在 umi 身上体现得淋漓尽致。

和 `parcel.js` 比什么优点？我觉得 umi 约定了一套好用、高效的开发模式，这是单纯的 webpack、parcel 这样的工具无法比拟的。你可以不再使用一些很低效的方式把自己搞的一团乱，而是使用一种更加高效、更加直觉化的模式，从而解放开发者，提高生产力。

只要文档完善，我相信 umi 将会是对新手非常友好的一套开发框架。因为这就仿佛回到了网站开发最初的样子：在根目录添加一个 index.html 就能在浏览器里访问，但是却拥有了更加灵活的控制和更强大的实现能力。

当然约定与配置的比重必须有所权衡，不然真的就会是「用起来爽歪歪，改起来火葬场」了。