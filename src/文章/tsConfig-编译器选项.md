# tsConfig 编译器选项

最终建议：只用 ts 的语法检查

## Root Fileds  根字段

### files
指定要包含在程序中的文件的允许列表。如果找不到文件，则会发生错误。 

当您只有少量文件并且不需要使用 glob 来引用许多文件时，这非常有用。如果您需要，请使用 include 

### extends
指定继承的 配置文件

### include
指定要包含在程序中的文件名或模式的数组。这些文件名是相对于包含tsconfig.json文件的目录进行解析的

### exclude
指定要排除在程序之外的文件名或模式的数组。这些文件名是相对于包含tsconfig.json文件的目录进行解析的

### references
支持项目间的引用和构建，特别是在使用项目结构（如 monorepo）时。通过配置 references，您可以将一个 TypeScript 项目作为另一个项目的依赖项，从而实现增量构建和更好的模块化

### compilerOptions
指定编译器选项，这些选项将影响 TypeScript 编译器的行为。这些选项可以覆盖项目中的 tsconfig.json 文件中的选项

#### 类型检查
##### allowUnreachableCode 允许无法访问的代码
- undefined （默认）提供建议作为对编辑的警告
- true 无法访问的代码被忽略
- false 会引发有关无法访问代码的编译器错误

##### alwaysStrict 始终严格
确保您的文件在 ECMAScript 严格模式下进行解析，并为每个源文件发出“use strict”

##### exactOptionalPropertyTypes 确切的可选属性类型
启用exactOptionalPropertyTypes后，TypeScript围绕如何处理带有 ? 前缀的type或interfaces上的属性应用更严格的规则
即 限制不能给可选属性赋值 为 undefined

##### noFallthroughCasesInSwitch Switch 中没有失败案例
报告 switch 语句中失败案例的错误。确保 switch 语句中的任何非空 case 都包含break 、 return或throw 

##### noImplicitAny 禁止隐式 any
禁止隐式 any 类型，要求所有变量和函数参数都显式声明类型
在某些不存在类型注释的情况下，当 TypeScript 无法推断类型时，它会回退到变量的any类型

##### noImplicitOverride 禁止隐式覆盖
即子类重写父类方法时，必须指定 override

##### noImplicitReturns 禁止隐式返回
必须为每个函数提供返回值定义

##### noImplicitThis 禁止隐式 this
在具有隐含“any”类型的“this”表达式上引发错误






