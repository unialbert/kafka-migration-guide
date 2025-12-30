# kafka-migration-guide

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

## 🎯 背景說明

### 現況
- **舊的 Kafka**：使用 Spring Boot 自動配置，無 SASL 認證
- **新的 Kafka**：手動配置 Bean，支援 SASL_PLAINTEXT 認證
- **配置已對齊**：新的 Kafka 配置與舊的 Kafka 預設值一致

### 遷移目標
- 將現有服務從舊的 Kafka 遷移到新的 Kafka
- 提供兩種使用方式：**手動模式**（現有方式）和 **@KafkaListener 模式**（新方式）

## 🚀 快速開始

### 最小變更遷移（推薦）

只需要修改注入的 `ConsumerFactory`：

// 修改前
@Autowired
private ConsumerFactory<String, String> consumerFactory;

// 修改後
@Autowired
@Qualifier("newKafkaConsumerFactory")
private ConsumerFactory<String, String> consumerFactory;就是這麼簡單！其他程式碼完全不需要修改。

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
        // 處理訊息邏輯
    }
    
    private int processBatch(List<DpaEventLog> batch, int currentProcessCount) {
        // 批次處理邏輯
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
        // 處理訊息邏輯
    }
    
    private void processBatch(List<DpaEventLog> batch) {
        // 批次處理邏輯
    }
}## 🔧 @KafkaListener 使用方式

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
}## 📊 兩種模式比較

| 項目 | 手動模式 | @KafkaListener 模式 |
|------|---------|-------------------|
| **程式碼複雜度** | 較複雜 | 較簡單 |
| **控制度** | 完全控制 | 較少控制 |
| **Consumer 生命週期** | 手動管理 | Spring 自動管理 |
| **適用場景** | 複雜業務邏輯 | 簡單業務邏輯 |

詳細比較請參考[完整文件](#兩種模式比較)。

## ✅ 遷移檢查清單

### 遷移前準備
- [ ] 確認新的 Kafka 配置正確
- [ ] 確認新的 Kafka 可以正常連接
- [ ] 準備回滾方案

### 遷移步驟（手動模式）
- [ ] 修改注入的 `ConsumerFactory`
- [ ] 在測試環境測試
- [ ] 確認業務邏輯正常運作
- [ ] 在生產環境部署

## ❓ 常見問題

### Q1: 遷移後 offset 會重置嗎？
A: 不會。Offset 是儲存在 Kafka 的 `__consumer_offsets` topic 中，與 Consumer Group ID 相關。

### Q2: 手動模式和 @KafkaListener 模式可以同時使用嗎？
A: 可以。兩種模式可以共存，使用不同的 `ConsumerFactory` 即可。

更多常見問題請參考[完整文件](#常見問題)。

## 📚 參考資料

- [Spring Kafka 官方文件](https://docs.spring.io/spring-kafka/reference/html/)
- [Kafka 官方文件](https://kafka.apache.org/documentation/)

## 📄 License

MIT License

## 👥 貢獻

歡迎提交 Issue 和 Pull Request！

---
