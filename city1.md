## ✅ 一、Controller：触发处理任务

```
java复制编辑@RestController
@RequestMapping("/user-records")
public class UserRecordController {

    @Autowired
    private UserRecordProcessor userRecordProcessor;

    @PostMapping("/process/{userId}")
    public ResponseEntity<String> triggerProcessing(@PathVariable Long userId) {
        userRecordProcessor.processAllRecordsForUser(userId);
        return ResponseEntity.ok("Processing started for userId: " + userId);
    }
}
```

------

## ✅ 二、异步处理服务：处理每一批数据

```
java复制编辑@Service
public class BatchProcessingService {

    @Autowired
    private RetryRecordRepository retryRepo;

    @Async
    public void processBatchAsync(RetryRecord retryRecord, List<UserRecord> batch) {
        try {
            for (UserRecord record : batch) {
                // 模拟调用外部微服务接口
                // externalService.call(record.getNetworkId(), record.getCity());
            }

            retryRecord.setStatus("SUCCESS");
        } catch (Exception e) {
            retryRecord.setStatus("FAILED");
            retryRecord.setRetryCount(retryRecord.getRetryCount() + 1);
            retryRecord.setErrorMessage(e.getMessage());
        }

        retryRecord.setLastUpdated(LocalDateTime.now());
        retryRepo.save(retryRecord);
    }
}
```

------

## ✅ 三、主处理逻辑：分页读取 + 写入重试记录 + 异步处理

```
java复制编辑@Service
public class UserRecordProcessor {

    private static final int BATCH_SIZE = 1000;

    @Autowired
    private UserRecordRepository userRepo;

    @Autowired
    private RetryRecordRepository retryRepo;

    @Autowired
    private BatchProcessingService batchService;

    public void processAllRecordsForUser(Long userId) {
        int page = 0;

        while (true) {
            Pageable pageable = PageRequest.of(page, BATCH_SIZE);
            List<UserRecord> batch = userRepo.findByUserId(userId, pageable);
            if (batch.isEmpty()) break;

            RetryRecord retry = new RetryRecord();
            retry.setUserId(userId);
            retry.setStatus("PROCESSING");
            retry.setRetryCount(0);
            retry.setLastUpdated(LocalDateTime.now());
            retry.setSource("AUTO");

            RetryRecord saved = retryRepo.save(retry);

            batchService.processBatchAsync(saved, batch);

            page++;
        }
    }
}
```

------

## ✅ 四、定时重试任务：定时扫描失败和超时记录并重试

```
java复制编辑@Component
public class RetryScheduler {

    @Autowired
    private RetryRecordRepository retryRepo;

    @Autowired
    private UserRecordRepository userRepo;

    @Autowired
    private BatchProcessingService batchService;

    @Scheduled(fixedDelay = 5 * 60 * 1000)
    public void retryFailedOrStuckBatches() {
        LocalDateTime threshold = LocalDateTime.now().minusMinutes(5);

        List<RetryRecord> toRetry = retryRepo.findByStatus("FAILED");
        toRetry.addAll(retryRepo.findByStatusAndLastUpdatedBefore("PROCESSING", threshold));

        for (RetryRecord retryRecord : toRetry) {
            Pageable pageable = PageRequest.of(0, BATCH_SIZE);
            List<UserRecord> batch = userRepo.findByUserId(retryRecord.getUserId(), pageable);

            if (!batch.isEmpty()) {
                retryRecord.setStatus("PROCESSING");
                retryRecord.setLastUpdated(LocalDateTime.now());
                retryRepo.save(retryRecord);

                batchService.processBatchAsync(retryRecord, batch);
            }
        }
    }
}
```

------

## ✅ 五、实体类：RetryRecord

```
java复制编辑@Entity
public class RetryRecord {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long userId;

    private String status; // PROCESSING, FAILED, SUCCESS

    private Integer retryCount = 0;

    private String errorMessage;

    private LocalDateTime lastUpdated;

    private String source;

    // Getters & Setters
}
```