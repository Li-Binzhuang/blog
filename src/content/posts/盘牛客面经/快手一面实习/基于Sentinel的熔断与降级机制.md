# 基于 Sentinel 的熔断与降级机制(Alibaba Sentinel 熔断降级完整指南)

## 一、核心概念

### 1. 熔断 vs 降级

- **熔断（Circuit Breaking）**：当下游服务出现问题时，自动切断调用，避免级联故障
- **降级（Degradation）**：服务降级到更低级别的服务质量，保证核心功能可用

### 2. Sentinel 的工作原理

```
请求 → 资源定义 → 规则判断 → 熔断/降级决策 → Fallback 处理
```

---

## 二、熔断触发条件详解

### 1. **慢调用比例策略**

```java
// 场景：当慢调用（RT > 阈值）比例超过设定值时熔断
DegradeRule slowCallRule = new DegradeRule("orderService")
    .setGrade(RuleConstant.DEGRADE_GRADE_RT)           // RT模式
    .setCount(500)                                      // 最大响应时间 500ms
    .setTimeWindow(10)                                  // 熔断时长 10秒
    .setMinRequestAmount(5)                             // 最小请求数
    .setSlowRatioThreshold(0.6)                        // 慢调用比例 60%
    .setStatIntervalMs(1000);                          // 统计时长 1秒
```

**触发逻辑**：

- 在 1 秒统计窗口内
- 请求数 ≥ 5
- 响应时间 > 500ms 的请求占比 ≥ 60%
- 触发熔断 10 秒

### 2. **异常比例策略**

```java
DegradeRule exceptionRatioRule = new DegradeRule("paymentService")
    .setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO)  // 异常比例模式
    .setCount(0.5)                                          // 异常比例 50%
    .setTimeWindow(15)                                      // 熔断 15 秒
    .setMinRequestAmount(10)                                // 最小请求数
    .setStatIntervalMs(1000);                              // 统计周期
```

### 3. **异常数策略**

```java
DegradeRule exceptionCountRule = new DegradeRule("smsService")
    .setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_COUNT)  // 异常数模式
    .setCount(10)                                           // 异常数阈值
    .setTimeWindow(60)                                      // 熔断 60 秒
    .setStatIntervalMs(60000);                             // 60秒统计窗口
```

---

## 三、降级策略实现

### 1. **注解方式配置**

```java
@Service
public class OrderService {

    /**
     * 基础降级配置
     * blockHandler: 处理限流/熔断
     * fallback: 处理业务异常
     */
    @SentinelResource(
        value = "createOrder",
        blockHandler = "createOrderBlockHandler",
        fallback = "createOrderFallback"
    )
    public OrderResult createOrder(OrderRequest request) {
        // 调用下游服务
        PaymentResult payment = paymentService.pay(request);
        InventoryResult inventory = inventoryService.deduct(request);
        return new OrderResult(payment, inventory);
    }

    // 熔断/限流时的处理逻辑
    public OrderResult createOrderBlockHandler(OrderRequest request,
                                               BlockException ex) {
        log.warn("订单服务被熔断: {}", ex.getClass().getSimpleName());
        return OrderResult.builder()
            .success(false)
            .message("系统繁忙，请稍后重试")
            .orderId(null)
            .build();
    }

    // 业务异常时的降级逻辑
    public OrderResult createOrderFallback(OrderRequest request,
                                           Throwable ex) {
        log.error("订单创建失败，执行降级: ", ex);

        // 返回降级数据
        return OrderResult.builder()
            .success(false)
            .message("订单提交失败，请重试")
            .orderId(generateTempOrderId())
            .needRetry(true)
            .build();
    }
}
```

### 2. **编程式配置（规则 API）**

```java
@Configuration
public class SentinelRuleConfig {

    @PostConstruct
    public void initDegradeRules() {
        List<DegradeRule> rules = new ArrayList<>();

        // 支付服务熔断规则
        rules.add(new DegradeRule("paymentService")
            .setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO)
            .setCount(0.3)              // 异常率 30%
            .setTimeWindow(30)          // 熔断 30 秒
            .setMinRequestAmount(20)
            .setStatIntervalMs(1000));

        // 库存服务慢调用熔断
        rules.add(new DegradeRule("inventoryService")
            .setGrade(RuleConstant.DEGRADE_GRADE_RT)
            .setCount(1000)             // RT > 1秒
            .setSlowRatioThreshold(0.5) // 慢调用 50%
            .setTimeWindow(20)
            .setMinRequestAmount(10));

        DegradeRuleManager.loadRules(rules);
    }
}
```

### 3. **动态配置（Nacos 集成）**

```java
@Configuration
public class SentinelNacosConfig {

    @Bean
    public Converter<String, List<DegradeRule>> degradeRuleConverter() {
        return source -> JSON.parseArray(source, DegradeRule.class);
    }

    @PostConstruct
    public void initNacosDataSource() {
        String remoteAddress = "localhost:8848";
        String groupId = "SENTINEL_GROUP";
        String dataId = "degrade-rules";

        ReadableDataSource<String, List<DegradeRule>> degradeRuleDataSource =
            new NacosDataSource<>(remoteAddress, groupId, dataId,
                source -> JSON.parseArray(source, DegradeRule.class));

        DegradeRuleManager.registerDataSource(degradeRuleDataSource);
    }
}
```

---

## 四、多级降级策略设计

### 1. **核心链路保护架构**

```java
@Service
public class MultiLevelDegradeService {

    /**
     * 三级降级策略：
     * Level 1: 正常调用完整服务
     * Level 2: 调用缓存数据
     * Level 3: 返回兜底数据
     */
    @SentinelResource(
        value = "getProductDetail",
        blockHandler = "getProductDetailBlock",
        fallback = "getProductDetailFallback"
    )
    public ProductDetail getProductDetail(Long productId) {
        // Level 1: 完整服务
        ProductDetail detail = productService.getDetailFromDB(productId);
        RecommendList recommends = recommendService.getRecommends(productId);
        ReviewList reviews = reviewService.getReviews(productId);

        return ProductDetail.builder()
            .basic(detail)
            .recommends(recommends)
            .reviews(reviews)
            .build();
    }

    // Level 2: 缓存降级
    public ProductDetail getProductDetailFallback(Long productId, Throwable ex) {
        log.warn("降级到缓存数据: productId={}", productId);

        // 尝试从缓存获取
        ProductDetail cached = redisCache.get("product:" + productId);
        if (cached != null) {
            cached.setFromCache(true);
            return cached;
        }

        // 缓存未命中，继续降级
        return getProductDetailDefaultValue(productId);
    }

    // Level 3: 兜底数据
    private ProductDetail getProductDetailDefaultValue(Long productId) {
        log.warn("使用兜底数据: productId={}", productId);

        return ProductDetail.builder()
            .productId(productId)
            .available(false)
            .message("商品信息暂时无法获取")
            .recommends(Collections.emptyList())
            .reviews(Collections.emptyList())
            .build();
    }

    // 熔断处理
    public ProductDetail getProductDetailBlock(Long productId, BlockException ex) {
        log.error("服务熔断: productId={}, reason={}",
                  productId, ex.getClass().getSimpleName());

        // 直接返回 Level 3 兜底数据
        return getProductDetailDefaultValue(productId);
    }
}
```

### 2. **核心与非核心链路分离**

```java
@Service
public class OrderProcessService {

    // 核心链路：订单创建（必须成功）
    @SentinelResource(
        value = "coreOrderCreate",
        blockHandler = "coreOrderBlockHandler"
    )
    public OrderResult createCoreOrder(OrderRequest request) {
        // 只保留核心逻辑
        String orderId = orderService.createOrder(request);
        paymentService.createPayment(orderId, request.getAmount());

        // 异步处理非核心逻辑
        asyncExecutor.execute(() -> {
            try {
                sendNotification(orderId);
                updateRecommendSystem(request);
            } catch (Exception e) {
                log.error("非核心逻辑失败，但不影响订单", e);
            }
        });

        return OrderResult.success(orderId);
    }

    // 非核心链路：营销活动（可降级）
    @SentinelResource(
        value = "marketingPromotion",
        fallback = "marketingPromotionFallback"
    )
    public PromotionInfo getPromotion(Long userId) {
        return marketingService.getPersonalizedPromotion(userId);
    }

    // 营销降级：返回通用活动
    public PromotionInfo marketingPromotionFallback(Long userId, Throwable ex) {
        log.warn("营销服务降级，返回默认活动");
        return PromotionInfo.getDefaultPromotion();
    }
}
```

---

## 五、实战最佳实践

### 1. **分级熔断配置模板**

```java
@Component
public class CircuitBreakerTemplate {

    /**
     * 根据服务重要性配置不同的熔断策略
     */
    public void configureDegradeRules() {
        // P0 核心服务：严格熔断
        addCoreServiceRule("paymentService", 0.05, 500, 60);
        addCoreServiceRule("orderService", 0.05, 300, 60);

        // P1 重要服务：适中熔断
        addImportantServiceRule("inventoryService", 0.2, 1000, 30);
        addImportantServiceRule("userService", 0.2, 1000, 30);

        // P2 非核心服务：宽松熔断
        addNormalServiceRule("recommendService", 0.5, 2000, 10);
        addNormalServiceRule("reviewService", 0.5, 2000, 10);
    }

    private void addCoreServiceRule(String resource, double exceptionRatio,
                                    int rt, int timeWindow) {
        DegradeRule rule = new DegradeRule(resource)
            .setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO)
            .setCount(exceptionRatio)
            .setTimeWindow(timeWindow)
            .setMinRequestAmount(50)    // 高流量阈值
            .setStatIntervalMs(1000);

        DegradeRuleManager.loadRules(Collections.singletonList(rule));
    }
}
```

### 2. **监控与告警**

```java
@Component
public class SentinelMonitor {

    @Scheduled(fixedRate = 10000)
    public void monitorCircuitBreaker() {
        Map<String, DegradeRule> rules = DegradeRuleManager.getRules()
            .stream()
            .collect(Collectors.toMap(DegradeRule::getResource, r -> r));

        for (Map.Entry<String, DegradeRule> entry : rules.entrySet()) {
            String resource = entry.getKey();

            // 检查熔断状态
            if (isCircuitBreakerOpen(resource)) {
                sendAlert(resource, "熔断器开启");
            }

            // 统计降级次数
            long degradeCount = getDegradeCount(resource);
            if (degradeCount > 100) {
                sendAlert(resource, "降级次数过高: " + degradeCount);
            }
        }
    }

    private void sendAlert(String resource, String message) {
        log.error("【告警】资源: {}, 消息: {}", resource, message);
        // 发送到告警系统
        alertService.send(resource, message);
    }
}
```

### 3. **灰度降级策略**

```java
@Service
public class GrayDegradeService {

    /**
     * 根据用户等级实施差异化降级
     */
    @SentinelResource(value = "vipService")
    public ServiceResult processRequest(UserContext user) {
        // VIP 用户：完整服务
        if (user.isVip()) {
            return fullService.process(user);
        }

        // 普通用户：在高峰期降级
        if (isHighTrafficPeriod()) {
            return degradedService.process(user);
        }

        return fullService.process(user);
    }

    /**
     * 基于流量百分比的降级
     */
    public ServiceResult processWithTrafficShaping(Request request) {
        int randomValue = ThreadLocalRandom.current().nextInt(100);

        // 10% 流量降级
        if (randomValue < 10) {
            return degradedService.handle(request);
        }

        return normalService.handle(request);
    }
}
```

---

## 六、关键配置建议

| 服务类型 | 异常比例 | RT 阈值 | 熔断时长 | 最小请求数 |
| -------- | -------- | ------- | -------- | ---------- |
| 核心服务 | 5%       | 300ms   | 60s      | 50         |
| 重要服务 | 20%      | 1000ms  | 30s      | 20         |
| 一般服务 | 50%      | 2000ms  | 10s      | 10         |

## 七、注意事项

1. **避免雪崩**：熔断时长不宜过短，给下游服务恢复时间
2. **监控覆盖**：所有熔断降级都要有监控和告警
3. **测试验证**：压测验证熔断阈值是否合理
4. **文档化**：明确记录每个服务的降级策略和预期行为
5. **动态调整**：通过配置中心实现规则热更新

通过合理配置 Sentinel 的熔断降级机制，可以有效保障微服务架构的稳定性和核心链路的可用性！
