## ✅ Java 示例代码

```
java复制编辑public class GeocodeTask implements Runnable {

    private final String networkId;
    private final List<String> addressList;
    private final DualRateLimiter rateLimiter;

    private final List<ResultRecord> resultList = new ArrayList<>();

    public GeocodeTask(String networkId, List<String> addressList, DualRateLimiter rateLimiter) {
        this.networkId = networkId;
        this.addressList = addressList;
        this.rateLimiter = rateLimiter;
    }

    @Override
    public void run() {
        for (String address : addressList) {
            GeocodeResult result = tryGeocodeWithFallback(address);
            result.setNetworkId(networkId);
            resultList.add(result);
        }

        // 可选：结果处理，如写库、打印、上传
        resultList.forEach(System.out::println);
    }

    private GeocodeResult tryGeocodeWithFallback(String address) {
        // === 1. 尝试 API1 ===
        for (int i = 0; i < 3; i++) {
            RateLimitResult qpsResult = rateLimiter.checkQpsLimit("api1", 15);
            if (!qpsResult.isAllowed()) {
                sleep(200); // 短暂等待再试
                continue;
            }

            RateLimitResult dailyResult = rateLimiter.checkDailyLimit("api1", 1000);
            if (!dailyResult.isAllowed()) {
                // 达到每日限制，尝试 API2
                return tryApi2(address, "DAILY_LIMIT_EXCEEDED");
            }

            try {
                GeocodeApiResponse api1Response = callApi1(address);
                if (api1Response != null && api1Response.hasValidCoordinates()) {
                    return GeocodeResult.success(address, api1Response.getCity(),
                            api1Response.getLat(), api1Response.getLng());
                } else {
                    return GeocodeResult.failure(address, "NO_RESULT");
                }
            } catch (Exception e) {
                rateLimiter.rollbackDailyLimit("api1");
                return GeocodeResult.failure(address, "CALL_FAILED_API1");
            }
        }

        // 如果 QPS 连续失败三次，认为 API1 限流，转用 API2
        return tryApi2(address, "QPS_LIMIT_EXCEEDED");
    }

    private GeocodeResult tryApi2(String address, String reasonFromApi1) {
        RateLimitResult qpsResult = rateLimiter.checkQpsLimit("api2", 15);
        if (!qpsResult.isAllowed()) {
            return GeocodeResult.failure(address, "QPS_LIMIT_EXCEEDED_API2");
        }

        RateLimitResult dailyResult = rateLimiter.checkDailyLimit("api2", 1000);
        if (!dailyResult.isAllowed()) {
            return GeocodeResult.failure(address, "DAILY_LIMIT_EXCEEDED_API2");
        }

        try {
            GeocodeApiResponse api2Response = callApi2(address);
            if (api2Response != null && api2Response.hasValidCoordinates()) {
                return GeocodeResult.success(address, api2Response.getCity(),
                        api2Response.getLat(), api2Response.getLng());
            } else {
                return GeocodeResult.failure(address, "NO_RESULT_API2");
            }
        } catch (Exception e) {
            rateLimiter.rollbackDailyLimit("api2");
            return GeocodeResult.failure(address, "CALL_FAILED_API2");
        }
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException ignored) {
        }
    }

    // === Stub for external API call ===
    private GeocodeApiResponse callApi1(String address) {
        // TODO: 实际调用API1
        return null;
    }

    private GeocodeApiResponse callApi2(String address) {
        // TODO: 实际调用API2
        return null;
    }
}
```

------

## ✅ ResultRecord / GeocodeResult 定义

```
java复制编辑public class GeocodeResult {
    private String networkId;
    private String address;
    private String city;
    private Double lat;
    private Double lng;
    private String result; // SUCCESS / QPS_LIMIT_EXCEEDED / NO_RESULT / ...

    public static GeocodeResult success(String address, String city, Double lat, Double lng) {
        GeocodeResult r = new GeocodeResult();
        r.address = address;
        r.city = city;
        r.lat = lat;
        r.lng = lng;
        r.result = "SUCCESS";
        return r;
    }

    public static GeocodeResult failure(String address, String reason) {
        GeocodeResult r = new GeocodeResult();
        r.address = address;
        r.result = reason;
        return r;
    }

    public void setNetworkId(String networkId) {
        this.networkId = networkId;
    }

    @Override
    public String toString() {
        return "[networkId=" + networkId + ", address=" + address + ", result=" + result + "]";
    }
}
```

------

## ✅ GeocodeApiResponse 示例（你根据实际情况替换）

```
java复制编辑public class GeocodeApiResponse {
    private String city;
    private Double lat;
    private Double lng;

    public boolean hasValidCoordinates() {
        return lat != null && lng != null;
    }

    // getters/setters omitted
}
```

------

## ✅ 如果你希望把结果返回给调用方

可以再加一个 `getResultList()` 方法：

```
java复制编辑public List<ResultRecord> getResultList() {
    return resultList;
}
```