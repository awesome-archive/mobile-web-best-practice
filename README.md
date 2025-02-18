# mobile-web-best-practice

> 本项目以基于 [vue-cli3](https://cli.vuejs.org/) 和 [typescript](http://www.typescriptlang.org/) 搭建的 Todo 应用为例，阐述了在使用 web 进行移动端开发中的一些最佳实践方案(并不局限于 [Vue](https://cn.vuejs.org/) 框架)。另外其中很多方案同样适用于 PC 端 Web 开发。

笔者会不定期地将实践中的最佳方案更新到本项目中，最近计划是 hybrid 离线包方案和微前端方案。

## 在线体验

该 Todo 应用交互简洁实用，另外无需服务器，而是将数据保存到 webview 的 indexDB 中，可保证数据安全，欢迎在实际工作生活中使用。效果图如下：

<img src="./assets/memo.gif" width=320/>

| 体验平台 | 二维码                                           | 链接                                            |
| -------- | ------------------------------------------------ | ----------------------------------------------- |
| Web      | <img src="./assets/mwbp.png" width=140>          | [点击体验](http://www.mcuking.club)             |
| Android  | <img src="./assets/mwbpcontainer.png" width=140> | [点击体验](https://www.pgyer.com/mwbpcontainer) |

## 目录

- [项目分层架构](#项目分层架构)
  - [Services 层](#services-层)
  - [Entities 层](#entities-层)
  - [Interactors 层](#interactors-层)
- [组件库](#组件库)
- [JSBridge](#jsbridge)
- [路由堆栈管理（模拟原生 APP 导航）](#路由堆栈管理模拟原生-app-导航)
- [请求数据缓存](#请求数据缓存)
- [Webpack 策略](#webpack-策略)
  - [基础库抽离](#基础库抽离)
- [离线化](#离线化)
- [微前端](#微前端)
- [构建时预渲染](#构建时预渲染)
- [手势库](#手势库)
- [样式适配](#样式适配)
- [表单校验](#表单校验)
- [通过 UA 获取设备信息](#通过-ua-获取设备信息)
- [mock 数据](#mock-数据)
- [调试控制台](#调试控制台)
- [抓包工具](#抓包工具)
- [异常监控平台](#异常监控平台)
- [性能监控平台](#性能监控平台)
- [部署](#部署)
  - [Docker 安装](#docker-安装)
  - [Jenkins 安装和配置](#jenkins-安装和配置)
  - [关联 Jenkins 和 Github](#关联-jenkins-和-github)
  - [实现自动触发打包](#实现自动触发打包)
- [常见问题](#常见问题)

## 项目分层架构

[react-clean-architecture](https://github.com/eduardomoroni/react-clean-architecture)

[business-rules-package](https://github.com/fabriciomendonca/business-rules-package)

[ddd-fe-demo](https://github.com/Vincedream/ddd-fe-demo)

目前前端开发主要是以单页应用为主，当应用的业务逻辑足够复杂的时候，总会遇到类似下面的问题：

- 业务逻辑过于集中在视图层，导致多平台无法共用本应该与平台无关的业务逻辑，例如一个产品需要维护 mobile 和 PC 两端，或者同一个产品有 web 和 react native 两端；

- 产品需要多人协作时，每个人的代码风格和对业务的理解不同，导致业务逻辑分布杂乱无章；

- 对产品的理解停留在页面驱动层面，导致实现的技术模型与实际业务模型出入较大，当业务需求变动时，技术模型很容易被摧毁；

- 过于依赖前端框架，导致如果重构进行框架切换时，需要重写所有业务逻辑并进行回归测试。

针对上面所遇到的问题，笔者学习了一些关于 DDD（领域驱动设计）、Clean Architecture 等知识，并收集了类似思想在前端方面的实践资料，形成了下面这种前端分层架构：

<img src="./assets/architecture.png" width=600/>

其中 View 层想必大家都很了解，就不在这里介绍了，重点介绍下下面三个层的含义：

### Services 层

Services 层是用来对底层技术进行操作的，例如封装 AJAX 请求,操作浏览器 cookie、locaStorage、indexDB，操作 native 提供的能力（如调用摄像头等），以及建立 Websocket 与后端进行交互等。

其中又可细分出来一个 translator 层，主要是对后端提供的接口进行数据的转换修正，例如接口返回的数据命名不规范或格式有问题等等，一般以纯函数形式存在。下面以本项目实际代码为例进行讲解。

向后端获取 quote 数据:

```ts
export class CommonService implements ICommonService {
  @m({ maxAge: 60 * 1000 })
  public async getQuoteList(): Promise<IQuote[]> {
    const {
      data: { list }
    } = await http({
      method: 'post',
      url: '/quote/getList',
      data: {}
    });

    return list;
  }
}
```

向客户端日历中同步任务数据:

```ts
export class NativeService implements INativeService {
  // 同步到日历
  @p()
  public syncCalendar(params: SyncCalendarParams, onSuccess: () => void): void {
    const cb = async (errCode: number) => {
      const msg = NATIVE_ERROR_CODE_MAP[errCode];

      Vue.prototype.$toast(msg);

      if (errCode !== 6000) {
        this.errorReport(msg, 'syncCalendar', params);
      } else {
        await onSuccess();
      }
    };

    dsbridge.call('syncCalendar', params, cb);
  }
  ...
}
```

向 indexDB 存储任务数据：

```ts
export class NoteService implements INoteService {
  public async create(payload: INote, notebookId: number): Promise<void> {
    const db = await createDB();

    const notebook = await db.getFromIndex('notebooks', 'id', notebookId);
    if (notebook) {
      notebook.notes.push(payload);
      await db.put('notebooks', notebook);
    }
  }
  ...
}
```

这里我们可以拓宽下思路，当后端 API 仍在开发的时候，我们可以使用 indexDB 等本地存储技术进行模拟，建立一个 note-indexDB 服务，先提供给上层 Interactors 层进行调用，当后端 API 开发好后，就可以创建一个 note-server 服务，来替换之前的服务。只要保证前后两个服务对外暴露的接口一致，另外与上层的 Interactors 层没有过度耦合，即可实现快速切换。

### Entities 层

实体 Entity 是领域驱动设计的核心概念，它是领域服务的载体，它定义了业务中某个个体的属性和方法。例如本项目中 Note 和 Notebook 都是实体。区分一个对象是否是实体，主要是看他是否有唯一的标志符（例如 id）。下面是本项目的实体 Note:

```ts
export default class Note {
  public id: number;
  public name: string;
  public deadline: Date | undefined;
  ...

  constructor(note: INote) {
    this.id = note.id;
    this.name = note.name;
    this.deadline = note.deadline;
    ...
  }

  public get isExpire() {
    if (this.deadline) {
      return this.deadline.getTime() < new Date().getTime();
    }
  }

  public get deadlineStr() {
    if (this.deadline) {
      return formatTime(this.deadline);
    }
  }
}
```

通过上面的代码可以看到，这里主要是以实体本身的属性以及派生属性为主，当然实体本身也可以具有方法，只是本项目中还没有涉及。至于 DDD 中的聚合等概念，也由于项目业务没有涉及，在这里就不作说明了，有兴趣的可以参考下面列出来的笔者翻译的文章：[可扩展的前端#2--常见模式（译）](https://juejin.im/post/5d8ac00cf265da5b6a16844a)。

另外笔者认为并不是所有的实体都应该按上面那样封装成一个类，如果某个实体本身业务逻辑很简单，就没有必要进行封装，例如本项目中 Notebook 实体就没有做任何封装，而是直接在 Interactors 层调用 Services 层提供的 API。

### Interactors 层

Interactors 层是负责处理业务逻辑的层，主要是由业务用例组成。下面是本项目中 Note 的 Interactors 层提供的对 Note 的增删改查以及同步到日历等业务：

```ts
class NoteInteractor {
  constructor(
    private noteService: INoteService,
    private nativeService: INativeService
  ) {}

  public async saveNote(payload: INote, notebookId: number, isEdit: boolean) {
    try {
      if (isEdit) {
        await this.noteService.edit(payload, notebookId);
      } else {
        await this.noteService.create(payload, notebookId);
      }
    } catch (error) {
      throw error;
    }
  }

  public async getNote(notebookId: number, id: number) {
    try {
      const note = await this.noteService.get(notebookId, id);
      if (note) {
        return new Note(note);
      }
    } catch (error) {
      throw error;
    }
  }

  ...

  public async changeSyncStatus(
    notebookId: number,
    id: number,
    status: boolean
  ) {
    try {
      const note = await this.getNote(notebookId, id);
      if (note) {
        note.isSync = status;
        await this.saveNote(note, notebookId, true);
      }
    } catch (error) {
      throw error;
    }
  }

  public async syncCalendar(params: SyncCalendarParams, notebookId: number) {
    const noteId = params.id;
    try {
      await this.nativeService.syncCalendar(params, async () => {
        await this.changeSyncStatus(notebookId, noteId, true);
      });
    } catch (error) {
      throw error;
    }
  }
}
```

通过上面的代码可以看到，Sevices 层提供的类的实例主要是通过 Interactors 层的类的构造函数获取到，这样就可以达到两层之间解耦，实现快速切换 service 的目的了，当然这个和依赖注入 DI 还是有些差距的，不过已经满足了我们的需求。另外，Interactors 层还可以获取 Entities 层提供的类，构造成实例提供给 View 层。

当然这种分层架构并不是银弹，其主要适用的场景是：实体关系复杂，而交互相对模式化，例如企业软件领域。相反实体关系简单而交互复杂多变就不适合这种分层架构了。

在具体业务开发实践中，这种领域模型以及实体一般都是有后端同学确定的，我们需要做的是，和后端的领域模型保持一致，但不是一样。例如同一个功能，在前端只是一个简单的按钮，而在后端则可能相当复杂。

然后需要明确的是，架构和项目文件结构并不是等同的，文件结构是你从视觉上分离应用程序各部分的方式，而架构是从概念上分离应用程序的方式。你可以在很好地保持相同架构的同时，选择不同的文件结构方式。没有完美的文件结构，因此请根据项目的不同选择适合你的文件结构。

最后引用蚂蚁金服数据体验技术的《前端开发-领域驱动设计》文章中的总结作为结尾：

> 要明白，驱动领域层分离的目的并不是页面被复用，这一点在思想上一定要转化过来。领域层并不是因为被多个地方复用而被抽离。它被抽离的原因是：
>
> - 领域层是稳定的（页面以及与页面绑定的模块都是不稳定的）
> - 领域层是解耦的（页面是会耦合的，页面的数据会来自多个接口，多个领域）
> - 领域层具有极高复杂度，值得单独管理(view 层处理页面渲染以及页面逻辑控制，复杂度已经够高，领域层解耦可以轻 view 层。view 层尽可能轻量是我们架构师 cnfi 主推的思路)
> - 领域层以层为单位是可以被复用的（你的代码可能会抛弃某个技术体系，从 vue 转成 react，或者可能会推出一个移动版，在这些情况下，领域层这一层都是可以直接复用）
> - 为了领域模型的持续衍进(模型存在的目的是让人们聚焦，聚焦的好处是加强了前端团队对于业务的理解，思考业务的过程才能让业务前进)

下面推荐几篇相关文章：

[前端架构-让重构不那么痛苦（译）](https://juejin.im/post/5d849084e51d456206115acb)

[可扩展的前端#1--架构基础（译）](https://juejin.im/post/5d897d13f265da03c5035030)

[可扩展的前端#2--常见模式（译）](https://juejin.im/post/5d8ac00cf265da5b6a16844a)

[领域驱动设计在互联网业务开发中的实践](https://tech.meituan.com/2017/12/22/ddd-in-practice.html)

[前端开发-领域驱动设计](https://juejin.im/post/5b1c71ad6fb9a01e5918398d)

[领域驱动设计在前端中的应用](https://juejin.im/post/5d3926176fb9a07ef161c719)

## 组件库

[vant](https://youzan.github.io/vant/#/zh-CN/intro)

[vux](https://github.com/airyland/vux)

[mint-ui](https://github.com/ElemeFE/mint-ui)

[cube-ui](https://github.com/didi/cube-ui)

Vue 移动端组件库目前主要就是上面罗列的这几个库，本项目使用的是有赞前端团队开源的 vant。

vant 官方目前已经支持自定义样式主题，基本原理就是在 [less-loader](https://github.com/webpack-contrib/less-loader) 编译 [less](http://lesscss.org/) 文件到 css 文件过程中，利用 less 提供的 [modifyVars](http://lesscss.org/usage/#using-less-in-the-browser-modify-variables) 对 less 变量进行修改，本项目也采用了该方式，具体配置请查看相关文档：

[定制主题](https://youzan.github.io/vant/#/zh-CN/theme)

推荐一篇介绍各个组件库特点的文章：

[Vue 常用组件库的比较分析（移动端）](https://blog.csdn.net/weixin_38633659/article/details/89736656)

## JSBridge

[DSBridge-IOS](https://github.com/wendux/DSBridge-IOS)

[DSBridge-Android](https://github.com/wendux/DSBridge-Android)

[WebViewJavascriptBridge](https://github.com/marcuswestin/WebViewJavascriptBridge)

混合应用中一般都是通过 webview 加载网页，而当网页要获取设备能力（例如调用摄像头、本地日历等）或者 native 需要调用网页里的方法，就需要通过 JSBridge 进行通信。

开源社区中有很多功能强大的 JSBridge，例如上面列举的库。本项目基于保持 iOS android 平台接口统一原因，采用了 DSBridge，各位可以选择适合自己项目的工具。

本项目以 h5 调用 native 提供的同步日历接口为例，演示如何在 dsbridge 基础上进行两端通信的。下面是两端的关键代码摘要：

安卓端同步日历核心代码，具体代码请查看与本项目配套的安卓项目 [mobile-web-best-practice-container](https://github.com/mcuking/mobile-web-best-practice-container)：

```java
public class JsApi {
    /**
     * 同步日历接口
     * msg 格式如下：
     * ...
     */
    @JavascriptInterface
    public void syncCalendar(Object msg, CompletionHandler<Integer> handler) {
        try {
            JSONObject obj = new JSONObject(msg.toString());
            String id = obj.getString("id");
            String title = obj.getString("title");
            String location = obj.getString("location");
            long startTime = obj.getLong("startTime");
            long endTime = obj.getLong("endTime");
            JSONArray earlyRemindTime = obj.getJSONArray("alarm");
            String res = CalendarReminderUtils.addCalendarEvent(id, title, location, startTime, endTime, earlyRemindTime);
            handler.complete(Integer.valueOf(res));
        } catch (Exception e) {
            e.printStackTrace();
            handler.complete(6005);
        }
    }
}
```

h5 端同步日历核心代码（通过装饰器来限制调用接口的平台）

```ts
class NativeMethods {
  // 同步到日历
  @p()
  public syncCalendar(params: SyncCalendarParams) {
    const cb = (errCode: number) => {
      const msg = NATIVE_ERROR_CODE_MAP[errCode];

      Vue.prototype.$toast(msg);

      if (errCode !== 6000) {
        this.errorReport(msg, 'syncCalendar', params);
      }
    };
    dsbridge.call('syncCalendar', params, cb);
  }

  // 调用 native 接口出错向 sentry 发送错误信息
  private errorReport(errorMsg: string, methodName: string, params: any) {
    if (window.$sentry) {
      const errorInfo: NativeApiErrorInfo = {
        error: new Error(errorMsg),
        type: 'callNative',
        methodName,
        params: JSON.stringify(params)
      };
      window.$sentry.log(errorInfo);
    }
  }
}

/**
 * @param {platforms} - 接口限制的平台
 * @return {Function} - 装饰器
 */
function p(platforms = ['android', 'ios']) {
  return (target: AnyObject, name: string, descriptor: PropertyDescriptor) => {
    if (!platforms.includes(window.$platform)) {
      descriptor.value = () => {
        return Vue.prototype.$toast(
          `当前处在 ${window.$platform} 环境，无法调用接口哦`
        );
      };
    }

    return descriptor;
  };
}
```

另外推荐一个笔者之前写的一个基于安卓平台实现的教学版 [JSBridge](https://github.com/mcuking/JSBridge)，里面详细阐述了如何基于底层接口一步步封装一个可用的 JSBridge：

[JSBridge 实现原理](https://github.com/mcuking/JSBridge)

## 路由堆栈管理（模拟原生 APP 导航）

[vue-page-stack](https://github.com/hezhongfeng/vue-page-stack)

[vue-navigation](https://github.com/zack24q/vue-navigation)

[vue-stack-router](https://github.com/luojilab/vue-stack-router)

在使用 h5 开发 app，会经常遇到下面的需求：
从列表进入详情页，返回后能够记住当前位置，或者从表单点击某项进入到其他页面选择，然后回到表单页，需要记住之前表单填写的数据。可是目前 vue 或 react 框架的路由，均不支持同时存在两个页面实例，所以需要路由堆栈进行管理。

其中 vue-page-stack 和 vue-navigation 均受 vue 的 keepalive 启发，基于 [vue-router](https://router.vuejs.org/)，当进入某个页面时，会查看当前页面是否有缓存，有缓存的话就取出缓存，并且清除排在他后面的所有 vnode，没有缓存就是新的页面，需要存储或者是 replace 当前页面，向栈里面 push 对应的 vnode，从而实现记住页面状态的功能。

而逻辑思维前端团队的 vue-stack-router 则另辟蹊径，抛开了 vue-router，自己独立实现了路由管理，相较于 vue-router，主要是支持同时可以存活 A 和 B 两个页面的实例，或者 A 页面不同状态的两个实例，并支持原生左滑功能。但由于项目还在初期完善，功能还没有 vue-router 强大，建议持续关注后续动态再做决定是否引入。

本项目使用的是 vue-page-stack，各位可以选择适合自己项目的工具。同时推荐几篇相关文章：

[【vue-page-stack】Vue 单页应用导航管理器 正式发布](https://juejin.im/post/5d2ef417f265da1b971aa94f)

[Vue 社区的路由解决方案：vue-stack-router](https://juejin.im/post/5d4ce4fd6fb9a06acd450e8c)

## 请求数据缓存

[mem](https://github.com/sindresorhus/mem)

在我们的应用中，会存在一些很少改动的数据，而这些数据有需要从后端获取，比如公司人员、公司职位分类等，此类数据在很长一段时间时不会改变的，而每次打开页面或切换页面时，就重新向后端请求。为了能够减少不必要请求，加快页面渲染速度，可以引用 mem 缓存库。

mem 基本原理是通过以接收的函数为 key 创建一个 WeakMap，然后再以函数参数为 key 创建一个 Map，value 就是函数的执行结果，同时将这个 Map 作为刚刚的 WeakMap 的 value 形成嵌套关系，从而实现对同一个函数不同参数进行缓存。而且支持传入 maxAge，即数据的有效期，当某个数据到达有效期后，会自动销毁，避免内存泄漏。

选择 WeakMap 是因为其相对 Map 保持对键名所引用的对象是弱引用，即垃圾回收机制不将该引用考虑在内。只要所引用的对象的其他引用都被清除，垃圾回收机制就会释放该对象所占用的内存。也就是说，一旦不再需要，WeakMap 里面的键名对象和所对应的键值对会自动消失，不用手动删除引用。

mem 作为高阶函数，可以直接接受封装好的接口请求。但是为了更加直观简便，我们可以按照类的形式集成我们的接口函数，然后就可以用装饰器的方式使用 mem 了（装饰器只能修饰类和类的类的方法，因为普通函数会存在变量提升）。下面是相关代码：

```ts
import http from '../http';
import mem from 'mem';

/**
 * @param {MemOption} - mem 配置项
 * @return {Function} - 装饰器
 */
export default function m(options: AnyObject) {
  return (target: AnyObject, name: string, descriptor: PropertyDescriptor) => {
    const oldValue = descriptor.value;
    descriptor.value = mem(oldValue, options);
    return descriptor;
  };
}

export class CommonService {
  @m({ maxAge: 60 * 1000 })
  public async getQuoteList(): Promise<IQuote[]> {
    const {
      data: { list }
    } = await http({
      method: 'post',
      url: '/quote/getList',
      data: {}
    });

    return list;
  }
}
```

## Webpack 策略

### 基础库抽离

对于一些基础库，例如 vue、moment 等，属于不经常变化的静态依赖，一般需要抽离出来以提升每次构建的效率。目前主流方案有两种：

一种是使用 [webpack-dll-plugin](https://webpack.docschina.org/plugins/dll-plugin/) 插件，在首次构建时就讲这些静态依赖单独打包，后续只需引入早已打包好的静态依赖包即可；

另一种就是外部扩展 [Externals](https://webpack.docschina.org/configuration/externals/) 方式，即把不需要打包的静态资源从构建中剔除，使用 CDN 方式引入。下面是 webpack-dll-plugin 相对 Externals 的缺点：

1. 需要配置在每次构建时都不参与编译的静态依赖，并在首次构建时为它们预编译出一份 JS 文件（后文将称其为 lib 文件），每次更新依赖需要手动进行维护，一旦增删依赖或者变更资源版本忘记更新，就会出现 Error 或者版本错误。

2. 无法接入浏览器的新特性 script type="module"，对于某些依赖库提供的原生 ES Modules 的引入方式（比如 vue 的新版引入方式）无法得到支持，没法更好地适配高版本浏览器提供的优良特性以实现更好地性能优化。

3. 将所有资源预编译成一份文件，并将这份文件显式注入项目构建的 HTML 模板中，这样的做法，在 HTTP1 时代是被推崇的，因为那样能减少资源的请求数量，但在 HTTP2 时代如果拆成多个 CDN Link，就能够更充分地利用 HTTP2 的多路复用特性。

不过选择 Externals 还是需要一个靠谱的 CDN 服务的。

本项目选择的是 Externals，各位可根据项目需求选择不同的方案。

更多内容请查看这篇文章（上面观点来自于这篇文章）：

[Webpack 优化——将你的构建效率提速翻倍](https://juejin.im/post/5d614dc96fb9a06ae3726b3e)

## 离线化

[mobile-web-best-practice-container](https://github.com/mcuking/mobile-web-best-practice-container)

[offline-package-admin](https://github.com/mcuking/offline-package-admin)

[offline-package-webpack-plugin](https://github.com/mcuking/offline-package-webpack-plugin)

离线化技术可以将网页的网络加载时间变为 0，极大提升应用的用户体验。目前有以下几种实现方式：

- Service Workers: 目前因无法在 iOS 大范围使用；

- 实现一个类 Service Workers 的被动离线化技术，解决 iOS 不兼容问题；

- 离线包技术，即将前端静态资源提前集成到客户端中。

笔者选择了离线包技术，并计划在今年年底之前完成，因这个方案会涉及到前端、客户端以及后端，尤其是客户端工作量较大，所以会花费较长周期，届时会开源整个方案的所有代码，敬请期待。

## 微前端

[qiankun](https://github.com/umijs/qiankun)

todo

## 构建时预渲染

针对目前单页面首屏渲染时间长（需要下载解析 js 文件然后渲染元素并挂载到 id 为 app 的 div 上），SEO 不友好（index.html 的 body 上实际元素只有 id 为 app 的 div 元素，真正的页面元素都是动态挂载的，搜索引擎的爬虫无法捕捉到），目前主流解决方案就是服务端渲染（SSR），即从服务端生成组装好的完整静态 html 发送到浏览器进行展示，但配置较为复杂，一般都会借助框架，比如 vue 的 [nuxt.js](https://github.com/nuxt/nuxt.js)，react 的 [next](https://github.com/zeit/next.js)。

其实有一种更简便的方式--构建时预渲染。顾名思义，就是项目打包构建完成后，启动一个 Web Server 来运行整个网站，再开启多个无头浏览器（例如 [Puppeteer](https://github.com/GoogleChrome/puppeteer)、[Phantomjs](https://github.com/ariya/phantomjs) 等无头浏览器技术）去请求项目中所有的路由，当请求的网页渲染到第一个需要预渲染的页面时（需提前配置需要预渲染页面的路由），会主动抛出一个事件，该事件由无头浏览器截获，然后将此时的页面内容生成一个 HTML（包含了 JS 生成的 DOM 结构和 CSS 样式），保存到打包文件夹中。

根据上面的描述，我们可以其实它本质上就只是快照页面，不适合过度依赖后端接口的动态页面，比较适合变化不频繁的静态页面。

实际项目相关工具方面比较推荐 [prerender-spa-plugin](https://github.com/chrisvfritz/prerender-spa-plugin) 这个 webpack 插件，下面是这个插件的原理图。不过有两点需要注意：

一个是这个插件需要依赖 Puppeteer，而因为国内网络原因以及本身体积较大，经常下载失败，不过可以通过 .npmrc 文件指定 Puppeteer 的下载路径为国内镜像；

另一个是需要设置路由模式为 history 模式（即基于 html5 提供的 history api 实现的，react 叫 BrowserRouter，vue 叫 history），因为 hash 路由无法对应到实际的物理路由。（即线上渲染时 history 下，如果 form 路由被设置成预渲染，那么访问 /form/ 路由时，会直接从服务端返回 form 文件夹下的 index.html，之前打包时就已经预先生成了完整的 HTML 文件 ）

<img src="./assets/prerender-spa-plugin.png" width="1200"/>

本项目已经集成了 prerender-spa-plugin，但由于和 vue-stack-page/vue-navigation 这类路由堆栈管理器一起使用有问题（原因还在查找，如果知道的朋友也可以告知下），所以 prerender 功能是关闭的。

同时推荐几篇相关文章：

[vue 预渲染之 prerender-spa-plugin 解析(一)](https://blog.csdn.net/vv_bug/article/details/84593052)

[使用预渲提升 SPA 应用体验](https://juejin.im/post/5d5fa22ee51d4561de20b5f5)

## 手势库

[hammer.js](https://github.com/hammerjs/hammer.js)

[AlloyFinger](https://github.com/AlloyTeam/AlloyFinger)

在移动端开发中，一般都需要支持一些手势，例如拖动（Pan）,缩放（Pinch）,旋转（Rotate）,滑动（swipe）等。目前已经有很成熟的方案了，例如 hammer.js 和腾讯前端团队开发的 AlloyFinger 都很不错。本项目选择基于 hammer.js 进行二次封装成 vue 指令集，各位可根据项目需求选择不同的方案。

下面是二次封装的关键代码，其中用到了 webpack 的 require.context 函数来获取特定模块的上下文，主要用来实现自动化导入模块，比较适用于像 vue 指令这种模块较多的场景：

```ts
// 用于导入模块的上下文
export const importAll = (
  context: __WebpackModuleApi.RequireContext,
  options: ImportAllOptions = {}
): AnyObject => {
  const { useDefault = true, keyTransformFunc, filterFunc } = options;

  let keys = context.keys();

  if (isFunction(filterFunc)) {
    keys = keys.filter(filterFunc);
  }

  return keys.reduce((acc: AnyObject, curr: string) => {
    const key = isFunction(keyTransformFunc) ? keyTransformFunc(curr) : curr;
    acc[key] = useDefault ? context(curr).default : context(curr);
    return acc;
  }, {});
};

// directives 文件夹下的 index.ts
const directvieContext = require.context('./', false, /\.ts$/);
const directives = importAll(directvieContext, {
  filterFunc: (key: string) => key !== './index.ts',
  keyTransformFunc: (key: string) =>
    key.replace(/^\.\//, '').replace(/\.ts$/, '')
});

export default {
  install(vue: typeof Vue): void {
    Object.keys(directives).forEach((key) =>
      vue.directive(key, directives[key])
    );
  }
};

// touch.ts
export default {
  bind(el: HTMLElement, binding: DirectiveBinding) {
    const hammer: HammerManager = new Hammer(el);
    const touch = binding.arg as Touch;
    const listener = binding.value as HammerListener;
    const modifiers = Object.keys(binding.modifiers);

    switch (touch) {
      case Touch.Pan:
        const panEvent = detectPanEvent(modifiers);
        hammer.on(`pan${panEvent}`, listener);
        break;
      ...
    }
  }
};
```

另外推荐一篇关于 hammer.js 和一篇关于 require.context 的文章：

[H5 案例分享：JS 手势框架 —— Hammer.js](https://www.h5anli.com/articles/201609/hammerjs.html)

[使用 require.context 实现前端工程自动化](https://www.jianshu.com/p/c894ea00dfec)

## 样式适配

[postcss-px-to-viewport](https://github.com/evrone/postcss-px-to-viewport)

[Viewport Units Buggyfill](https://github.com/rodneyrehm/viewport-units-buggyfill)

[flexible](https://github.com/amfe/lib-flexible)

[postcss-pxtorem](https://github.com/cuth/postcss-pxtorem)

[Autoprefixer](https://github.com/postcss/autoprefixer)

[browserslist](https://github.com/browserslist/browserslist)

在移动端网页开发时，样式适配始终是一个绕不开的问题。对此目前主流方案有 vw 和 rem（当然还有 vw + rem 结合方案，请见下方 rem-vw-layout 仓库），其实基本原理都是相通的，就是随着屏幕宽度或字体大小成正比变化。因为原理方面的详细资料网络上已经有很多了，就不在这里赘述了。下面主要提供一些这工程方面的工具。

关于 rem，阿里无线前端团队在 15 年的时候基于 rem 推出了 flexible 方案，以及 postcss 提供的自动转换 px 到 rem 的插件 postcss-pxtorem。

关于 vw，可以使用 postcss-px-to-viewport 进行自动转换 px 到 vw。postcss-px-to-viewport 相关配置如下：

```js
"postcss-px-to-viewport": {
  viewportWidth: 375, // 视窗的宽度，对应的是我们设计稿的宽度，一般是375
  viewportHeight: 667, // 视窗的高度，根据750设备的宽度来指定，一般指定1334，也可以不配置
  unitPrecision: 3,  // 指定`px`转换为视窗单位值的小数位数（很多时候无法整除）
  viewportUnit: 'vw', // 指定需要转换成的视窗单位，建议使用vw
  selectorBlackList: ['.ignore', '.hairlines'], // 指定不转换为视窗单位的类，可以自定义，可以无限添加,建议定义一至两个通用的类名
  minPixelValue: 1, // 小于或等于`1px`不转换为视窗单位，你也可以设置为你想要的值
  mediaQuery: false // 媒体查询里的单位是否需要转换单位
}
```

下面是 vw 和 rem 的优缺点对比图：

<img src="./assets/vw-rem.png" width="1200"/>

关于 vw 兼容性问题，目前在移动端 iOS 8 以上以及 Android 4.4 以上获得支持。如果有兼容更低版本需求的话，可以选择 viewport 的 pollify 方案，其中比较主流的是 [Viewport Units Buggyfill](https://github.com/rodneyrehm/viewport-units-buggyfill)。

本方案因不准备兼容低版本，所以直接选择了 vw 方案，各位可根据项目需求选择不同的方案。

另外关于设置 css 兼容不同浏览器，想必大家都知道 Autoprefixer（vue-cli3 已经默认集成了），那么如何设置要兼容的范围呢？推荐使用 browserslist，可以在 .browserslistrc 或者 pacakage.json 中 browserslist 部分设置兼容浏览器范围。因为不止 Autoprefixer，还有 Babel，postcss-preset-env 等工具都会读取 browserslist 的兼容配置，这样比较容易使 js css 兼容浏览器的范围保持一致。下面是本项目的 .browserslistrc 配置：

```js
iOS >= 10  //  即 iOS Safari
Android >= 6.0 // 即 Android WebView
last 2 versions // 每个浏览器最近的两个版本
```

最后推荐一些移动端样式适配的资料：

[rem-vw-layout](https://github.com/imwtr/rem-vw-layout)

[细说移动端 经典的 REM 布局 与 新秀 VW 布局](https://www.cnblogs.com/imwtr/p/9648233.html)

[如何在 Vue 项目中使用 vw 实现移动端适配](https://www.jianshu.com/p/1f1b23f8348f)

## 表单校验

[async-validator](https://github.com/yiminghe/async-validator)

[vee-validate](https://github.com/baianat/vee-validate)

由于大部分移动端组件库都不提供表单校验，因此需要自己封装。目前比较多的方式就是基于 async-validator 进行二次封装（elementUI 组件库提供的表单校验也是基于 async-validator ），或者使用 vee-validate（一种基于 vue 模板的轻量级校验框架）进行校验，各位可根据项目需求选择不同的方案。

本项目的表单校验方案是在 async-validator 基础上进行二次封装，代码如下，原理很简单，基本满足需求。如果还有更完善的方案，欢迎提出来。

其中 setRules 方法是将组件中设置的 rules（符合 async-validator 约定的校验规则）按照需要校验的数据的名字为 key 转化一个对象 validator，value 是 async-validator 生成的实例。validator 方法可以接收单个或多个需要校验的数据的 key，然后就会在 setRules 生成的对象 validator 中寻找 key 对应的 async-validator 实例，最后调用实例的校验方法。当然也可以不接受参数，那么就会校验所有传入的数据。

```ts
import schema from 'async-validator';
...

class ValidatorUtils {
  private data: AnyObject;
  private validators: AnyObject;

  constructor({ rules = {}, data = {}, cover = true }) {
    this.validators = {};
    this.data = data;
    this.setRules(rules, cover);
  }

  /**
   * 设置校验规则
   * @param rules async-validator 的校验规则
   * @param cover 是否替换旧规则
   */
  public setRules(rules: ValidateRules, cover: boolean) {
    if (cover) {
      this.validators = {};
    }

    Object.keys(rules).forEach((key) => {
      this.validators[key] = new schema({ [key]: rules[key] });
    });
  }

  public validate(
    dataKey?: string | string[]
  ): Promise<ValidateError[] | string | string[] | undefined> {
    // 错误数组
    const err: ValidateError[] = [];

    Object.keys(this.validators)
      .filter((key) => {
        // 若不传 dataKey 则校验全部。否则校验 dataKey 对应的数据（dataKey 可以对应一个（字符串）或多个（数组））
        return (
          !dataKey ||
          (dataKey &&
            ((_.isString(dataKey) && dataKey === key) ||
              (_.isArray(dataKey) && dataKey.includes(key))))
        );
      })
      .forEach((key) => {
        this.validators[key].validate(
          { [key]: this.data[key] },
          (error: ValidateError[]) => {
            if (error) {
              err.push(error[0]);
            }
          }
        );
      });

    if (err.length > 0) {
      return Promise.reject(err);
    } else {
      return Promise.resolve(dataKey);
    }
  }
}
```

## 通过 UA 获取设备信息

在开发 h5 开发时，可能会遇到下面几种情况：

1. 开发时都是在浏览器进行开发调试的，所以需要避免调用 native 的接口，因为这些接口在浏览器环境根本不存在；
2. 有些情况需要区分所在环境是在 android webview 还是 ios webview，做一些针对特定平台的处理；
3. 当 h5 版本已经更新，但是客户端版本并没有同步更新，那么如果之间的接口调用发生了改变，就会出现调用出错。

所以需要一种方式来检测页面当前所处设备的平台类型、app 版本、系统版本等，目前比较靠谱的方式是通过 android / ios webview 修改 UserAgent，在原有的基础上加上特定后缀，然后在网页就可以通过 UA 获取设备相关信息了。当然这种方式的前提是 native 代码是可以为此做出改动的。以安卓为例关键代码如下：

安卓关键代码：

```java
// Activity -> onCreate
...
// 获取 app 版本
PackageManager packageManager = getPackageManager();
PackageInfo packInfo = null;
try {
  // getPackageName()是你当前类的包名，0代表是获取版本信息
  packInfo = packageManager.getPackageInfo(getPackageName(),0);
} catch (PackageManager.NameNotFoundException e) {
  e.printStackTrace();
}
String appVersion = packInfo.versionName;

// 获取系统版本
String systemVersion = android.os.Build.VERSION.RELEASE;

mWebSettings.setUserAgentString(
  mWebSettings.getUserAgentString() + " DSBRIDGE_"  + appVersion + "_" + systemVersion + "_android"
);
```

h5 关键代码：

```ts
const initDeviceInfo = () => {
  const UA = navigator.userAgent;
  const info = UA.match(/\s{1}DSBRIDGE[\w\.]+$/g);
  if (info && info.length > 0) {
    const infoArray = info[0].split('_');
    window.$appVersion = infoArray[1];
    window.$systemVersion = infoArray[2];
    window.$platform = infoArray[3] as Platform;
  } else {
    window.$appVersion = undefined;
    window.$systemVersion = undefined;
    window.$platform = 'browser';
  }
};
```

## mock 数据

[Mock](https://github.com/nuysoft/Mock)

当前后端进度不一致，接口还尚未实现时，为了不影响彼此的进度，此时前后端约定好接口数据格式后，前端就可以使用 mock 数据进行独立开发了。本项目使用了 Mock 实现前端所需的接口。

## 调试控制台

[eruda](https://github.com/liriliri/eruda)

[vconsole](https://github.com/Tencent/vConsole)

在调试方面，本项目使用 eruda 作为手机端调试面板，功能相当于打开 PC 控制台，可以很方便地查看 console, network, cookie, localStorage 等关键调试信息。与之类似地工具还有微信的前端研发团队开发的 vconsole，各位可以选择适合自己项目的工具。

关于 eruda 使用，推荐使用 cdn 方式加载，至于什么时候加载 eruda，可以根据不同项目制定不同策略。示例代码如下：

```ts
<script>
  (function() {
    const NO_ERUDA = window.location.protocol === 'https:';
    if (NO_ERUDA) return;
    const src = 'https://cdn.jsdelivr.net/npm/eruda@1.5.8/eruda.min.js';
    document.write('<scr' + 'ipt src="' + src + '"></scr' + 'ipt>');
    document.write('<scr' + 'ipt>eruda.init();</scr' + 'ipt>');
  })();
</script>
```

## 抓包工具

[charles](https://www.charlesproxy.com/)

[fiddler](https://www.telerik.com/fiddler)

虽然有了 eruda 调试工具，但某些情况下仍不能满足需求，比如现网完全关闭 eruda 等情况。

此时就需要抓包工具，相关工具主要就是上面罗列的这两个，各位可以选择适合自己项目的工具。

通过 charles 可以清晰的查看所有请求的信息(注：https 下抓包需要在手机上配置相关证书)。当然 charles 还有更多强大功能，比例模拟弱网情况，资源映射等。

推荐一篇不错的 charles 使用教程：

[解锁 Charles 的姿势](https://juejin.im/post/5a1033d2f265da431f4aa81f)

## 异常监控平台

[sentry](https://github.com/getsentry/sentry)

[fundebug](https://www.fundebug.com/)

移动端网页相对 PC 端，主要有设备众多，网络条件各异，调试困难等特点。导致如下问题：

- 设备兼容或网络异常导致只有部分情况下才出现的 bug，测试无法全面覆盖

- 无法获取出现 bug 的用户的设备，又不能复现反馈的 bug

- 部分 bug 只出现几次，后面无法复现，不能还原事故现场

这时就非常需要一个异常监控平台，将异常实时上传到平台，并及时通知相关人员。

相关工具有 sentry，fundebug 等，其中 sentry 因为功能强大，支持多平台监控（不仅可以监控前端项目），完全开源，可以私有化部署等特点，而被广泛采纳。

下面是 sentry 在本项目应用时使用的相关配套工具。

**sentry 针对 javascript 的 sdk**

[sentry-javascript](https://github.com/getsentry/sentry-javascript)

**自动上传 sourcemap 的 webpack 插件**

[sentry-webpack-plugin](https://github.com/getsentry/sentry-webpack-plugin)

**编译时自动在 try catch 中添加错误上报函数的 babel 插件**

[babel-plugin-try-catch-error-report](https://github.com/mcuking/babel-plugin-try-catch-error-report)

**补充：**

前端的异常主要有以下几个部分：

- 静态资源加载异常

- 接口异常（包括与后端和 native 的接口）

- js 报错

- 网页崩溃

其中静态资源加载失败，可以通过 window.addEventListener('error', ..., true) 在事件捕获阶段获取，然后筛选出资源加载失败的错误并手动上报错误。核心代码如下：

```ts
// 全局监控资源加载错误
window.addEventListener(
  'error',
  (event) => {
    // 过滤 js error
    const target = event.target || event.srcElement;
    const isElementTarget =
      target instanceof HTMLScriptElement ||
      target instanceof HTMLLinkElement ||
      target instanceof HTMLImageElement;
    if (!isElementTarget) {
      return false;
    }
    // 上报资源地址
    const url =
      (target as HTMLScriptElement | HTMLImageElement).src ||
      (target as HTMLLinkElement).href;

    this.log({
      error: new Error(`ResourceLoadError: ${url}`),
      type: 'resource load'
    });
  },
  true
);
```

关于服务端接口异常，可以通过在封装的 http 模块中，全局集成上报错误函数（native 接口的错误上报类似，可在项目中查看）。核心代码如下：

```ts
function errorReport(
  url: string,
  error: string | Error,
  requestOptions: AxiosRequestConfig,
  response?: AnyObject
) {
  if (window.$sentry) {
    const errorInfo: RequestErrorInfo = {
      error: typeof error === 'string' ? new Error(error) : error,
      type: 'request',
      requestUrl: url,
      requestOptions: JSON.stringify(requestOptions)
    };

    if (response) {
      errorInfo.response = JSON.stringify(response);
    }

    window.$sentry.log(errorInfo);
  }
}
```

关于全局 js 报错，sentry 针对的前端的 sdk 已经通过 window.onerror 和 window.addEventListener('unhandledrejection', ..., false) 进行全局监听并上报。

需要注意的是其中 window.onerror = (message, source, lineno, colno, error) =>{} 不同于 window.addEventListener('error', ...)，window.onerror 捕获的信息更丰富，包括了错误字符串信息、发生错误的 js 文件，错误所在的行数、列数、和 Error 对象（其中还会有调用堆栈信息等）。所以 sentry 会选择 window.onerror 进行 js 全局监控。

但有一种错误是 window.onerror 监听不到的，那就是 unhandledrejection 错误，这个错误是当 promise reject 后没有 catch 住所引起的。当然 sentry 的 sdk 也已经做了监听。

针对 vue 项目，也可对 errorHandler 钩子进行全局监听，react 的话可以通过 componentDidCatch 钩子，vue 相关代码如下：

```ts
// 全局监控 Vue errorHandler
Vue.config.errorHandler = (error, vm, info) => {
  window.$sentry.log({
    error,
    type: 'vue errorHandler',
    vm,
    info
  });
};
```

但是对于我们业务中，经常会对一些以报错代码使用 try catch，这些错误如果没有在 catch 中向上抛出，是无法通过 window.onerror 捕获的，针对这种情况，笔者开发了一个 babel 插件 [babel-plugin-try-catch-error-report](https://github.com/mcuking/babel-plugin-try-catch-error-report)，该插件可以在 [babel](https://babeljs.io/) 编译 js 的过程中，通过在 ast 中查找 catch 节点，然后再 catch 代码块中自动插入错误上报函数，可以自定义函数名，和上报的内容（源码所在文件，行数，列数，调用栈，以及当前 window 属性，比如当前路由信息 window.location.href）。相关配置代码如下：

```js
if (!IS_DEV) {
  plugins.push([
    'try-catch-error-report',
    {
      expression: 'window.$sentry.log',
      needFilename: true,
      needLineNo: true,
      needColumnNo: false,
      needContext: true,
      exclude: ['node_modules']
    }
  ]);
}
```

针对跨域 js 问题，当加载的不同域的 js 文件时，例如通过 cdn 加载打包后的 js。如果 js 报错，window.onerror 只能捕获到 script error，没有任何有效信息能帮助我们定位问题。此时就需要我们做一些事情：
第一步、服务端需要在返回 js 的返回头设置 Access-Control-Allow-Origin: \*
第二部、设置 script 标签属性 crossorigin，代码如下：

```html
<script src="http://helloworld/main.js" crossorigin></script>
```

如果是动态添加的，也可动态设置：

```js
const script = document.createElement('script');
script.crossOrigin = 'anonymous';
script.src = url;
document.body.appendChild(script);
```

针对网页崩溃问题，推荐一个基于 service work 的监控方案，相关文章已列在下面的。如果是 webview 加载网页，也可以通过 webview 加载失败的钩子监控网页崩溃等。

[如何监控网页崩溃？](https://juejin.im/entry/5be158116fb9a049c6434f4a)

最后，因为部署到线上的代码一般都是经过压缩混淆的，如果没有上传 sourcemap 的话，是无法定位到具体源码的，可以现在 项目中添加 .sentryclirc 文件，其中内容可参考本项目的 .sentryclirc，然后通过 sentry-cli (需要全局全装 sentry-cli 即`npm install sentry-cli`)命令行工具进行上传，命令如下：

```
sentry-cli releases -o 机构名 -p 项目名 files 版本 upload-sourcemaps sourcemap 文件相对位置 --url-prefix js 在线上相对根目录的位置 --rewrite
// 示例
sentry-cli releases -o mcukingdom -p hello-world files 0.2.1 upload-sourcemaps dist/js --url-prefix '~/js/' --rewrite
```

当然官方也提供了 webpack 插件 [sentry-webpack-plugin](https://github.com/getsentry/sentry-webpack-plugin)，当打包时触发 webpack 的 after-emit 事件钩子（即生成资源到 output 目录之后），插件会自动上传打包目录中的 sourcemap 和关联的 js，相关配置可参考本项目的 vue.config.js 文件。

通常为了安全，是不允许在线上部署 sourcemap 文件的，所以上传 sourcemap 到 sentry 后，可手动删除线上 sourcemap 文件。

## 性能监控平台

[performance](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/performance)

todo

## 部署

[Jenkins](https://jenkins.io/zh/)

[Docker](https://www.docker.com/)

Jenkins 和 Docker 相关官网如上，具体用途就不再这里赘述了，如果想了解相关知识，也可以阅读本模块结尾处推荐的几篇文章。

笔者以 CentOS 7.6 系统为基础，介绍如何使用 Github + Jenkins + Docker 实现项目的自动化打包部署。

### Docker 安装

**1.安装 Docker 并启动 Docker**

```
// 更新软件库
yum update -y

// 安装 docker
yum install docker -y

// 启动 docker 服务
service docker start

// 重启docker 服务
service docker restart

// 停止 docker 服务
service docker stop
```

**2.安装 Docker-Compose 插件用于编排镜像**

```
// 下载并安装 docker-compose (当前最新版本为 1.24.1，读者可以根据实际情况修改最新版本)
curl -L https://github.com/docker/compose/releases/download/1.24.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

// 设置权限
chmod +x /usr/local/bin/docker-compose

// 安装完查看版本
docker-compose -version
```

### Jenkins 安装和配置

**1.搜索 Jenkins**

```
docker search jenkins
```

![image](https://user-images.githubusercontent.com/22924912/68565589-f6c20900-048e-11ea-97b8-968542236015.png)

**注意**：虽然上图中第一个是 Docker 官方维护的版本，但它很长时间没有更新了，是一个过时的版本。所以这里我们要安装第二个，这个是 Jenkins 官方维护的。

**2.安装 Jenkins**

```
sudo docker run -d -u 0 --privileged --name jenkins -p 49003:8080 -v /root/jenkins_home:/var/jenkins_home jenkins/jenkins:latest
```

其中:
-d 指的是在后台运行；
-u 0 指的是传入 root 账号 ID，覆盖容器中内置的账号；
-v /root/jenkins_home:/var/jenkins_home 指的是将 docker 容器内的目录 /var/jenkins_home 映射到宿主机 /root/jenkins_home 目录上；
--name jenkins 指的是将 docker 容器内的目录 /var/jenkins_home 映射到宿主机 /root/jenkins_home 目录上；
-p 49003:8080 指的是将容器的 8080 端口映射到宿主机的 49003 端口；
--privileged 指的是赋予最高权限。

整条命令的意思就是：
运行一个镜像为 jenkins/jenkins:latest 的容器，命名为 jenkins_home，使用 root 账号覆盖容器中的账号，赋予最高权限，将容器的 /var/jenkins_home 映射到宿主机的 /root/jenkins_home 目录下，映射容器中 8080 端口到宿主机 49003 端口

执行完成后，等待几十秒，等待 Jenkins 容器启动初始化。到浏览器中输入 `http://your ip:49003` 查看 Jenkins 是否启动成功

看到如下界面说明启动成功：

![image](https://user-images.githubusercontent.com/22924912/68573362-8a510500-04a2-11ea-9f42-1f8a85812788.png)

通过如下命令获取密码，复制到上图输入框中

```
cat /root/jenkins_home/secrets/initialAdminPassword
```

进入到下个页面，选择【安装推荐的插件】。

由于墙的问题，需要修改 Jenkins 的默认下载地址。可以在浏览器另起一个 tab 页面，进入 `http://your ip:49003/pluginManager/advanced`，修改最下面的升级站点 URL 为 `http://mirror.esuni.jp/jenkins/updates/update-center.json`

![image](https://user-images.githubusercontent.com/22924912/68567715-aa79c780-0494-11ea-9b03-4311bd083470.png)

然后重启容器，再次进入初始化页面，通常下载速度会加快。

```
docker restart [docker container id]
```

然后就是创建管理员账号。

进入首页后，因为自动化部署中需要通过 ssh 登陆服务器执行命令以及 node 环境，所以需要下载 Publish Over SSH 和 NodeJS 插件，可通过系统管理 -> 管理插件 -> 可选插件进入，搜索选中并直接安装。如下图所示：

![image](https://user-images.githubusercontent.com/22924912/68568520-c41c0e80-0496-11ea-9f18-6e3d62687ee1.png)

需要注意的是，Publish Over SSH 需要配置相关 ssh 服务器，通过系统管理 -> 系统设置 进入并拉到最下面，输入 Name、Hostname、Username、Passphrase / Password 等参数。如下图所示：

![image](https://user-images.githubusercontent.com/22924912/68636791-af438780-0537-11ea-9ab8-2130d6affd8a.png)

然后点击 Test Configuration 校验能否登陆。

至此 Jenkins 已经完成了全局配置。

### 关联 Jenkins 和 Github

在 GitHub 创建一个项目，以本项目为例，在项目根目录下创建 nginx.conf 和 docker-compose.yml 文件

nginx.conf

```nginx
#user nobody;
worker_processes 1;
events {
  worker_connections 1024;
}
http {
  include    mime.types;
  default_type application/octet-stream;
  sendfile    on;
  #tcp_nopush   on;
  #keepalive_timeout 0;
  keepalive_timeout 65;
  #用于对前端资源进行 gzip 压缩
  #gzip on;
  gzip on;
  gzip_min_length 5k;
  gzip_buffers   4 16k;
  #gzip_http_version 1.0;
  gzip_comp_level 3;
  gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
  gzip_vary on;
  server {
    listen 80;
    server_name localhost;
    #前端项目
    location / {
      index index.html index.htm;  #添加属性。
      root /usr/share/nginx/html;  #站点目录
      # 所有静态资源均指向 /index.html
      try_files $uri $uri/ /index.html;
    }

    error_page  500 502 503 504 /50x.html;
    location = /50x.html {
      root  /usr/share/nginx/html;
    }
  }
}
```

docker-compose.yml

```yml
version: '3'
services:
  mobile-web-best-practice: #项目的service name
    container_name: 'mobile-web-best-practice-container' #容器名称
    image: nginx #指定镜像
    restart: always
    ports:
      - 80:80
    volumes:
      #~ ./nginx.conf为宿主机目录, /etc/nginx为容器目录
      - ./nginx.conf:/etc/nginx/nginx.conf #挂载nginx配置
      #~ ./dist 为宿主机 build 后的dist目录, /usr/src/app为容器目录,
      - ./dist:/usr/share/nginx/html/ #挂载项目
    privileged: true
```

这里需要解释下 volumes 参数，在打包 Docker 镜像时，如果将 nginx.conf 和 dist 直接拷贝到镜像中，那么每次修改相关文件时，都需要重新打包新的镜像。通过 volumes 可以将宿主机的某个文件映射到容器的某个文件，那么改动相关文件，就不要重新打包镜像了，只需修改宿主机上的文件即可。

然后在 Jenkins 创建一个新的任务，选择【构建一个自由风格的软件项目】，并设置相关配置，如下图所示。

![image](https://user-images.githubusercontent.com/22924912/68570157-edd73480-049a-11ea-9c82-c1c98c493208.png)

![image](https://user-images.githubusercontent.com/22924912/68570189-00516e00-049b-11ea-99db-3e67e2dd000f.png)

![image](https://user-images.githubusercontent.com/22924912/68573211-3a723e00-04a2-11ea-85ec-4aa3a17100f1.png)

其中第三张图两部分命令含义如下：

第一部分 shell 命令是 build 前端项目，会在项目根目录下生成 dist 目录

```
echo $PATH
node -v
npm -v
npm install
npm run build
```

第二部分 shell 命令就是通过 ssh 登陆服务器，通过 docker-compose 进行构建 docker 镜像并运行容器。相较于直接使用 docker ，当更新代码时无需执行停止删除容器，重新构建新的镜像等操作。

```
cd /root/jenkins_home/workspace/mobile-web-best-practice \
&& docker-compose up -d
```

最后可以回到该任务页，点击【立即构建】来构建我们的项目了。

### 实现自动触发打包

不过仍有个问题，那就是当向 GitHub 远程仓库 push 代码时，需要能够自动触发构建，相关操作如下。

**1.修改 Jenkins 安全策略**

通过系统管理 -> 全局安全配置 进入，并如下图操作

![image](https://user-images.githubusercontent.com/22924912/68571182-65a65e80-049d-11ea-80e8-fc63733941f8.png)

**2.生成 Jenkins API Token**

通过用户列表 -> 点击管理员用户 -> 设置，点击添加新 token，然后复制身份验证令牌 token

![image](https://user-images.githubusercontent.com/22924912/68571430-f11fef80-049d-11ea-8b42-2d51528981ea.png)

**3.在 Jenkins 项目对应任务的设置中配置【构建触发器】，将刚复制的 token 粘贴进去，如下图所示：**

![image](https://user-images.githubusercontent.com/22924912/68571656-760b0900-049e-11ea-8d42-94ed69d0a629.png)

**4.在 Github 相关项目中打开 Setting -> Webhooks -> Add webhooks，输入格式如下的 URL :**

```
// 前面是 Jenkins 服务地址，mobile-web-best-practice 指在 Jenkins 的任务名称，Token指上面获取的令牌
http://12x.xxx.xxx.xxx:xxxx/job/mobile-web-best-practice/build?token=Token
```

![image](https://user-images.githubusercontent.com/22924912/68571806-d69a4600-049e-11ea-8681-143e5282f81c.png)

这样，我们就实现了在 push 新的代码后，自动触发 Jenkins 打包项目代码，并打包 docker 镜像然后创建容器运行。

最后推荐几篇相关文章：

[写给前端的 Docker 实战教程](https://juejin.im/post/5d8440ebe51d4561eb0b2751)

[[手把手系列之]Docker 部署 vue 项目](https://juejin.im/post/5cce4b1cf265da0373719819)

[[手把手系列之] Jenkins+Docker 自动化部署 vue 项目](https://juejin.im/post/5db9474bf265da4d1206777e)

[从零搭建 docker+jenkins+node.js 自动化部署环境](https://juejin.im/post/5b8ddb70e51d45389153f288)

## 常见问题

- **iOS WKWebView cookie 写入慢以及易丢失**

  **现象：**

  1. iOS 登陆后立即进入网页，会出现 cookie 获取不到或获取的上一次登陆缓存的 cookie
  2. 重启 App 后，cookie 会丢失

  **原因：**
  WKWebView 对 NSHTTPCookieStorage 写入 cookie，不是实时存储的。从实际的测试中发现，不同的 IOS 版本，延迟的时间还不一样。同样，发起请求时，也不是实时读取，无法做到和 native 同步，导致页面逻辑出错。

  **两种解决办法：**

  1. 客户端手动干预一下 cookie 的存储。将服务响应的 cookie，持久化到本地，在下次 webview 启动时，读取本地的 cookie 值，手动再去通过 native 往 webview 写入。但是偶尔还有 spa 的页面路由切换的时候丢失 cookie 的问题。
  2. 将 cookie 存储的 session 持久化到 localSorage，每次请求时都会取 localSorage 存储的 session，并在请求头部添加 cookieback 字段，服务端鉴权时，优先校验 cookieback 字段。这样即使 cookie 丢失或存储的上一次的 session，都不会有影响。不过这种方式相当于绕开了 cookie 传输机制，无法享受 这种机制带来的安全特性。

  各位可以选择适合自己项目的方式，有更好的处理方式欢迎留言。

- **input 标签在部分安卓 webview 上无法实现上传图片功能**

  因为 Android 的版本碎片问题，很多版本的 WebView 都对唤起函数有不同的支持。我们需要重写 WebChromeClient 下的 openFileChooser()（5.0 及以上系统回调 onShowFileChooser()）。我们通过 Intent 在 openFileChooser()中唤起系统相机和支持 Intent 的相关 app。

  相关文章：
  [【Android】WebView 的 input 上传照片的兼容问题](https://juejin.im/post/5a322cdef265da43176a2913)

- **input 标签在 iOS 上唤起软键盘，键盘收回后页面不回落（部分情况页面看上去已经回落，实际结构并未回落）**

  input 焦点失焦后，ios 软键盘收起，但没有触发 window resize，导致实际页面 dom 仍然被键盘顶上去--错位。
  解决办法：全局监听 input 失焦事件，当触发事件后，将 body 的 scrollTop 设置为 0。

  ```ts
  document.addEventListener('focusout', () => {
    document.body.scrollTop = 0;
  });
  ```

- **唤起软键盘后会遮挡输入框**

  当 input 或 textarea 获取焦点后，软键盘会遮挡输入框。
  解决办法：全局监听 window 的 resize 事件，当触发事件后，获取当前 active 的元素并检验是否为 input 或 textarea 元素，如果是则调用元素的 scrollIntoViewIfNeeded 即可。

  ```ts
  window.addEventListener('resize', () => {
    // 判断当前 active 的元素是否为 input 或 textarea
    if (
      document.activeElement!.tagName === 'INPUT' ||
      document.activeElement!.tagName === 'TEXTAREA'
    ) {
      setTimeout(() => {
        // 原生方法，滚动至需要显示的位置
        document.activeElement!.scrollIntoView();
      }, 0);
    }
  });
  ```

- **唤起键盘后 `position: fixed;bottom: 0px;` 元素被键盘顶起**

  解决办法：全局监听 window 的 resize 事件，当触发事件后，获取 id 名为 fixed-bottom 的元素（可提前约定好如何区分定位在窗口底部的元素），将其设置成 `display: none`。键盘收回时，则设置成 `display: block;`。

  ```ts
  const clientHeight = document.documentElement.clientHeight;
  window.addEventListener('resize', () => {
    const bodyHeight = document.documentElement.clientHeight;
    const ele = document.getElementById('fixed-bottom');
    if (!ele) return;
    if (clientHeight > bodyHeight) {
      (ele as HTMLElement).style.display = 'none';
    } else {
      (ele as HTMLElement).style.display = 'block';
    }
  });
  ```

- **点击网页输入框会导致网页放大**
  通过 viewport 设置 user-scalable=no 即可，（注意：当 user-scalable=no 时，无需设置 minimum-scale=1, maximum-scale=1，因为已经禁止了用户缩放页面了，允许的缩放范围也就不存在了）。代码如下：

  ```html
  <meta
    name="viewport"
    content="width=device-width,initial-scale=1.0,user-scalable=0,viewport-fit=cover"
  />
  ```

- **webview 通过 loadUrl 加载的页面运行时却通过第三方浏览器打开，代码如下**

  ```java
  // 创建一个 Webview
  Webview webview = (Webview) findViewById(R.id.webView);
  // 调用 Webview loadUrl
  webview.loadUrl("http://www.baidu.com/");
  ```

  解决办法：在调用 loadUrl 之前，设置下 WebviewClient 类，当然如果需要也可自己实现 WebviewClient（例如通过拦截 prompt 实现 js 与 native 的通信）

  ```java
  webview.setWebViewClient(new WebViewClient());
  ```
