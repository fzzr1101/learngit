public class RateLimitResult {
    private boolean allowed;
    private String reason; // null if allowed

    // 构造方法和 getter/setter 略
    public static RateLimitResult ok() {
        return new RateLimitResult(true, null);
    }
    
    public static RateLimitResult fail(String reason) {
        return new RateLimitResult(false, reason);
    }
    
    public boolean isAllowed() {
        return allowed;
    }
    
    public String getReason() {
        return reason;
    }
}
java
复制
编辑
@Component
public class DualRateLimiter {

    @Autowired
    private RedissonClient redissonClient;
    
    public RateLimitResult checkDailyLimit(String apiKey, long dailyLimit) {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        String key = "limit:daily:" + apiKey + ":" + date;
    
        RAtomicLong counter = redissonClient.getAtomicLong(key);
        if (!counter.isExists()) {
            long secondsUntilEndOfDay = Duration.between(
                    LocalDateTime.now(),
                    LocalDate.now().plusDays(1).atStartOfDay()
            ).getSeconds();
            counter.expire(Duration.ofSeconds(secondsUntilEndOfDay));
        }
    
        long current = counter.incrementAndGet();
        if (current <= dailyLimit) {
            return RateLimitResult.ok();
        } else {
            return RateLimitResult.fail("DAILY_LIMIT_EXCEEDED");
        }
    }
    
    public RateLimitResult checkQpsLimit(String apiKey, long qpsLimit) {
        String key = "limit:qps:" + apiKey;
        RRateLimiter limiter = redissonClient.getRateLimiter(key);
    
        if (!limiter.isExists()) {
            limiter.trySetRate(RateType.OVERALL, qpsLimit, 1, RateIntervalUnit.SECONDS);
        }
    
        boolean acquired = limiter.tryAcquire();
        if (acquired) {
            return RateLimitResult.ok();
        } else {
            return RateLimitResult.fail("QPS_LIMIT_EXCEEDED");
        }
    }
    
    public void rollbackDailyLimit(String apiKey) {
        String date = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyyMMdd"));
        String key = "limit:daily:" + apiKey + ":" + date;
        RAtomicLong counter = redissonClient.getAtomicLong(key);
        counter.decrementAndGet();
    }
}
✅ 示例调用逻辑（多 API + 失败原因收集）
java
复制
编辑
@Autowired
private DualRateLimiter limiter;

public void handleTwoApiCalls() {
    List<Map<String, String>> failList = new ArrayList<>();

    String[] apis = {"api1", "api2"};
    
    for (String api : apis) {
        // 单独检查 QPS
        RateLimitResult qpsResult = limiter.checkQpsLimit(api, 15);
        if (!qpsResult.isAllowed()) {
            failList.add(Map.of("api", api, "reason", qpsResult.getReason()));
            continue;
        }
    
        // 单独检查每日次数
        RateLimitResult dailyResult = limiter.checkDailyLimit(api, 1000);
        if (!dailyResult.isAllowed()) {
            failList.add(Map.of("api", api, "reason", dailyResult.getReason()));
            continue;
        }
    
        // 限流全部通过，执行调用
        try {
            callThirdPartyApi(api);
        } catch (Exception e) {
            // 调用失败回滚每日计数
            limiter.rollbackDailyLimit(api);
            failList.add(Map.of("api", api, "reason", "CALL_FAILED"));
        }
    }
    
    if (!failList.isEmpty()) {
        log.warn("以下 API 被限流或失败: {}", failList);
        // 或做别的处理
    }
}