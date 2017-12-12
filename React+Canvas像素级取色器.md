## 前言
有时候我们需要通过图片去获得具体像素的颜色。而强大的 Canvas 为我们提供了现成的接口。 这个功能其实并不难，只不过我们需要正确的理解 Canvas 并学会利用它的 API 。
如果你急于看到效果，可以直接访问这个地址：https://zaynex.github.io/Breathe/
源码地址：https://github.com/Zaynex/Breathe/blob/master/src/Picker/index.js
我不会详细得写下每一个步骤，但是你可以一边参照源码，一边配合这篇教程进行阅读。

## 绘制图片(-)

首先，我们需要基于图片去绘制 Canvas。
操作步骤
1. 创建一个与图片等宽高的canvas
2. 获得canvas的 context
3. 将图片绘制到 canvas

我们在React中用最小化模型展示出来
我们在 React 的 DidMount 里拿到 image 实例。当然，你也可以直接创建一个 image 对象。


```
import React, { PureComponent } from 'react'
import PropTypes from 'prop-types'

export class TestPicker extends PureComponent {
  static propTypes = {
    src: PropTypes.string.isRequired,
    width: PropTypes.number.isRequired,
    height: PropTypes.number.isRequired,
  }
  static defaultProps = {
    width: 1300,
    height: 769,
    src: '/sec3.png'
  }
  // 在初始化阶段注册 ref 回调函数去获得 DOM 的实例
  constructor (props) {
    super(props)
    this.imageCanvasRef = ref => this.imageCanvas = ref
    this.image = new Image()
    this.image.src = props.src

  }
  // 请注意，一定要在图片加载完全之后才开始绘制 Canvas
  componentDidMount () {
    this.image.onload = () => this.renderImageCanvas()
  }

  renderImageCanvas = () => {
    const { width, height } = this.props
    this.imageCtx = this.imageCanvas.getContext('2d')
    this.imageCtx.drawImage(this.image, 0, 0, width, height)
  }

  render () {
    const { width, height, src } = this.props
    return <div>
      <canvas
        width={width}
        height={height}
        style={{ width, height }}
        ref={this.imageCanvasRef}>
      </canvas>
    </div>
  }
}
```
只要将它挂载到相应的节点下，你可以看到有一个和图片一样大小的 Canvas 并且绘制了图片。


但是我们需要注意图片应该是同源的，如果不是同源，Canvas 绘制图片时会报错。具体如何设置可以参考 
[使用图像 Using images](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API/Tutorial/Using_images)


### Canvas 画布与实际宽高
本质上canvas的宽高设定包含两个层面，一个是画布的大小，另外一个则是Canvas 在文档对象所占据的宽高。由于Canvas内部的绘制区域画布大小默认是（width: 300px, height: 150px) ，比如当你 通过 css 设定 （width: 3000px;height: 1500px)的时候，内部的绘制区域大小会被强制与整体宽高保持统一，即内部的绘制区域会被放大十倍。像素级别的放大会导致实际的渲染效果变得更加模糊。因为要注意有时候你的绘制区域出现缩放现象。


### 实现放大镜位移（二）
我们需要让放大镜的位置在鼠标正中心，并且跟随鼠标移动。
实现方式也比较简单，通过 onmousemove 时获得当前 clientX 和 clientY, 并且减去当前 Canvas 视窗所占据的 left 和 top 即可。

首先，我们在构造函数加了初始化的 state用来表示当前鼠标位移。
在鼠标移动时触发 onmousemove 时去修改 state,通过改变 state 触发 re-render,修改 left 和 top。

```

constructor() {
	this.glassCanvasRef = ref => this.glassCanvas = ref
	this.state = {
    left: 0,
    top: 0
  }
}
handleMouseMove = (e) => {
  // 计算当前鼠标相对 canvas 中的位置
  this.calculateCenterPoint({ clientX: e.clientX, clientY: e.clientY })
  const { centerX, centerY } = this.centerPoint
  this.setState({ left: centerX, top: centerY })
}

calculateCenterPoint = ({ clientX, clientY }) => {
  const { left, top } = this.imageCanvas.getBoundingClientRect()
  this.centerPoint = {
    centerX: Math.floor(clientX - left),
    centerY: Math.floor(clientY - top)
  }
}
render () {
	const { width, height, src } = this.props
	const { left, top } = this.state
	return <div style={{ position: 'relative' }}>
	  <canvas
	    width={width}
	    height={height}
	    style={{ width, height }}
	    onMouseMove={this.handleMouseMove}
	    ref={this.imageCanvasRef}>
	  </canvas>
	  <canvas 
	  ref={this.glassCanvasRef}
	  className="glass" 
	  style={{ left: left - glassWidth/2, top: top - glassHeight/2, width: glassWidth, height: glassHeight }}>
	  </canvas>
	</div>
}
const glassWidth = 100
const glassHeight = 100
```

### 绘制放大区域内容（三）
好了，其实我们完成快一半了。接下来就是把放大区域部分的图像放置到我们的放大镜中。
在绘制之前，我们先清除一次画布
```
handleMouseMove = (e) => {
    this.glassCtx.clearRect(0, 0, glassWidth, glassWidth)
}
```

我们希望将放大镜部分的元素放大, 我默认取了10倍放大效果。这种情况呈现的样式比较友好，如果你还需要对元素再放大，你只需要修改 scale 即可。
```
const INIT_NUMBER = 10
const finallyScale = INIT_NUMBER * (scale < 1 ? 1 : scale)
```

接下来我们使用 canvas 提供的 drawImage 的复杂版本进行截取部分图像。
[CanvasRenderingContext2D.drawImage()](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/drawImage)
根据 MDN 中的演示图片，我们知道 
1. sx 和 sy 是原图到我们需要绘制的放大镜放大的左侧以及顶部距离
2. sWidth 和 sHeight 是我们要选择放大的部分
3. dx 和 dy 是当前绘制内容在放大镜中的偏移量
4. dWidth 和 dHeight 即为放大镜大小

```
drawImageSmoothingEnable(this.glassCtx, false)
this.glassCtx.drawImage(this.image,
  Math.floor(centerX - (glassWidth / 2) / finallyScale), Math.floor(centerY - (glassHeight / 2) / finallyScale),
  Math.floor(glassWidth / finallyScale), Math.floor(glassHeight / finallyScale),
  -INIT_NUMBER, -INIT_NUMBER,
  glassWidth, glassHeight
)

const drawImageSmoothingEnable = (context, enable) => {
  context.mozImageSmoothingEnabled = enable
  context.webkitImageSmoothingEnabled = enable
  context.msImageSmoothingEnabled = enable
  context.imageSmoothingEnabled = enable
}
```
我们需要计算放大后的因素。此外，由于在计算鼠标当前位置时，可能会有1像素偏差，但被放大了10倍。所以我增加了10个像素的偏移量。你可以根据实际情况来决定偏移。

通过drawImageSmoothingEnable函数让我们最终绘制的图像产生锯齿效果。这样就会有真实的像素风格了。


### 绘制网格线(四）
关于绘制网格线，依然可以参考 MDN 上的文档。
```
const GRID_COLOR = 'lightgray'
drawGrid(this.glassCtx, GRID_COLOR, INIT_NUMBER, INIT_NUMBER)

const drawGrid = (context, color, stepx, stepy) => {
  context.strokeStyle = color
  context.lineWidth = 0.5

  for (let i = stepx + 0.5; i < context.canvas.width; i += stepx) {
    context.beginPath()
    context.moveTo(i, 0)
    context.lineTo(i, context.canvas.height)
    context.stroke()
  }

  for (let i = stepy + 0.5; i < context.canvas.height; i += stepy) {
    context.beginPath()
    context.moveTo(0, i)
    context.lineTo(context.canvas.width, i)
    context.stroke()
  }
}
```

### 实现取色（五）
我们通过 getImageData 获得具体的像素点的数据，不过还需要转换一下才能变成可用的数据。
```
getColor = () => {
  const { centerX, centerY } = this.centerPoint
  const { data } = this.imageCtx.getImageData(centerX, centerY, 1, 1)
  const color = transform2rgba(data)
}

const transform2rgba = (arr) => {
  arr[3] = parseFloat(arr[3] / 255)
  return `rgba(${arr.join(', ')})`
}

```


### 结语
原本我实现了一个在 Canvas 里又绘制一个放大镜去放大图像。但这样的问题就是放大镜只能在 Canvas 内部活动，添加样式之类的需要通过 Canvas 绘制，失去了 CSS 的能力。
现在这种分离的方式可以支持自定义 CSS 样式，而且减少了 Canvas 中继续绘制 Canvas 放大倍数的复杂度。

当然，这只是一个 启发性的demo，依然有许多粗糙的地方。希望能对你有用~











