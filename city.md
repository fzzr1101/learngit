## 🧩 1. 实体类：`UserRecord.java`

```
java复制编辑@Entity
@Table(name = "user_record")
public class UserRecord {
    @Id
    private Long id;

    private Long userId;
    private String networkId;
    private String city;

    // getters/setters
}
```

------

## 🧩 2. 实体类：`ProcessingBatchStatus.java`

```
java复制编辑@Entity
@Table(name = "processing_batch_status", uniqueConstraints = {
    @UniqueConstraint(columnNames = {"userId", "batchOffset"})
})
public class ProcessingBatchStatus {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long userId;
    private Integer batchOffset;
    private Integer batchSize;
    private Long startId;
    private Long endId;

    private String status; // PENDING, SUCCESS, FAILED
    private Integer retryCount = 0;
    private LocalDateTime lastAttemptAt;

    @Column(columnDefinition = "TEXT")
    private String errorMessage;

    // getters/setters
}
```

------

## 🧩 3. Repository 接口

### `UserRecordRepository.java`

```
java复制编辑public interface UserRecordRepository extends JpaRepository<UserRecord, Long> {

    @Query("SELECT u FROM UserRecord u WHERE u.userId = :userId AND u.id > :lastId ORDER BY u.id ASC")
    List<UserRecord> findNextBatch(@Param("userId") Long userId,
                                   @Param("lastId") Long lastId,
                                   Pageable pageable);

    @Query("SELECT u FROM UserRecord u WHERE u.userId = :userId AND u.id BETWEEN :startId AND :endId ORDER BY u.id ASC")
    List<UserRecord> findByUserIdAndIdRange(@Param("userId") Long userId,
                                            @Param("startId") Long startId,
                                            @Param("endId") Long endId);
}
```

------

### `ProcessingBatchStatusRepository.java`

```
java复制编辑public interface ProcessingBatchStatusRepository extends JpaRepository<ProcessingBatchStatus, Long> {
    Optional<ProcessingBatchStatus> findByUserIdAndBatchOffset(Long userId, Integer batchOffset);
    List<ProcessingBatchStatus> findByStatus(String status);
}
```

------

## 🧩 4. 异步配置类

```
java复制编辑@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("batchExecutor")
    public Executor batchExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("BatchProcessor-");
        executor.initialize();
        return executor;
    }
}
```

------

## 🧩 5. Service：`BatchProcessingService.java`

```
java复制编辑@Service
public class BatchProcessingService {

    @Autowired
    private ProcessingBatchStatusRepository batchStatusRepository;

    @Autowired
    private ExternalApiService externalApiService;

    @Async("batchExecutor")
    public void processBatchAsync(Long userId, int offset, List<UserRecord> batch, long startId, long endId) {
        ProcessingBatchStatus batchStatus = batchStatusRepository
            .findByUserIdAndBatchOffset(userId, offset)
            .orElseThrow(() -> new RuntimeException("Batch record not found"));

        boolean hasFailure = false;
        StringBuilder errors = new StringBuilder();

        for (UserRecord record : batch) {
            try {
                externalApiService.send(record.getNetworkId(), record.getCity());
            } catch (Exception e) {
                hasFailure = true;
                errors.append("id=").append(record.getId()).append(": ").append(e.getMessage()).append("\n");
            }
        }

        batchStatus.setStatus(hasFailure ? "FAILED" : "SUCCESS");
        batchStatus.setRetryCount(batchStatus.getRetryCount() + (hasFailure ? 1 : 0));
        batchStatus.setErrorMessage(hasFailure ? errors.toString() : null);
        batchStatus.setLastAttemptAt(LocalDateTime.now());
        batchStatus.setStartId(startId);
        batchStatus.setEndId(endId);
        batchStatusRepository.save(batchStatus);
    }
}
```

------

## 🧩 6. 主调度器：`UserRecordProcessor.java`

```
java复制编辑@Service
public class UserRecordProcessor {

    @Autowired
    private UserRecordRepository userRecordRepository;

    @Autowired
    private ProcessingBatchStatusRepository batchStatusRepository;

    @Autowired
    private BatchProcessingService batchProcessingService;

    public void processUser(Long userId) {
        int batchSize = 1000;
        int offset = 0;
        Long lastId = 0L;

        while (true) {
            List<UserRecord> batch = userRecordRepository.findNextBatch(userId, lastId, PageRequest.of(0, batchSize));
            if (batch.isEmpty()) break;

            Long startId = batch.get(0).getId();
            Long endId = batch.get(batch.size() - 1).getId();

            Optional<ProcessingBatchStatus> existing = batchStatusRepository.findByUserIdAndBatchOffset(userId, offset);
            if (existing.isPresent() && "SUCCESS".equals(existing.get().getStatus())) {
                lastId = endId;
                offset++;
                continue;
            }

            ProcessingBatchStatus status = existing.orElse(new ProcessingBatchStatus());
            status.setUserId(userId);
            status.setBatchOffset(offset);
            status.setBatchSize(batch.size());
            status.setStartId(startId);
            status.setEndId(endId);
            status.setStatus("PENDING");
            status.setLastAttemptAt(LocalDateTime.now());
            batchStatusRepository.save(status);

            batchProcessingService.processBatchAsync(userId, offset, batch, startId, endId);

            lastId = endId;
            offset++;
        }
    }
}
```

------

## 🧩 7. 重试失败批次（定时任务）

```
java复制编辑@Component
public class FailedBatchRetryJob {

    @Autowired
    private ProcessingBatchStatusRepository batchStatusRepository;

    @Autowired
    private UserRecordRepository userRecordRepository;

    @Autowired
    private BatchProcessingService batchProcessingService;

    @Scheduled(fixedDelay = 5 * 60 * 1000)
    public void retryFailedBatches() {
        List<ProcessingBatchStatus> failedBatches = batchStatusRepository.findByStatus("FAILED");

        for (ProcessingBatchStatus batch : failedBatches) {
            List<UserRecord> records = userRecordRepository.findByUserIdAndIdRange(
                batch.getUserId(), batch.getStartId(), batch.getEndId());

            if (!records.isEmpty()) {
                batchProcessingService.processBatchAsync(
                    batch.getUserId(), batch.getBatchOffset(), records, batch.getStartId(), batch.getEndId());
            }
        }
    }
}
```