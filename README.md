# Kafka 遷移指南與 @KafkaListener 使用說明

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Kafka](https://img.shields.io/badge/Kafka-3.9.1-orange.svg)](https://kafka.apache.org/)
[![Spring](https://img.shields.io/badge/Spring-Kafka-green.svg)](https://spring.io/projects/spring-kafka)

> 從舊的 Kafka（Spring Boot 自動配置）遷移到新的 Kafka（手動配置 + SASL 認證）的完整指南

## 📋 目錄

- [背景說明](#背景說明)
- [快速開始](#快速開始)
- [遷移方式](#遷移方式)
- [程式碼範例對比](#程式碼範例對比)
- [@KafkaListener 使用方式](#kafkalistener-使用方式)
- [兩種模式比較](#兩種模式比較)
- [遷移檢查清單](#遷移檢查清單)
- [常見問題](#常見問題)
- [參考資料](#參考資料)

---

## 🎯 背景說明

### 現況

- **舊的 Kafka**：使用 Spring Boot 自動配置，無 SASL 認證
- **新的 Kafka**：手動配置 Bean，支援 SASL_PLAINTEXT 認證
- **配置已對齊**：新的 Kafka 配置與舊的 Kafka 預設值一致

### 遷移目標

- 將現有服務從舊的 Kafka 遷移到新的 Kafka
- 提供兩種使用方式：**手動模式**（現有方式）和 **@KafkaListener 模式**（新方式）

### 配置對比

| 配置項目 | 舊的 Kafka (預設) | 新的 Kafka (已對齊) |
|---------|------------------|-------------------|
| ACKS | `"1"` | `"1"` ✅ |
| RETRIES | `2147483647` | `2147483647` ✅ |
| enable.auto.commit | `true` | `true` ✅ |
| 認證機制 | 無 | SASL_PLAINTEXT |
| 連接超時配置 | 預設值 | 明確設定 ✅ |
| 重連退避策略 | 預設值 | 明確設定 ✅ |

---

## 🚀 快速開始

### 最小變更遷移（推薦）

只需要修改注入的 `ConsumerFactory`：

// 修改前
@Autowired
private ConsumerFactory<String, String> consumerFactory;

// 修改後
@Autowired
@Qualifier("newKafkaConsumerFactory")
private ConsumerFactory<String, String> consumerFactory;**就是這麼簡單！** 其他程式碼完全不需要修改。

---

## 📝 遷移方式

### 方式一：最小變更遷移（推薦）

僅需修改注入的 `ConsumerFactory`，其他程式碼完全不變。

#### 修改步驟

**修改前：**
@Service
public class DpaEventLogService {
    
    @Autowired
    private ConsumerFactory<String, String> consumerFactory;  // 使用舊的 Kafka
    
    @Override
    public void saveDpaEventLogsFromKafka() {
        Consumer<String, String> consumer = consumerFactory.createConsumer();
        consumer.subscribe(Collections.singletonList(TOPIC));
        // ... 其他程式碼不變
    }
}**修改後：**
@Service
public class DpaEventLogService {
    
    @Autowired
    @Qualifier("newKafkaConsumerFactory")  // 指定使用新的 Kafka ConsumerFactory
    private ConsumerFactory<String, String> consumerFactory;
    
    @Override
    public void saveDpaEventLogsFromKafka() {
        Consumer<String, String> consumer = consumerFactory.createConsumer();
        consumer.subscribe(Collections.singletonList(TOPIC));
        // ... 其他程式碼完全不變
    }
}#### 優點

- ✅ **變更最小**：只需改一行程式碼
- ✅ **風險最低**：現有邏輯完全不變
- ✅ **測試簡單**：只需測試連接和基本功能
- ✅ **可逐步遷移**：可以一個服務一個服務遷移

---

## 💻 程式碼範例對比

### 完整範例：手動模式 vs @KafkaListener 模式

#### 範例 1：批次處理訊息

**手動模式（現有方式）：**
@Service
@Transactional
public class DpaEventLogService {
    
    private static final String TOPIC = "dpaeventlog";
    private static final int MAX_POLLS = 60;
    private static final int BATCH_SIZE = 500;
    
    @Autowired
    @Qualifier("newKafkaConsumerFactory")
    private ConsumerFactory<String, String> consumerFactory;
    
    @Autowired
    private IDpaEventLogDao dpaEventLogDao;
    
    public void saveDpaEventLogsFromKafka() {
        logger.info("開始處理 Kafka 訊息，Topic: {}", TOPIC);
        
        // 1. 手動創建 Consumer
        Consumer<String, String> consumer = consumerFactory.createConsumer();
        consumer.subscribe(Collections.singletonList(TOPIC));
        
        int emptyPollCount = 0;
        int processCount = 0;
        
        try {
            // 2. 手動 poll 訊息（需要寫迴圈）
            for (int i = 0; i < MAX_POLLS; i++) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000L));
                
                if (records.isEmpty()) {
                    emptyPollCount++;
                    if (emptyPollCount >= MAX_POLLS) {
                        logger.info("無更多資料，處理完成");
                        break;
                    }
                    continue;
                }
                
                emptyPollCount = 0;
                List<DpaEventLog> batch = new ArrayList<>();
                
                // 3. 手動處理每筆訊息（需要寫迴圈）
                for (ConsumerRecord<String, String> record : records) {
                    DpaEventLog dpaEventLog = processMessage(record);
                    if (dpaEventLog != null) {
                        batch.add(dpaEventLog);
                    }
                    
                    // 4. 手動批次處理
                    if (batch.size() >= BATCH_SIZE) {
                        processCount = processBatch(batch, processCount);
                        batch.clear();
                    }
                }
                
                // 處理最後不足批次大小的資料
                if (!batch.isEmpty()) {
                    processCount = processBatch(batch, processCount);
                }
            }
        } finally {
            // 5. 手動關閉 Consumer
            try {
                consumer.close();
                logger.info("Consumer 已關閉，總處理筆數: {}", processCount);
            } catch (Exception e) {
                logger.error("Consumer 關閉時發生錯誤", e);
            }
        }
    }
    
    private DpaEventLog processMessage(ConsumerRecord<String, String> record) {
        try {
            String message = record.value();
            logger.info("處理訊息: {}, 分區: {}, Offset: {}", 
                message, record.partition(), record.offset());
            
            JsonNode jsonNode = new ObjectMapper().readTree(message);
            String rawMessage = jsonNode.get("message").asText();
            
            if ("##".equals(rawMessage) || StringUtils.isBlank(rawMessage)) {
                return null;
            }
            
            return new DpaEventLog(rawMessage);
        } catch (Exception e) {
            logger.error("處理訊息發生錯誤: {}", record.value(), e);
            return null;
        }
    }
    
    private int processBatch(List<DpaEventLog> batch, int currentProcessCount) {
        int newProcessCount = dpaEventLogDao.batchInsertDpaEventLogs(batch) + currentProcessCount;
        logger.info("批次處理完成，總筆數: {}", newProcessCount);
        return newProcessCount;
    }
}**@KafkaListener 模式（新方式）：**
@Service
@Transactional
public class DpaEventLogService {
    
    private static final String TOPIC = "dpaeventlog";
    private static final int BATCH_SIZE = 500;
    
    @Autowired
    private IDpaEventLogDao dpaEventLogDao;
    
    private List<DpaEventLog> batch = new ArrayList<>();
    
    /**
     * 使用 @KafkaListener 自動處理訊息
     * Spring 會自動：
     * 1. 創建和管理 Consumer
     * 2. 自動 poll 訊息
     * 3. 自動呼叫此方法處理每筆訊息
     * 4. 自動提交 offset（enable.auto.commit=true）
     * 5. 自動處理錯誤和重試
     */
    @KafkaListener(
        topics = "dpaeventlog",
        containerFactory = "newKafkaListenerContainerFactory"
    )
    public void listen(ConsumerRecord<String, String> record) {
        try {
            // 只需要處理業務邏輯，其他都由 Spring 自動處理
            DpaEventLog dpaEventLog = processMessage(record);
            
            if (dpaEventLog != null) {
                batch.add(dpaEventLog);
                
                // 批次處理
                if (batch.size() >= BATCH_SIZE) {
                    processBatch(batch);
                    batch.clear();
                }
            }
        } catch (Exception e) {
            logger.error("處理訊息發生錯誤: {}", record.value(), e);
            // Spring 會自動處理錯誤和重試
        }
    }
    
    // 處理最後的批次（可以使用 @PreDestroy 或定時任務）
    @PreDestroy
    public void flushBatch() {
        if (!batch.isEmpty()) {
            processBatch(batch);
            batch.clear();
        }
    }
    
    private DpaEventLog processMessage(ConsumerRecord<String, String> record) {
        try {
            String message = record.value();
            logger.info("處理訊息: {}, 分區: {}, Offset: {}", 
                message, record.partition(), record.offset());
            
            JsonNode jsonNode = new ObjectMapper().readTree(message);
            String rawMessage = jsonNode.get("message").asText();
            
            if ("##".equals(rawMessage) || StringUtils.isBlank(rawMessage)) {
                return null;
            }
            
            return new DpaEventLog(rawMessage);
        } catch (Exception e) {
            logger.error("處理訊息發生錯誤: {}", record.value(), e);
            return null;
        }
    }
    
    private void processBatch(List<DpaEventLog> batch) {
        dpaEventLogDao.batchInsertDpaEventLogs(batch);
        logger.info("批次處理完成，筆數: {}", batch.size());
    }
}#### 範例 2：簡單訊息處理

**手動模式：**ava
@RestController
public class HelloController {
    
    @Autowired
    @Qualifier("newKafkaConsumerFactory")
    private ConsumerFactory<String, String> consumerFactory;
    
    @GetMapping("/api/kafka/test-consume")
    public ResponseEntity<Map<String, Object>> testConsume(
            @RequestParam String topic,
            @RequestParam int maxRecords) {
        
        Map<String, Object> result = new HashMap<>();
        List<Map<String, Object>> messages = new ArrayList<>();
        
        // 1. 手動創建 Consumer
        Consumer<String, String> consumer = consumerFactory.createConsumer();
        
        try {
            // 2. 手動訂閱
            consumer.subscribe(Collections.singletonList(topic));
            
            int pollCount = 0;
            int totalRecords = 0;
            
            // 3. 手動 poll
            while (totalRecords < maxRecords && pollCount < 5) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(2));
                
                if (records.isEmpty()) {
                    pollCount++;
                    continue;
                }
                
                // 4. 手動處理
                for (ConsumerRecord<String, String> record : records) {
                    if (totalRecords >= maxRecords) {
                        break;
                    }
                    
                    Map<String, Object> msg = new HashMap<>();
                    msg.put("key", record.key());
                    msg.put("value", record.value());
                    msg.put("partition", record.partition());
                    msg.put("offset", record.offset());
                    messages.add(msg);
                    totalRecords++;
                }
            }
            
            result.put("status", "success");
            result.put("messages", messages);
            return ResponseEntity.ok(result);
            
        } finally {
            // 5. 手動關閉
            consumer.close();
        }
    }
}**@KafkaListener 模式：**
@RestController
public class HelloController {
    
    private final List<Map<String, Object>> messages = new CopyOnWriteArrayList<>();
    
    /**
     * 使用 @KafkaListener 自動處理訊息
     */
    @KafkaListener(
        topics = "#{@topicResolver.resolve()}",
        containerFactory = "newKafkaListenerContainerFactory"
    )
    public void listen(ConsumerRecord<String, String> record) {
        // Spring 自動處理所有事情，只需要處理業務邏輯
        Map<String, Object> msg = new HashMap<>();
        msg.put("key", record.key());
        msg.put("value", record.value());
        msg.put("partition", record.partition());
        msg.put("offset", record.offset());
        messages.add(msg);
    }
    
    @GetMapping("/api/kafka/test-consume")
    public ResponseEntity<Map<String, Object>> testConsume() {
        Map<String, Object> result = new HashMap<>();
        result.put("status", "success");
        result.put("messages", new ArrayList<>(messages));
        messages.clear(); // 清空已讀取的訊息
        return ResponseEntity.ok(result);
    }
}---

## 🔧 @KafkaListener 使用方式

### 基本使用

@KafkaListener(
    topics = "my-topic",
    containerFactory = "newKafkaListenerContainerFactory"
)
public void listen(String message) {
    // 處理訊息
    logger.info("收到訊息: {}", message);
}### 進階使用

#### 1. 接收 ConsumerRecord（取得完整資訊）

@KafkaListener(
    topics = "my-topic",
    containerFactory = "newKafkaListenerContainerFactory"
)
public void listen(ConsumerRecord<String, String> record) {
    logger.info("Key: {}, Value: {}, Partition: {}, Offset: {}", 
        record.key(), 
        record.value(), 
        record.partition(), 
        record.offset());
}#### 2. 批次處理

@KafkaListener(
    topics = "my-topic",
    containerFactory = "newKafkaListenerContainerFactory"
)
public void listen(List<ConsumerRecord<String, String>> records) {
    logger.info("收到 {} 筆訊息", records.size());
    for (ConsumerRecord<String, String> record : records) {
        // 處理每筆訊息
    }
}#### 3. 多個 Topic

@KafkaListener(
    topics = {"topic1", "topic2", "topic3"},
    containerFactory = "newKafkaListenerContainerFactory"
)
public void listen(ConsumerRecord<String, String> record) {
    logger.info("Topic: {}, Value: {}", record.topic(), record.value());
}#### 4. 使用 Topic Pattern

@KafkaListener(
    topicPattern = "my-topic-.*",
    containerFactory = "newKafkaListenerContainerFactory"
)
public void listen(ConsumerRecord<String, String> record) {
    // 會監聽所有符合 pattern 的 topic
}
#### 5. 指定 Consumer Group

@KafkaListener(
    topics = "my-topic",
    containerFactory = "newKafkaListenerContainerFactory",
    groupId = "my-custom-group"
)
public void listen(String message) {
    // 使用自訂的 group ID
}#### 6. 錯誤處理

@KafkaListener(
    topics = "my-topic",
    containerFactory = "newKafkaListenerContainerFactory"
)
public void listen(String message) {
    try {
        // 處理訊息
        processMessage(message);
    } catch (Exception e) {
        logger.error("處理訊息失敗: {}", message, e);
        // Spring 會自動重試（根據配置）
        throw e; // 拋出異常會觸發重試
    }
}---

## 📊 兩種模式比較

### 對比表

| 項目 | 手動模式 | @KafkaListener 模式 |
|------|---------|-------------------|
| **程式碼複雜度** | 較複雜（需要寫迴圈、管理生命週期） | 較簡單（只需一個方法） |
| **控制度** | 完全控制（poll 次數、批次大小等） | 較少控制（由 Spring 管理） |
| **Consumer 生命週期** | 手動管理（創建、關閉） | Spring 自動管理 |
| **Poll 訊息** | 手動 `consumer.poll()` | Spring 自動 poll |
| **處理訊息** | 手動迴圈處理 | 自動呼叫方法 |
| **錯誤處理** | 手動 try-catch | Spring 自動處理和重試 |
| **並發處理** | 手動控制 | Spring 自動控制（可配置） |
| **Offset 提交** | 自動提交（enable.auto.commit=true） | 自動提交（可配置） |
| **適用場景** | 複雜業務邏輯、需要精確控制 | 簡單業務邏輯、標準處理流程 |
| **遷移成本** | 低（只需改注入） | 中（需要重構程式碼） |

### 優缺點分析

#### 手動模式

**優點：**
- ✅ 完全控制處理流程
- ✅ 可自訂 poll 次數、批次大小、空轉處理
- ✅ 適合複雜業務邏輯
- ✅ 遷移成本低（只需改注入）

**缺點：**
- ❌ 程式碼較多
- ❌ 需要手動管理生命週期
- ❌ 錯誤處理需要自己寫

**適用場景：**
- 需要批次處理大量訊息
- 需要控制 poll 次數和空轉處理
- 需要複雜的業務邏輯控制
- 現有程式碼已運作良好

#### @KafkaListener 模式

**優點：**
- ✅ 程式碼簡潔
- ✅ Spring 自動管理生命週期
- ✅ 自動錯誤處理和重試
- ✅ 支援並發處理
- ✅ 符合 Spring 最佳實踐

**缺點：**
- ❌ 控制度較少
- ❌ 不適合需要複雜控制流程的場景
- ❌ 需要重構現有程式碼

**適用場景：**
- 簡單的訊息處理邏輯
- 標準的處理流程
- 新開發的功能
- 不需要複雜控制的場景

---

## ✅ 遷移檢查清單

### 遷移前準備

- [ ] 確認新的 Kafka 配置正確
- [ ] 確認新的 Kafka 可以正常連接
- [ ] 確認新的 Kafka 可以正常發送和接收訊息
- [ ] 準備回滾方案

### 遷移步驟（手動模式）

- [ ] 修改注入的 `ConsumerFactory`，加上 `@Qualifier("newKafkaConsumerFactory")`
- [ ] 確認程式碼可以編譯
- [ ] 在測試環境測試
- [ ] 確認可以正常 poll 訊息
- [ ] 確認業務邏輯正常運作
- [ ] 確認 offset 正常提交
- [ ] 在生產環境部署
- [ ] 監控運行狀況

### 遷移步驟（@KafkaListener 模式）

- [ ] 重構程式碼，改用 `@KafkaListener`
- [ ] 確認程式碼可以編譯
- [ ] 在測試環境測試
- [ ] 確認訊息可以正常接收
- [ ] 確認業務邏輯正常運作
- [ ] 確認錯誤處理正常
- [ ] 在生產環境部署
- [ ] 監控運行狀況

---

## ❓ 常見問題

### Q1: 遷移後 offset 會重置嗎？

**A:** 不會。Offset 是儲存在 Kafka 的 `__consumer_offsets` topic 中，與 Consumer Group ID 相關。只要 Group ID 相同，offset 就會保持。

### Q2: 手動模式和 @KafkaListener 模式可以同時使用嗎？

**A:** 可以。兩種模式可以共存，使用不同的 `ConsumerFactory` 即可。

### Q3: @KafkaListener 如何控制批次大小？

**A:** 可以透過 `max.poll.records` 配置控制每次 poll 的訊息數量：

// 在 NewKafkaConfig 中
configProps.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "100");### Q4: @KafkaListener 如何處理錯誤和重試？

**A:** Spring Kafka 會自動處理錯誤和重試。可以透過 `ContainerProperties` 配置重試策略：

factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);
factory.setCommonErrorHandler(new DefaultErrorHandler());### Q5: 如何從手動模式遷移到 @KafkaListener 模式？

**A:** 建議步驟：
1. 先使用手動模式完成遷移（最小變更）
2. 確認運作正常後
3. 再逐步重構為 @KafkaListener 模式

### Q6: 新的 Kafka 配置與舊的有什麼差異？

**A:** 主要差異：
- 新的 Kafka 支援 SASL 認證
- 配置已對齊，行為與舊的 Kafka 一致
- 其他配置（ACKS、RETRIES、超時等）都已對齊

### Q7: 舊的程式手動處理訊息是指處理 offset 嗎？

**A:** 不是。舊的程式沒有手動處理 offset，而是依賴自動提交（`enable.auto.commit=true`）。「手動處理訊息」指的是：
- 手動 poll 訊息：`consumer.poll()`
- 手動解析訊息：`record.value()`
- 手動處理業務邏輯：`processMessage(record)`
- 手動存入資料庫：`batchInsertDpaEventLogs(batch)`

### Q8: 如果沒有特別設定，新舊 Kafka 會有相同的重連行為嗎？

**A:** 是的。如果沒有特別設定，兩者會有相同的重連行為：
- 都使用 Kafka 客戶端的預設值
- 連接失敗時都會持續重連
- 重連頻率相同（預設退避策略）

---

## 📚 參考資料

- [Spring Kafka 官方文件](https://docs.spring.io/spring-kafka/reference/html/)
- [Kafka 官方文件](https://kafka.apache.org/documentation/)
- [Spring Boot Kafka 自動配置](https://docs.spring.io/spring-boot/docs/current/reference/html/messaging.html#messaging.kafka)

---

## 📄 License

MIT License

---

## 👥 貢獻

歡迎提交 Issue 和 Pull Request！
