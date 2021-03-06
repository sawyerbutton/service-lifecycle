# ServiceLifecycle

## Angular中服务的生命周期

> Angular官方教程中只提及了生命周期在组件,指令中的应用而并没有提及服务也有其生命周期

> service同样可以实现OnDestroy, OnInit, AfterViewInit, AfterViewChecked, AfterContentChecked, AfterContentInit接口

> 需要注意的是只有`OnDestroy`接口有实际的调用效果

> 也就是说可以通过`constructor`和`OnDestroy`接口实现对service的监控效果


### 声明在Module中的Service生命周期

> 如果服务申明于模块之中,模块中的每个部分都将会共享同样的服务实例

> 服务将会在某个时间点被构造出来,而这个时间点和依赖注入了该服务的 `组件/ 指令 / 服务 / 管道` 初次被创建的时间点有关
  
> 服务被创建后不会跟随被注入的 `组件/ 指令 / 服务 / 管道`的销毁而被销毁, 而是跟随模块的销毁而销毁

```typescript
// ...
@NgModule({
  declarations: [
    AppComponent,
    HelloComponent
  ],
  imports: [
    BrowserModule
  ],
  bootstrap: [AppComponent],
  providers: [GlobalService]
})
```

> 需要注意的是,在同步`NgModule` 里面配置的 `provider` 在整个 APP 范围内都是可见的,

> 亦即,即使在某个子模块里面配置的`provider`服务,它们依然是全局可见的,可以被注射到任意类里面

> 更需要注意的是,异步模块里面配置的`providers`只对本模块中的成员可见

> 如果在其他模块里面引用异步模块里面配置的`provider`会产生异常

> 其本质原因是,Angular会给异步加载的模块创建独立的注射器树
           


### 直接声明于更小颗粒中的Service生命周期

> 如果服务直接申明于` 组件 / 指令 / 管道` 中时,服务会在注入的 `组件 / 指令/ 管道`被创建时创建对于的服务实例

> 该实例依存于创被建的 ` 组件 / 指令 / 管道`,并随着` 组件 / 指令 / 管道`的销毁而销毁

```typescript
// hello.component.ts
@Component({
  selector: 'app-hello',
  templateUrl: './hello.component.html',
  styleUrls: ['./hello.component.css'],
  providers: [LocalService]
})
```

> 如果将service配置在 `组件 / 指令 / 管道`内部的 `providers` 中,服务奖不再是单例模式了

> 每个`组件 / 指令 / 管道`都拥有自己独立的 UserListService 实例, 其内部的`provider`生命周期与其自身保持一致,这才是service生命周期的关键
                                                

### Service的注入机制

- 使用组件举例避免`providers`无法插入到`pipe`中的情况,以下组件可以替换为指令进行适配,注入机制同样适用于指令的某些场景

- 如果组件内部配置了`providers`优先使用组件上的配置来创建注入对象,否则向父层组件继续查找,父组件上找不到继续向所属的模块查找,一直查询到根模块 `AppModule` 里面的`providers` 配置

- 最终若未找到指定的服务,则抛出异常

- 同步模块里面配置的 `providers` 是全局可见的,即使是很深的子模块里面配置的 `providers`依然是全局可见的(与深度无关)

- 异步模块里面配置的`providers`只对本模块中的成员可见

- 组件里面配置的 `providers` 对组件自身和所有`子层组件`可见

- `injector`的生命周期与组件自身保持一致,当组件被销毁的时候,对应的`injector`也会被销毁

### @Inject 和 @Injectable

> 与常见的通过`@Injectable`为服务生成元数据之外,也可以通过`@Inject`作为手动挡的方式手动指定元数据类型信息

```typescript
import {Inject, Injectable} from '@angular/core';
import { UUID } from 'angular2-uuid';
import {HttpClient} from '@angular/common/http';

export class InjectTestService {
  private _id: string;
  constructor(
    @Inject(HttpClient) private http
  ) {
    this._id = UUID.UUID();
  }
}
```

> 值得注意的是,用` @Inject` 和用 `@Injectable` 最终编译出来的代码是不一样的

> 用 `@Inject` 生成的代码会多于`@Injectable`导致最后生成的文件变大

#### @Inject的另类用法

> `@Inject`不仅可以注入强类型的对象也可以注入弱类型的对象字面值

```typescript
import { Injectable, Inject } from '@angular/core';
import {MY_CONFIG_TOKEN} from '../my.config';

// @Injectable({
//   providedIn: 'root'
// })
export class LiteralService {

  constructor(
    @Inject(MY_CONFIG_TOKEN) config: object
  ) {
    console.log(config);
  }
}
```

> 前提是

```typescript
import { InjectionToken} from '@angular/core';

export const MY_CONFIG = {
  name: '@Inject标签测试'
};

export const MY_CONFIG_TOKEN = new InjectionToken<string>('my_config.ts');
```

```typescript
@NgModule({
  declarations: [
    AppComponent,
    HelloComponent,
    TestDirective
  ],
  imports: [
    BrowserModule
  ],
  bootstrap: [AppComponent],
  providers: [GlobalService, LiteralService, {provide: MY_CONFIG_TOKEN, useValue: MY_CONFIG}]
})
```

> 只是事实上上述方式也可以通过直接将配置文件直接通过`import`引入服务中实现

> 使用上述方式更多的是一种屠龙之术,只是龙在Angular中是存在的


### @Self标签

> 对于子组件而言,其可以使用在父组件的层面上定义的服务

![non self tag](./src/assets/angury1.png)

> 由于`SonComponent`嵌套在`FatherComponent`内部,而且其自身没有配置 `providers`
> 所以其共享了父层的`inject-test-service`实例


- 利用`@Self` 装饰器来提示注射器不要向上查找只在组件自身内部查找依赖

> 需要注意的是,使用`@Self`标签后需要注意本组件中是否包含所需依赖`providers`

> 若注射器没有找到相应的依赖则会抛出错误

```typescript
import {Component, OnInit, Self} from '@angular/core';
import {InjectTestService} from '../../services/inject-test.service';

@Component({
  selector: 'app-self-son',
  templateUrl: './self-son.component.html',
  styleUrls: ['./self-son.component.css'],
  providers: [InjectTestService]
})
export class SelfSonComponent implements OnInit {
  id$: any;
  constructor(
    @Self() private injectTestService: InjectTestService
  ) { }

  ngOnInit() {
    this.id$ = this.injectTestService.getServiceId();
  }
}
```

![self tag](./src/assets/angury2.png)

### @SkipSelf和@Optional

> 见名知意系列之 `@SkipSelf`

> 采用`@SkipSelf`标签将会使注射器跳过组件自身,沿着`injector tree`向上寻找依赖

```typescript
import {Component, OnInit, SkipSelf} from '@angular/core';
import {InjectTestService} from '../../services/inject-test.service';

@Component({
  selector: 'app-skip-self-son',
  templateUrl: './skip-self-son.component.html',
  styleUrls: ['./skip-self-son.component.css'],
  providers: [InjectTestService]
})
export class SkipSelfSonComponent implements OnInit {
  id$: any;
  constructor(
    @SkipSelf() private injectTestService: InjectTestService
  ) { }

  ngOnInit() {
    this.id$ = this.injectTestService.getServiceId();
  }
}
```

![skip self tag](./src/assets/angury3.png)

>可以组合使用` @SkipSelf `与 `@Optional`

```typescript
@SkipSelf() @Optional() public injectTestService: InjectTestService
```

> 使用` @Optional `装饰器后如果在父层上找到了指定的依赖类型则会据此创建实例,否则,直接设置依赖为null并不抛异常(_存疑_)

![skip self optional tag](./src/assets/angury4.png)

## @Host与Content Projection

> Content Projection 的作用是动态指定组件所需要的内容

![skip self tag](./src/assets/angury5.png)

- `@Host`装饰器会提示注射器:在组件自己内部查找需要的依赖或到 `Host（宿主）`上去查找

- 换言之`@Host` 装饰器是`被投影的组件`和`其宿主`之间构建联系的纽带

> 如果宿主上面并没有所需要的依赖时可以使用`@Optional` 装饰器避免程序崩溃(抛出异常不可避)

### Use Injector Manually(待定)

### Providers 和 viewProviders

> 当在component中使用`viewProviders`提供providers时,注入在父组件的服务奖不会注入到通过投影引入的内容

> 如果希望像上述的方式在将注入在父组件的服务同样提供给投影内容，需要使用`Providers`实现

> `viewProviders`的实际作用可以理解为限制`providers`注入只面向子组件而不面向投影组件

> `viewProviders`防止投影内容扰乱服务，其家住在构建angular library时将会得到体现

```typescript
import { Component, OnInit } from '@angular/core';
import {InjectTestService} from '../../services/inject-test.service';

@Component({
  selector: 'app-view-provider-father',
  templateUrl: './view-provider-father.component.html',
  styleUrls: ['./view-provider-father.component.css'],
  // viewProviders: [InjectTestService]
  providers: [InjectTestService]
})
export class ViewProviderFatherComponent implements OnInit {
  id$: any;
  constructor(
    private injectTestService: InjectTestService
  ) { }
  ngOnInit() {
    this.id$ = this.injectTestService.getServiceId();
  }
}
```

> 当是用`viewProviders`属性作为元数据时,投影内容无法获取父组件中的注入的服务，所以会报出`NullInjectorError: NoProvider for InjectTestService!

![view provider bug](./src/assets/bug1.png)




