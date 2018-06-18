# 【译】在Angular中自动取消订阅
- 原文链接：https://netbasal.com/automagically-unsubscribe-in-angular-4487e9853a88
- 创建时间：2018-06-18
- 修改时间：2018-06-18
- 参与人员：@Zaynex

如你所知，当你在 Javascript 中订阅一个 observable 或者事件时，你通常需要在特定的时候取消订阅以释放内存。否则，就会导致[内存泄漏](https://appendto.com/2016/11/avoiding-memory-leaks-in-javascript/)

> A memory leak occurs when a section of memory that is no longer being used is still being occupied needlessly instead of being returned to the OS

当一部分内存不再使用但它仍被不必要得占用而不是返回给操作系统时就会产生内存泄漏。

在 Angular 的组件或者指令中，你需要在 ngOnDestroy 的生命周期中 unsubscribe。

比如，如果你有一个组件，它有2个订阅源。

```
@Component({
  selector: 'test',
  template: `...`,
})
export class TestComponent {
  one$;
  two$;

  constructor( private store: Store<any>, private element : ElementRef ) {}

  ngOnInit() {
    this.one$ = store.select("data").subscribe(data => // do something);
    this.two$ = Observable.interval(1000).subscribe(data => // do something);
  }

  ngOnDestroy() {
   this.one$.unsubscribe();
   this.two$.unsubscribe();
  }

}
```

你需要创建 ngOnDestroy 的方法并且给每个订阅源取消订阅。

这很不错，但是我想要这些取消订阅的过程自动化。如果我可以创建一个 `decorator` 类去帮我做这个事情会怎样呢？ 我们假设它会像下面这样：
```
@Component({
  selector: 'test',
  template: `...`,
})
@AutoUnsubscribe
export class TestComponent {
  one$;
  two$
  three;

  constructor( private store: Store<any>, private element : ElementRef) {}

  ngOnInit() {
    //...same subscriptions
  }

  // Notice that we don't have the ngOnDestroy method anymore
}
```

让我们创建一个类装饰器并且给它命名为 `AutoUnsubscribe`。

在 `Typescript` 或 `Babel` 中， 类装饰器仅仅是一个接受参数的函数，构造函数的装饰类。
(a class decorator is just a function that takes one parameter, the constructor of the decorated class.)

类装饰器作用于类的 `constructor`，并且观察、修改或者替换一个类的定义。

```
export function AutoUnsubscribe( constructor ) {

  const original = constructor.prototype.ngOnDestroy;

  constructor.prototype.ngOnDestroy = function () {
    for ( let prop in this ) {
      const property = this[ prop ];
      if ( property && (typeof property.unsubscribe === "function") ) {
        property.unsubscribe();
      }
    }
    original && typeof original === "function" && original.apply(this, arguments);
  };
}
```

🤓 这里于三个简单的步骤：
1. 保存一个引用指向原来的 `ngOnDestory` 函数。
2. 创建你自己的 `ngOnDestroy`，循环遍历类的属性并且调用 `unsubscribe` 函数，如果存在的话。
3. 调用原来的 `ngOndestroy` 函数

😁 但是...
等等，如果因为某些疯狂的理由，你仍需要保留订阅源呢？比如当组件注销的时候你并不想取消订阅 `$two` 这个订阅源。

这种情况下我们需要传入一个参数给装饰器(一个数组用来过滤自动订阅)，所以我们需要使用`Decorator Factory`：
> A Decorator Factory is simply a function that returns the expression that will be called by the decorator at runtime.
一个装饰器工厂就是一个函数，它会返回的运行期间被装饰器调用的函数。

```
export function AutoUnsubscribe(blackList = []) {
  return function(constructor) {
    const original = constructor.prototype.ngOndestroy;

    constructor.prototype.ngOndestroy = function() {
      for(let prop in this) {
        const property = this[prop];
        if(!blackList.includes(prop)) {
          if(property && (typeof property.unsubscribe === 'function')) {
            property.unsubscribe();
          }
        }
      }

      original && typeof original === 'function' && origin.apply(this, arguments);
    };
  }
}
```

我们只是针对属性名做了检测，如果它不在 `blacklist` 的名单里那我们就调用 `unsubscribe()`。

现在我们可以像这样使用 decorator：
```
@AutoUnsubscribe(["one$", "two$"])
class TestComponent {
  ...
}
```

😃 现在我们完成了！
你可以在这里找到[decorator](https://github.com/NetanelBasal/ngx-auto-unsubscribe)，如果你有更好的想法，欢迎 pull request。



如果你想要更加声明式的方式在[takeUntil](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/operators/takeuntil.md) 中使用，你可以查看我的 [tiny](https://github.com/NetanelBasal/angular2-take-until-destroy) class decorator，它有能力去做到：

```
import TakeUntilDestroy from "angular2-take-until-destroy";

@Component({
  selector: 'app-inbox',
  templateUrl: './inbox.component.html'
})
@TakeUntilDestroy
export class InboxComponent {
  componentDestroy;
  constructor( ) {
    const timer$ = Observable.interval(1000)
      .takeUntil(this.componentDestroy())
      .subscribe(val => console.log(val))

    const timer2$ = Observable.interval(2000)
      .takeUntil(this.componentDestroy())
      .subscribe(val => console.log(val))
  }
}
```

不要害怕看源码，它就那么几行！
如果你喜欢这篇文档，查看我之前的一篇————[Make your Code Cleaner with Decorators](https://medium.com/front-end-hacking/javascript-make-your-code-cleaner-with-decorators-d34fc72af947)


## 总结

你可以利用装饰器为你的应用添加强大的功能。
装饰器不是一个库也不是框架，所以要善于创造并利用好他们。
你可以探索更多的 decorators，[在这里](http://blog.wolksoftware.com/decorators-reflection-javascript-typescript)
