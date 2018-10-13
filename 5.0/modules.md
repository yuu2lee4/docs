# 模块

模块是具有 `@Module()` 装饰器的类。 `@Module()` 装饰器提供了元数据，Nest 用它来组织应用程序结构。
 
 
<center>![图1](https://docs.nestjs.com/assets/Modules_1.png)</center>

每个 Nest 应用程序至少有一个模块，即根模块。根模块是 Nest 开始安排应用程序树的地方。事实上，根模块可能是应用程序中唯一的模块，特别是当应用程序很小时，但是对于大型程序来说这是没有意义的。在大多数情况下，您将拥有多个模块，每个模块都有一组紧密相关的功能。


所述 `@Module()` 装饰采用单个对象，其属性描述该模块：

|||
|:-----:|:-----:|
|providers| 由 Nest 注入器实例化的提供者，并且可以至少在整个模块中共享|	
|controllers|必须创建的一组控制器|
|imports|导入模块所需的导入模块列表|
|exports|此模块提供的提供者的子集, 并应在其他模块中使用|

默认情况下, 模块封装提供者。这意味着无法插入既不是当前模块的直接部分，也不能从导入的模块导出提供者。

# CatsModule

`CatsController` 和 `CatsService` 属于同一个应用程序域。 应该将它们移动到功能模块 `CatsModule`。

> cats/cats.module.ts

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

我已经创建了 `cats.module.ts` 文件，并把与这个模块相关的所有东西都移到了 cats 目录下。 我们需要做的最后一件事是将这个模块导入根模块 `(ApplicationModule)`。

> app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { CatsModule } from './cats/cats.module';

@Module({
  imports: [CatsModule],
})
export class ApplicationModule {
```

现在 Nest 知道除了 `ApplicationModule` 之外，注册 `CatsModule` 也是非常重要的。 这就是我们现在的目录结构:

```text
src
├──cats
│    ├──dto
│    │   └──create-cat.dto.ts
│    ├──interfaces
│    │     └──cat.interface.ts
│    ├─cats.service.ts
│    ├─cats.controller.ts
│    └──cats.module.ts
├──app.module.ts
└──main.ts
```

# 共享模块

在 Nest 中，默认情况下，模块是单例，因此您可以毫不费力地在多个模块之间共享同一个组件实例。


<center>![图1](https://docs.nestjs.com/assets/Shared_Module_1.png)</center>

实际上，每个模块都是一个共享模块。 一旦创建就被每个模块重复使用。 假设我们将在几个模块之间共享 `CatsService` 实例。 我们需要把 `CatsService` 放到 `exports` 数组中，如下所示：

> cats.module.ts

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

现在，每个导入 `CatsModule` 的模块都可以访问 `CatsService` ，并且它们将共享相同的 `CatsService` 实例。


# 模块重新导出

模块可以导出他们的提供者。 而且，他们可以再导出自己导入的模块。

```typescript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

# 依赖注入

模块类也可以注入提供者（例如，用于配置目的）：


> cats.module.ts

```typescript
import { Module } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {
  constructor(private readonly catsService: CatsService) {}
}
```

但是，由于[循环依赖](5.0/fundamentals?id=circular-dependency)性，模块类不能由提供者注入。

# 全局模块

如果你不得不在任何地方导入相同的模块，那可能很烦人。在 Angular 中，提供者是在全局范围内注册的。一旦定义，他们到处可用。另一方面，Nest 封装了模块范围内的组件。您无法在其他地方使用模块提供者而不导入他们。但是有时候，你可能只想提供一组随时可用的东西 - 例如：helper，数据库连接等等。这就是为什么你能够使模块成为全局模块。

```typescript
import { Module, Global } from '@nestjs/common';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

@Global()
@Module({
  controllers: [CatsController],
  providers: [CatsService],
  exports: [CatsService]
})
export class CatsModule {}
```

`@Global` 装饰器使模块成为全局作用域。 全局模块应该只注册一次，最好由根或核心模块注册。 之后，`CatsService` 组件将无处不在，但 `CatsModule` 不会被导入。

!> 使一切全局化并不是一个好的解决方案。 全局模块在这里减少了必要的样板数量。 `imports` 数组仍然是使模块 API 透明的最佳方式。

# 动态模块

Nest 模块系统带有一个称为动态模块的功能。 它使您能够毫不费力地创建可定制的模块。 让我们来看看 `DatabaseModule`：

```typescript
import { Module, DynamicModule } from '@nestjs/common';
import { createDatabaseProviders } from './database.providers';
import { Connection } from './connection.provider';

@Module({
  providers: [Connection],
})
export class DatabaseModule {
  static forRoot(entities = [], options?): DynamicModule {
    const providers = createDatabaseProviders(options, entities);
    return {
      module: DatabaseModule,
      providers: providers,
      exports: providers,
    };
  }
}
```

此模块默认定义了 `Connection` 提供者，但另外 - 根据传递的 `options` 和 `entities` - 创建一个提供者集合，例如存储库。 实际上，动态模块扩展了模块元数据。 当您需要动态注册组件时，这个实质特性非常有用。 然后你可以通过以下方式导入 `DatabaseModule`：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [
    DatabaseModule.forRoot([User]),
  ],
})
export class ApplicationModule {}
```

为了导出动态模块，可以省略函数调用部分：

```typescript
import { Module } from '@nestjs/common';
import { DatabaseModule } from './database/database.module';
import { User } from './users/entities/user.entity';

@Module({
  imports: [
    DatabaseModule.forRoot([User]),
  ],
  exports: [DatabaseModule]
})
export class ApplicationModule {}
```
 ### 译者署名

| 用户名 | 头像 | 职能 | 签名 |
|---|---|---|---|
| [@zuohuadong](https://github.com/zuohuadong)  | <img class="avatar-66 rm-style" src="https://wx3.sinaimg.cn/large/006fVPCvly1fmpnlt8sefj302d02s742.jpg">  |  翻译  | 专注于 caddy 和 nest，[@zuohuadong](https://github.com/zuohuadong/) at Github  |
| [@Drixn](https://drixn.com/)  | <img class="avatar-66 rm-style" src="https://cdn.drixn.com/img/src/avatar1.png">  |  翻译  | 专注于 nginx 和 C++，[@Drixn](https://drixn.com/) |
