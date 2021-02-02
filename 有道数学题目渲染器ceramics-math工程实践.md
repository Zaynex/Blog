#! https://zhuanlan.zhihu.com/p/339104146
## 业务背景

有道数学是为学龄前儿童提供视听触三觉互动的趣味数学思维启蒙与学习的 APP。

![有道数学客户端](https://pic4.zhimg.com/80/v2-94624a65da9ec56d299168fa68b52283.png)

其客户端整体采用 Hybrid 架构，即由 Native 客户端来负责数据持久化（存储、通信）和部分 UI 交互，由 WebView 来承载用户的学习做题等相关操作。

![有道数学客户端练习题](https://pic4.zhimg.com/80/v2-de122221ab85a0f866542b0f6cd576fa.png)
目前在有道数学有大量练习题目，而所有题目的项目都在一个 git 工程中。基于该工程，又扩展了诸如出题系统（由老师配置题目数据）、错题预览等业务场景。（在同一个工程目录中，配置多份 webpack 打包入口），实际上这些业务之间并无关联，只是都依赖了其中的题目部分代码。

由于所有需求均在一个工程中开发，维护麻烦且不利于多人协作。而随着题目数量的快速增长，工程也越来越庞大，面临后续出现的更多应用场景（如直播、难题讲解等），将会更加棘手。

题目与题目之间也没有很好的隔离，题目与业务逻辑存在部分耦合。
![当下业务依赖](https://pic4.zhimg.com/80/v2-b6fe7831f68b384f56273a6d421fbd5c.png)

可以通过这张图看到，因为底层题目与题目之间的依赖，导致上层没法单独拆分出部分业务进行独立开发，这也是导致所有需求都需要依赖该工程开发的部分原因。

基于此，团队将开发一套新的题目渲染器ceramics-math（可以理解为题目组件库），每个题目都是一个单独的组件，可单独维护发布。使用题目渲染器进行题目与业务逻辑以及题目与题目之间的解耦。

接下来介绍有道数学团队是如何进行题目渲染器 ceramics-math 的工程实践以及如何将其无缝得迁移至原有业务中。

## 工程架构实践

### 模块管理

由于所有的题目都是一个单独的组件（我们的开发目标），为了便于对题目组件的管理（迭代、维护等），我们采用了 monorepos （多包工程）的方式进行开发管理。在项目中，我们使用 lerna 进行多包管理。它带来的收益是所有组件可以共享同一份打包、测试配置。当我们的题目组件有依赖更底层的库时，可以很方便得为我们处理依赖包依赖关系。关于 lerna 和 monorepos，推荐阅读

[https://zhuanlan.zhihu.com/p/71385053](https://zhuanlan.zhihu.com/p/71385053)

[https://zhuanlan.zhihu.com/p/35237759](https://zhuanlan.zhihu.com/p/35237759)

[https://github.com/lerna/lerna](https://github.com/lerna/lerna)

### 脚手架配置

由于我们的题目组件会包含部分图片以及音频等静态资源，我们采用了 Rollup 对题目进行打包，而针对工具库（无静态资源）则使用 babel-cli 进行打包，但所有编译产物都会生成`CommonJS`、`ESM`  以及 `type` 文件。

```json
"main": "dist/index.js",
"module": "dist/index.es.js",
"types": "dist/index.d.ts",
"typings": "dist/index.d.ts",
"files": [
  "dist"
],
"sideEffects": false,
```

每个组件都会包含如下 scripts，主要用于打包、调试以及测试相关。

```json
"prebuild": "rimraf dist",
"start": "nodemon --watch src --exec yarn build -e ts,tsx",
"build": "rollup -c",
"test": "jest --env=jsdom --passWithNoTests",
"lint": "concurrently yarn:lint:*",
"test:cov": "yarn test --coverage",
"lint:src": "eslint src --ext .ts,.tsx --config ../../../.eslintrc",
"lint:types": "tsc --noEmit"
```

在项目的根目录中，通过 lerna 的 `lerna exec` 命令可以或者 `lerna run` 执行所有的组件 `build` 或 `test` 流程。

### 题目接口设计

首先在编程语言上采用了 TypeScript 进行开发，它带来的好处就不过多赘述。经过内部的题目组件需求整理后，我们定义了通用的题目组件的 Interface，表明了一个组件支持哪些 props。主要是题目组件内部的交互逻辑的状态获取，比如自身需要哪些数据，有哪些在用户答题时可触发回调的时机等。

![通用接口设计](https://pic4.zhimg.com/80/v2-f456e42e8a0d24f4401cc6195d867e10.png)

比如针对一道比大小`(CompareSelect)`的题目，当用户点击进行选择之后，业务层可以通过注册 onCheck 的 props 来获取当前用户是否答对。当上层得知用户答对后，进行加分、切换到下一题或状态保存等业务逻辑操作。

![交互展示](https://pic4.zhimg.com/80/v2-d5e79267aa8ccd3f582c5fd74e321250.png)


```tsx
// 业务层
function App() {
	const handleCheck = (isRight: boolean) => {
    // console.log(isRight)
  }
  // 题目组件
	return <CompareSelect onCheck={handleCheck}/>
}
```

![俯视图](https://pic4.zhimg.com/80/v2-6a0b462c1022d9afc1e5000e840e93d2.png)

针对定制化的题目组件需求，采用继承的方式来实现类型。
![接口继承](https://pic4.zhimg.com/80/v2-4fa6dcd2e431972a57233a7845416ad4.png)

值得一提的是，这些注释可以基于一些 AST 的手段提炼并生成相应的说明文档。明确了所有题目的接口，上层的业务可以更快速地接入题目组件库。

### 题目组件内部的原子（逻辑、组件）组合

一个题目组件其实拆分下来，也可以分为多个原子组件。比如 Button 组件，即用户点击按钮进行提交，选项卡，包含图片或者文字展示的卡片。或者是小喇叭组件（用于读题）。

![组件拆解](https://pic4.zhimg.com/80/v2-733b3c1cd3ba8c9a63f36cc31317bdff.png)

像 Button 组件其实是一个原子组件，可以应用到任何其他项目中。而 Button 组件内部，我们又封装了 `useButton` 的 hooks。而 useButton 内部又基于 usePress 的 hooks 处理了移动端和 Web 端的“点击/按压交互”。

![hooks 组合](https://pic4.zhimg.com/80/v2-ac159f8c4963302651d82d9bc233980e.png)

如上图所示，通过 hooks 可以实现更大程度的逻辑复用。团队将所有答题校验抽象成单独的 `useQuestion` 的hooks，它将整个做题流程整合在一起，原先可能每个题目有多个 useState，现在统一用单个 hooks，这种约束使得团队所有开发同学编写的核心逻辑高度类似也易于理解。此外，像更多底层通用的实现也被放到了 useQuestion 中，比如为了更友好得展示用户习题中的反馈，我们会根据答错次数播放不同的提示音，而累计错误次数的能力在 `useQuestion` 中实现。
![useQuestion](https://pic4.zhimg.com/80/v2-8f5802192e176d7011ef9f1f2a69edef.png)

在组件内部，可能还会依赖一些通用的 hooks 以及动画组件以及 utils 方法等，下图是一个题目组件可能需要的底层依赖。

![组件内部依赖](https://pic4.zhimg.com/80/v2-e84f9f3b0990abe321c8d59b705ca174.png)

### 题目组件间的全局数据、逻辑共享

在移动端开发体系中，题目组件有比较严格的适配要求（移动端、iPad 端）。题目组件内部有一套基于设计稿的适配方案，这套适配方案逻辑需要在每个组件中使用，一旦适配方案调整，那么所有组件都需要更新。我们基于 React 的 Context 将适配逻辑放在 Context 中，组件则通过 useContext  的方式获取适配结果。假设适配逻辑调整，我们也只需修改 Context 中的适配方法即可。

![组件间数据共享](https://pic4.zhimg.com/80/v2-ff214653e39cf50a25e64c2e185bd716.png)

此外，对于一些通用的反馈逻辑，比如答对播放 A 语音，答错播放 B 语音等我们也采用类似 `useFeedback` 的 hooks 进行通用处理，放在 Context 中的好处在于可以通过 `Provider` 顶层注入相应的 props 来调整 useFeedback 的反馈逻辑。

### 题板组件预览调试

有很多题目都存在复杂的交互（如拖拽）和动画，包括播放音频等交互场景，仅编写单元测试是不够的。我们引入了 Storybook 作为组件的预览开发调试工具。Storybook 在目前比较流行的响应式  UI 开发调试工具（支持多种前端框架，包括 React)，在许多 github 社区开源的组件（库）中均有应用。

![storybook](https://pic4.zhimg.com/80/v2-8374207c325f29e96bd7ec8a037eafc4.png)

这是我们项目中应用 Storybook 开发的应用示例。如果仔细观察的话，不难发现之前编写的接口注释此时会在 Storybook 中显示。其中的 controls 面板是可以由使用者实时调整配置的。

当前端同学完成题目的开发后，可以发布 Storybook 文档，产品、交互同学可以直接通过线上的地址走查 UI 交互细节。而原来则需要通过打包到客户端才能让产品测同学验收（可能需要反复打包走查），这也大幅提升了验收体验。

### 题目组件模板

在介绍题目组件模板之前，我们先看看题目组件一共有哪些相同的文件。
![组件文件结构](https://pic4.zhimg.com/80/v2-c2604d1fb2ce164a2c02e7c334dbe04b.png)

对于 package.json、roolup.config.js、jest.config、tsconfig.json 等配置文件大部分几乎都是一样的，而且唯一不同的是组件的名称（在 package.json 中体现）。此外，所有的题目组件又有一份通用的 props 接口文件，包括所有题目组件都需要设置背景，我们可以将这些通用的不论是配置还是组件内的逻辑都可以通过模板的方式自动生成。

为此，我们引入了命令行工具，帮助我们手动创建模板

![自动化生成模板](https://pic4.zhimg.com/80/v2-e9bd169b95cd6aab875d37a9ad035c80.png)

通过交互式命令行输入可动态生成 package.json 的名称，并将模板生成在指定的目录下。

现有的提炼出来的模板比较少，配置选项也非常有限。后续可以通过比如是否有提交按钮的组件等命令行交互动态生成更多的组件代码。

### 基于 gitlab-ci 的发布流程

在开发并验收完成之后，我们需要真正进入到组件发布阶段。使用 lerna 可以帮助更新组件相应的依赖版本号并且发布，还会在组件发布之后打 tag 以及生成 changelog。这套流程高度自动化。但原先我们是在本地进行打包发布，这不仅占用开发同学的机器资源，还要中断流程进行发布。如果打包过程中出现问题，又需要中断处理。这里有两个问题要解决：

1. 如何自动发布
2. 如何避免出现 release 时 build 失败导致中断

针对问题2，大部分出现的场景是 TS 类型不正确，或者单元测试未通过。每次改完代码都要本地重新 build、test 一遍确认无误后提 pull request 又非常影响效率。因此我们引入了 gitlab ci 来解决。当开发同学完成开发后，push branch 到 gitlab，gitlab ci 检测到有新的分支 push 后，会自动执行相应的任务，在我们的业务中会对所有 push 的分支进行 build 和 test 操作。如果 build 和 test 通过才允许合并该分支。

![gitlab-ci](https://pic4.zhimg.com/80/v2-f68718f6205088c1d201c02db687ff9e.png)

针对问题1，其实也是基于 gitlab ci。我们设置了只允许 master branch 发布。当 master 有新的提交时，gitlab ci 会再次触发 build、test 操作，当前两步都完成之后，最后进入发布阶段。目前我们将所有组件发布在私有 npm 仓库中。需要注意的是，由于 ci 中会有打 tag，commit changelog 的场景（lerna 内集成），会再次 push 到 master，需要避免循环触发 ci 的 task，因此务必在 gitlab-ci.yml 文件中设置好 `except` 字段。

### 整体工作流

我们回顾下整体工作流，开发同学通过命令行工具新建组件模板，再通过 `lenra bootstrap` 使得新建的组件建立依赖关系。开发完成后 push 到 feature 分支进行 gitlab-ci 执行 build 和 test。并且部署 storybook ，由产品设计同学验收。待分支稳定后合并到 develop，再提稳定分支到 master 进入发布流程。

![workflow](https://pic4.zhimg.com/80/v2-05b36f700c8f38ca3d02987a96149b79.png)

### 上层接入指南和迁移

组件发布之后，业务层接入其实非常简单。

```tsx
import { CeramicsProvider, CompareSelectQuestion } from '@ceramics-math/core'
function App() {
	return <CeramicsProvider>
		<CompareSelectQuestion {...props}/>
	</CeramicsProvider>
}
```

由于我们使用了 TypeScript，在 VSCode 中编写 props 时出现智能提示，如果开启相应的 lint 配置，还可以做到更严格的开发提示。

那如何在不影响原有业务逻辑的情况下接入 ceramics-math 呢？

我们在原有的项目中通过配置字段以进行区分。如果是 ceramics-math 的题目则进入到新的业务组件并且加载对应的题目组件，反之按原有逻辑处理。

有了题目组件之后，所有上层的逻辑可以拆分放到各自独立的工程中处理，面对未来的场景如小程序或者是直播等采用这种方式隔离各个业务。上层应用无需关注题目组件本身的实现，只需根据业务的需要引入即可。

![业务解耦](https://pic4.zhimg.com/80/v2-8489e9a1aca05d39b90e30019147410b.png)

## 总结与展望

在使用 lerna 进行多包开发调试的时候，会有些问题。比如当你的组件A 依赖模块B时，B 的修改并不会使得 A 更新，需要重新本地 build 模块 B 才行。因此我们在开发时还需要为特定的模块执行 `start` 命令，对于刚接触多包管理开发的同学可能会有些不适应，而且实际 ceramics 由于开启 hoist 功能会导致在其他工程内调试组件时会出现不同版本的 React，虽然 link 可以解决问题，但实际调试过程中 link 的方案依然有些缺陷。目前我们将所有静态资源打包在内，文件体积较大，但后续也会有相应的措施。团队在组件的单元测试覆盖率还较低...

此外，Storybook 的 addons 还有很多与业务结合的空间，比如直接在 Storybook 配置出题数据而无需额外开发出题系统。
虽然目前有很多小问题，但团队也一直在寻找优化的方案不断迭代和完善。

以上是有道数学题目渲染器 ceramics-math 的工程实践，仓促写下难免有些遗漏，有需要改善的意见/建议欢迎提出，谢谢。

### 参考资源

- [chakra-ui](https://chakra-ui.com/)
- [react-spectrum](https://react-spectrum.adobe.com/)
- [storybook](https://storybook.js.org/)
- [gitlab-ci](https://docs.gitlab.com/ee/ci/yaml/)
- [rollup](https://www.rollupjs.org/)
- [babel](https://babeljs.io/)
- [lerna](https://github.com/lerna/lerna)
- [基于lerna和yarn workspace的monorepo工作流](https://zhuanlan.zhihu.com/p/71385053)
- [使用lerna优雅地管理多个package](https://zhuanlan.zhihu.com/p/35237759)
