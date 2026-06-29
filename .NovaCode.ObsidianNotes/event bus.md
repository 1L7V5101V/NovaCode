
**Event Bus（事件总线）** 是一种在软件架构中常用的**设计模式**。

简单来说，它就像一个**中央邮局**或者**广播电台**。在复杂的系统中，不同的组件之间需要通信，如果让它们互相直接调用，系统会变得非常臃肿且难以维护（这叫强耦合）。

有了 Event Bus 之后：

1. **发布者（Publisher）**：只管把事件（消息）丢到总线上，它不知道也不关心谁会收到。
    
2. **订阅者（Subscriber）**：只管向总线注册自己感兴趣的事件，当这种事件发生时，总线就会通知它。
    

这种模式实现了组件之间的**解耦**。

### 🐍 Python 简单演示

我们用一个**电商购物**的场景来做个例子。当用户下单成功后，我们需要做两件事：

1. 发送短信通知。
    
2. 扣减库存。
    

如果没有 Event Bus，下单逻辑里就要直接调用短信模块和库存模块。现在我们用 Event Bus 来优雅地实现它。

我们用纯粹的伪代码（Pseudo-code）来抽离语言细节，看看这个模式的核心逻辑是怎么运转的：
### 1. 定义事件总线（EventBus）的结构
#### 实现 Event Bus 核心代码
```Python
class EventBus:
    def __init__(self):
        # 存储事件与订阅者的对应关系 { "event_name": [callback1, callback2] }
        self._listeners = {}

    def subscribe(self, event_type: str, listener_callable):
        """订阅事件"""
        if event_type not in self._listeners:
            self._listeners[event_type] = []
        self._listeners[event_type].append(listener_callable)

    def publish(self, event_type: str, data: dict):
        """发布事件"""
        if event_type in self._listeners:
            for listener in self._listeners[event_type]:
                listener(data)  # 触发订阅者的回调函数
```

```
CLASS EventBus:
    // 构造函数，初始化一个空字典来存储 事件 -> 监听列表 的映射
    CONSTRUCTOR():
        SET listeners = EMPTY_DICTIONARY 

    // 订阅方法：谁想听什么事件，就把自己的名字（回调函数）登记上
    METHOD subscribe(eventName, callback):
        IF eventName NOT IN listeners THEN:
            listeners[eventName] = EMPTY_LIST
        ENDIF
        APPEND callback TO listeners[eventName]

    // 发布方法：发生某事了，大声广播，并把相关数据传过去
    METHOD publish(eventName, eventData):
        IF eventName IN listeners THEN:
            FOR EACH callback IN listeners[eventName]:
                CALL callback(eventData) // 执行登记过的函数
            ENDFOR
        ENDIF
```

### 2. 定义具体的业务逻辑（订阅者）

#### 编写订阅者（业务模块）
```Python
# 短信服务
def send_sms_notification(data):
    print(f"【短信服务】收到！已向用户 {data['user_phone']} 发送订单 {data['order_id']} 的确认短信。")

# 库存服务
def decrease_inventory(data):
    print(f"【库存服务】收到！商品 {data['item_id']} 的库存已成功扣减 1 件。")
```

```
// 订阅者 A：负责发通知
FUNCTION sendNotification(data):
    PRINT "通知：您的账号在 " + data.time + " 登录了。"

// 订阅者 B：负责安全审计
FUNCTION logSecurityAudit(data):
    PRINT "日志：记录登录IP -> " + data.ip
```

### 3. 业务组装与运行流程
#### 组装并运行
```Python
# 初始化全局事件总线
bus = EventBus()

# 订阅者向总线注册自己感兴趣的事件
bus.subscribe("order_created", send_sms_notification)
bus.subscribe("order_created", decrease_inventory)

# --- 模拟用户下单 ---
print("--- 订单系统：用户开始下单 ---")
order_data = {
    "order_id": "202606090001",
    "user_phone": "13888888888",
    "item_id": "PROD_999"
}

# 下单系统只需要发布事件，根本不需要知道“短信服务”和“库存服务”的存在
bus.publish("order_created", order_data)
```

```
// 🚀 程序启动，初始化总线
SET globalBus = NEW EventBus()

// 各种服务向总线注册自己想听的广播
CALL globalBus.subscribe("user_logged_in", sendNotification)
CALL globalBus.subscribe("user_logged_in", logSecurityAudit)

// --- 之后的某个时间点，用户触发了登录操作 ---
FUNCTION onUserLoginAction():
    // 准备好事件产生的数据
    SET loginData = {
        "userId": "9527",
        "ip": "192.168.1.100",
        "time": "2026-06-09"
    }
    
    // 核心动作：登录系统只需要把事件“扔给”总线，任务就完成了
    CALL globalBus.publish("user_logged_in", loginData)
```

### 💡 伪代码运行输出效果：

Plaintext

```
通知：您的账号在 2026-06-09 登录了。
日志：记录登录IP -> 192.168.1.100
```
### 运行结果：

Plaintext

```
--- 订单系统：用户开始下单 ---
【短信服务】收到！已向用户 13888888888 发送订单 202606090001 的确认短信。
【库存服务】收到！商品 PROD_999 的库存已成功扣减 1 件。
```

### 💡 核心好处是什么？

假设过几天老板说：“用户下单后，还要给用户加 10 个积分！”

- **没有 Event Bus**：你必须去修改下单系统的核心代码，把积分模块加进去。
    
- **有了 Event Bus**：下单系统的代码**一行都不用动**。你只需要写一个积分服务的函数，然后执行 `bus.subscribe("order_created", add_points)` 即可。这就是大名鼎鼎的“开闭原则”（对扩展开放，对修改关闭）。