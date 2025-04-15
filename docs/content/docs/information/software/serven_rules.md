---
weight: 100
title: "设计模式与七大原则"
draft: false
---

## 设计模式概述
设计模式是**可复用面向对象软件的设计经验总结**，通过23种经典模式解决特定场景下的代码扩展性、复用性、维护性问题。其本质遵循七大核心设计原则：

---

## 七大设计原则详解

### 1. 单一职责原则（SRP）
**核心思想**：一个类只负责一个功能领域中的相应职责  
**解决问题**：  
- 避免类因承担过多职责导致的高耦合  
- 减少修改代码时引发连锁错误的风险  

**示例**：  
```java
// 错误示例：用户管理类同时处理信息存储和日志记录
class UserManager {
    void saveUser() { /* 存储用户 */ }
    void logActivity() { /* 记录日志 */ }
}

// 正确拆分
class UserRepository { void saveUser() }
class ActivityLogger { void logActivity() }
```

---

### 2. 开闭原则（OCP）
**核心思想**：对扩展开放，对修改关闭  
**解决问题**：  
- 新功能扩展无需修改原有代码  
- 提升系统稳定性与可维护性  

**示例**：  
```java
// 基础图形接口
interface Shape { double area(); }

// 新增图形只需扩展接口
class Circle implements Shape { /* 实现圆面积计算 */ }
class Square implements Shape { /* 实现正方形面积计算 */ }
```

---

### 3. 里氏替换原则（LSP）
**核心思想**：子类必须能替换父类且不影响程序正确性  
**解决问题**：  
- 防止继承关系破坏系统原有逻辑  
- 确保多态的正确使用  

**经典案例**：  
```java
// 错误设计
class Rectangle { int width, height; }
class Square extends Rectangle {} // 修改width会自动改变height

// 正确做法：通过接口定义形状
interface Quadrilateral { int getWidth(); int getHeight(); }
```

---

### 4. 依赖倒置原则（DIP）
**核心思想**：高层模块不依赖低层模块，二者都依赖抽象  
**解决问题**：  
- 降低模块间耦合度  
- 提高系统可扩展性  

**场景示例**：  
```java
interface MessageService { void send(); }

// 高层模块通过接口调用
class Notification {
    private MessageService service;
    Notification(MessageService service) { this.service = service; }
}

// 可扩展多种实现（EmailService/SMSService）
```

---

### 5. 接口隔离原则（ISP）
**核心思想**：客户端不应依赖它不需要的接口  
**解决问题**：  
- 避免接口臃肿导致实现类冗余  
- 减少接口变更对客户端的影响  

**实现方式**：  
```java
// 错误：大型接口
interface Animal {
    void eat();
    void fly(); // 企鹅不需要此方法
}

// 正确拆分
interface Swimmable { void swim(); }
interface Flyable { void fly(); }
```

---

### 6. 迪米特法则（LoD）
**核心思想**：一个对象应尽可能少了解其他对象  
**解决问题**：  
- 降低类之间的耦合度  
- 提高模块独立性  

**应用场景**：  
```java
// 直接通信（高耦合）
class A { void call(B b) { b.method(); } }

// 通过中介类（低耦合）
class Mediator {
    void process() { 
        B b = new B();
        b.method();
    }
}
```

---

### 7. 合成复用原则（CARP）
**核心思想**：优先使用组合/聚合，而非继承  
**解决问题**：  
- 避免继承破坏封装性  
- 更灵活地复用已有功能  

**对比示例**：  
```java
// 错误：通过继承实现复用
class Car extends Engine {} 

// 正确：通过组合实现复用
class Car {
    private Engine engine;
    Car(Engine engine) { this.engine = engine; }
}
```

---

## 总结
| 原则 | 核心价值 | 典型应用场景 |
|------|---------|-------------|
| SRP  | 功能聚焦 | 模块拆分、微服务设计 |
| OCP  | 稳定扩展 | 插件系统、框架设计 |
| LSP  | 继承安全 | 多态实现、API设计 |
| DIP  | 解耦抽象 | 依赖注入、中间件 |
| ISP  | 接口精简 | SDK设计、协议定义 |
| LoD  | 最小交互 | 分层架构、MVC模式 |
| CARP | 灵活复用 | 组件化开发、库封装 |

**核心目标**：通过高内聚、低耦合的设计，构建可维护、可扩展、高复用的软件系统
