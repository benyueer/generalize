# SOLID原则
SOLID 设计原则是面向对象的设计原则，即：
- 单一功能（SRP）
- 开闭原则（OCP）
- 里氏替换（LSP）
- 接口隔离（ISP）
- 依赖反转（DIP）

## 单一职责原则
### 定义
一个对象应该只包含*单一的职责*，并且该职责被完整的封装在一个类中

### 分析
一个类（或者模块、方法） 承担的职责越多，他被复用的可能性越小，而且当一个类承担的职责过多时，就相当于将这些职责耦合到一起，当其中一个职责变化时，可能会影响其他职责运行

类的职责主要包括两个方面：数据职责和行为职责
数据职责通过其属性来体现，行为职责通过其方法来体现

单一职责是实现高内聚、低耦合的指导方针，它是最简单但又最难运用的原则，需要设计人员仔细思考，并在设计类的时候要特别小心，除了一个类需要完成的责任之外，不要在该类中加入其他的职责。

### 实例
视频网站用户分类，访客用户只可观看480P视频，普通用户可观看480P、但不可关闭广告，vip用户可观看4K视频、也可关闭广告
错误做法：
```java
public class VideoUserService {
  public void serveGrde(String userType) {
    if ("VIP".equals(userType)) {
      System.out.println("VIP用户可观看4K视频、也可关闭广告");
    } else if ("普通用户".equals(userType)) {
      System.out.println("普通用户可观看480P、但不可关闭广告");
    } else if ("访客用户".equals(userType)) {
      System.out.println("访客用户只可观看480P视频");
    }
  }
}
```

这样的情况下，对所有类型的用户提供支撑，违反单一职责原则

正确做法：
```java
// 抽象出一个接口
public interface IVideoUserService {
  // 视频清晰度级别
  void definition();

  // 广告
  void advertisement();
}

// 访客逻辑
public class GuestVideoUserService implements IVideoUserService {
  @Override
  public void definition() {
    System.out.println("访客用户只可观看480P视频");
  }

  @Override
  public void advertisement() {
    System.out.println("访客用户可观看广告");
  }
}

```

假如此时来了SVIP的需求，不用修改原有代码，只要新增实现类即可





## 开闭原则
在面向对象编程领域中，开闭原则（Open Closed Principle，OCP）是指软件实体（类、模块、函数等等）应该对扩展开放，对修改关闭。在程序需要进行拓展的时候，不能去修改原有的代码，实现一个热插拔的效果。简言之，是为了使程序的扩展性好，易于维护和升级。想要达到这样的效果，我们需要使用接口和抽象类

错误示范：
```java
public interface UserDAO{
  public void insert();
}

public interface UserDAOImpl implements UserDAO{
  public void insert() {
    // 基于JDBC实现数据插入
    pstmt.executeUpdate();
  }
}

// 新需求，放弃JDBC该用JNDI
public interface UserDAOImpl implements UserDAO{
  public void insert() {
    // pstmt.executeUpdate();  注释原有代码
    // 基于JNDI实现数据插入
    jndi.insert();
  }
}
```

正确做法：
对修改关闭，对扩展开放
构建新的接口来实现新的功能
```java
public interface UserDAOJndiImpl implements UserDAO{
  public void insert() {
    // 基于JNDI实现数据插入
    jndi.insert();
  }
  
}
```


## 里氏替换原则
子类可以扩展弗雷德功能，但不能改变父类原有的功能，当子类继承父类时，除添加新的方法且完成新的功能外，尽量不要重写父类的方法


实例：
储蓄卡和信用卡都有支付、提现、还款、充值等功能，但支付时，储蓄卡是扣款，信用卡是贷款

错误示范：
这里现有储蓄卡类，之后继承储蓄卡实现信用卡
```java
public class CashCard {
  private Logger logger = LoggerFactory.getLogger(CashCard.class);

  public String withdrawal(String orderId, BigDecimal amount) {
    // 扣款逻辑
    logger.info("CashCard withdrawal success, orderId={}, amount={}", orderId, amount);
    return "success";
  }

  public String recharge(String orderId, BigDecimal amount) {
    // 充值逻辑
    logger.info("CashCard recharge success, orderId={}, amount={}", orderId, amount);
    return "success";
  }
}

public class CreditCard extends CashCard {
  private Logger logger = LoggerFactory.getLogger(CreditCard.class);

  @override
  public String recharge(String orderId, BigDecimal amount) {
    // 校验
    // 还款
  }

  @override
  public String withdrawal(String orderId, BigDecimal amount) {
    // 校验

    // 生成贷款单
    // 支付成功
  }
}
```

正确做法：
将公共部分分离，形成父类
```java
public abstract class Card {
  // 正向入账 +
  public String positive(String orderId, BigDecimal amount) {

  }

  // 反向入账 -
  public String negative(String orderId, BigDecimal amount) {
    
  }
}

// 储蓄卡
public class CashCard extends Card {
  // 扣款逻辑
  public String withdrawal(String orderId, BigDecimal amount) {
    // 扣款逻辑

    super.negative(orderId, amount);
    return "success";
  }
}

// 信用卡
public class CreditCard extends Card {

  Boolean rule() {
    // 校验规则
  }
  // 还款逻辑
  public String recharge(String orderId, BigDecimal amount) {
    // 还款逻辑
    if (!rule()) {
      return "fail";
    }
    super.positive(orderId, amount);
    return "success";
  }
}

```

对公共部分进行抽象，子类根据需求实现自己功能


## 接口隔离原则
将庞大的接口拆分为更小更具体的接口，让接口中只包含合理拆分的方法



## 依赖倒置原则
高层模块不应该依赖于低层模块，二者都应该依赖于抽象，抽象不应该依赖于细节，细节应该依赖于抽象

实例：
抽奖活动具有中奖权重，一个用户的权重可能由活跃度、贡献度等计算而来

错误做法：
```java
public class User {
  private String userName;
  private Integer userWeight;
  // ...
}

// 抽奖逻辑诶
public class DrawControl {
  // 随机抽奖，返回一定量的用户作为中奖用户
  public List<User> randomDraw(List<User> userList, Integer count) {
    
  }

  // 权重获取指定数量的用户
  public List<User> weightDraw(List<User> userList, Integer count) {
    
  }

}
```


正确做法：
```java
// 抽奖接口
public interface IDraw {
  // 获取中奖用户
  public List<User> draw(List<User> userList, Integer count);
}

// 随机抽奖实现
public class RandomDrawControl implements DrawControl {
  // ...
}

// 权重抽奖实现
public class WeightDrawControl implements DrawControl {
  
}

// 开奖类
public class DrawControl {
  public List<User> draw(IDraw drawControl, List<User> userList, Integer count) {
    return drawControl.draw(userList, count);
  }

  public static main(String[] args) {
    List<User> userList = new ArrayList<>();

    // 可以把任何抽奖逻辑的实现传过来
    DrawControl drawControl = new WeightDrawControl();
    new DrawControl().draw(drawControl, userList, 10);
  }
}
```