# LLM 微調學習歷程

## 緣起

我很早以前就思考能否利用 AI 搭配即時通訊軟體，作為往生者回應對話的延伸，在面對意外時藉由 LLM 技術安撫在世的人。近年 AI 技術發展神速，終於讓一般人也能依照個人資料訓練模擬特定風格的模型，以下是我的學習過程。

## 資料準備

### 對話紀錄轉換

首先我導出 LINE 的對話紀錄，格式如下：

```
18:57 Darren 一起吃晚飯？
18:57 Darren 泰式？
19:05 Chun 好
19:07 Chun 哪等
19:07 Chun 土城？
19:07 Darren 看你
19:08 Darren 現在要搭捷運
19:09 Darren 還是想吃花雕雞？
```

原本打算直接用 ChatGPT 將對話紀錄轉為高品質的 JSON 資料，但考慮到資料量龐大且不想將對話隱私上傳雲端，因此決定自行開發解決方案。

### 資料處理工具開發

我使用 LINQPad 開發了專用的解析器，下面是核心功能的程式碼片段及說明：

#### 主要處理流程

```csharp
// 主要處理流程
async Task<int> ProcessChatLogByDayWithImmediateWriteAsync(string chatLogFilePath, string apiUrl, string modelName, string outputFilePath)
{
    int totalObjectsProcessed = 0;
    
    // 讀取整個聊天紀錄檔案
    string[] allLines = File.ReadAllLines(chatLogFilePath);
    
    // 儲存每天的對話
    Dictionary<string, List<string>> conversationsByDay = new Dictionary<string, List<string>>();
    
    // 當前日期
    string currentDate = "";
    
    // 逐行讀取並根據日期分組
    foreach (string line in allLines)
    {
        // 檢查是否為日期行（格式如 2023/02/28（二））
        if (IsDateLine(line))
        {
            currentDate = line.Trim();
            if (!conversationsByDay.ContainsKey(currentDate))
            {
                conversationsByDay[currentDate] = new List<string>();
            }
            conversationsByDay[currentDate].Add(currentDate);
        }
        else if (!string.IsNullOrWhiteSpace(line) && !string.IsNullOrWhiteSpace(currentDate))
        {
            conversationsByDay[currentDate].Add(line);
        }
    }
    
    // 逐一日期處理
    foreach (var date in conversationsByDay.Keys)
    {
        Console.WriteLine($"正在處理 {date} 的對話...");
        
        // 將當日對話轉換為單一字串
        string dayConversation = string.Join("\n", conversationsByDay[date]);
        
        // 呼叫 API
        var jsonObjects = await CallLMStudioApiAsync(apiUrl, modelName, dayConversation);
        
        if (jsonObjects != null && jsonObjects.Count > 0)
        {
            // 立即寫入檔案
            foreach (string jsonObject in jsonObjects)
            {
                File.AppendAllText(outputFilePath, jsonObject + Environment.NewLine);
                totalObjectsProcessed++;
            }
        }
    }
    
    return totalObjectsProcessed;
}
```

### 提示詞設計

為了讓 Gemma3 4b 模型能有效轉換對話紀錄，我設計了專門的提示詞。以下是提示詞的關鍵部分：

```
對話紀錄轉模型訓練 JSON（Gemma3-4b 專用強化版）

核心任務：
將 Darren 與 Chun 的對話紀錄轉換為嚴格過濾的高品質 JSON 訓練資料，專為 Gemma3-4b 小型模型優化。

強制過濾規則（無例外執行）：
1. **絕對禁止包含以下任何元素的訓練對話**：
   - 任何 URL/超連結（包含 http://、https://、www.、.com、.tw 等網址片段）
   - 任何非文字元素標記（[貼圖]、[照片]、[圖片]、[影像]、[語音]、[檔案]）
   - 任何不完整或意義不明的對話
   - 任何一方發送相同非文字元素的情況（如：input:[貼圖], output:[貼圖]）

輸出 JSON 格式：
[
  {
    "instruction": "極度簡潔的意圖標籤（2-5字）",
    "input": "Chun 的話語（清晰明確）",
    "output": "Darren 的回應（自然對話風格）"
  }
]

處理流程與嚴格篩選：

1. **前置檢查**（必須逐一執行）：
   - 檢查原始文本是否包含禁止元素（URL、[貼圖]等）
   - 檢查對話是否有明確的上下文和意義
   - 檢查對話長度是否適合（過短或過長都不適合）

2. **instruction 生成**：
   - 限定 2-5 個字的極簡標籤
   - 直接標示交互意圖（如：詢問意見、分享煩惱、討論計畫）

3. **input 處理**：
   - 保留原始話語的核心意義
   - 確保語意完整，避免不明確或過於簡短的表達

4. **output 處理**：
   - 完整保留原始回應的語氣和用詞特點
   - 確保回應與 input 邏輯對應
```

提示詞中特別強調了 Gemma3-4b 小型模型的特性考量：
- 保持數據簡潔：避免過長的對話
- 確保語意明確：每組對話都應有明確的目的
- 優先自然對話：重視日常對話模式的學習
- 避免過於特殊的情境：優先普適性的對話模式

## 訓練資料成果

經過處理後，成功產出符合訓練格式的 JSON 資料，例如：

```json
{"instruction":"工作安排","input":"認真上班窩","output":"好"}

{"instruction":"晚餐安排","input":"晚上吃什麼","output":"我再去買好了，還是有點餓"}

{"instruction":"確認上班","input":"到公司了嗎","output":"到了"}

{"instruction":"提醒飼料","input":"記得補飼料","output":"監視器線好像掉了"}

{"instruction":"詢問除蟎效果","input":"除蟎後媽媽有睡得比較好嗎？","output":"她應該沒有什麼感覺"}

{"instruction":"商議回家路","input":"你要把12街上去嗎","output":"還是我下班過去"}

{"instruction":"討論晚餐","input":"你想吃什麼？我今天不想煮飯","output":"叫外送吧，我想吃那家泰式料理"}
```

## 目前進度

1. **資料隱私處理的重要性**：自建工具可以有效控制隱私風險
2. **本地化 API 的便利性**：使用 LM Studio 搭配 Gemma3-4b 可在本地處理資料
3. **提示詞設計的關鍵作用**：清晰的指令和篩選標準能大幅提升資料質量
4. **資料格式的重要性**：合適的 instruction/input/output 格式對模型學習至關重要
5. **過濾機制的必要性**：嚴格的過濾確保訓練資料的乾淨和一致性

未來計劃進一步優化模型訓練方法，並探索更多應用場景，希望能讓這項技術真正服務於情感需求，成為生命延續的另一種可能。
