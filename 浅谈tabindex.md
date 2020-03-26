# 谈谈 html 中的 tabIndex

hello
在开发的过程中遇到一个问题：
选中一个 div 元素，监听用户的键盘事件。

但是该元素没办法接收到键盘事件，我只能通过监听window/document 的 keydown 来判断用户的键盘输入。
虽然这种方式可行，但每次都需要判断该元素是否被选中。于是便开始了搜索之旅。

当然，我们应当明确的是 input、textarea 等表单元素是可以接受键盘事件的。

我摘录了 W3C 的一段话：
> The KeyboardEvent interface provides specific contextual information associated with keyboard devices. Each keyboard event references a key using a value. Keyboard events are commonly directed at the element that has the focus.

我们关注到最后一句: 键盘事件通常针对具有 focus 的元素。

这就是为什么这些常用的表单元素都可以接受键盘事件了。
显然，想要让一个 div 元素可以接收键盘事件，则需要将该元素调整为可 focus 的元素。
我所知的一种方法是将 div 的 contenteditable 设置为 true。
而另一种方式则是今天要讲的重点: tabindex

> 最初 tabindex 出现是做为 HTML4的一部分，提供一种手段让开发人员定义元素的顺序，可以让用户通过键盘按此顺序来获取焦点。最新在 HTML5 草案里面针对 tabindex 的具体行为表现做了改动。所有主流浏览器都已经实现了这个修改后的设计。

这个属性也是为了提高网站的易用性，即使在用户没有鼠标的情况下，也可以直接按照键盘来完成相应的操作流程。

Tab 键会聚焦可被 focus 的元素。这也是为什么在网站的一些登录表单中，输入完账号之后，按 TAB 键会切换到下一个密码输入表单的原因。


关于 tabindex 的小结三点
1. tabIndex设定 >= 0 可以使得元素具备 focus 能力
2. TAB 键的切换选中会依据 tabIndex 从小到大（正数）开始递减，然后是 tabIndex 为 0，最后是可 focus 元素（如 input、textarea）。
3. 设定了负值的 tabIndex 的表单元素，没办法被键盘事件 focus,也就是 TAB 键是无法将其选中的。


只要正确的使用 html 标签，就能满足基本的易用性，只是有时候我们不得不将某些元素去模拟 web 组件的效果。不过这模仿的过程中有很多细节值得我们学习。


### 参考资料
1. https://www.w3.org/TR/DOM-Level-3-Events/#interface-keyboardevent
2. https://developers.google.com/web/fundamentals/accessibility/focus/using-tabindex?hl=zh-cn
