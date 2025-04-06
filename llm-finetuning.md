# LLM 模型微調學習筆記

## 前言

這是我嘗試將日常對話記錄轉換為 LLM 訓練資料的實驗記錄。目標是透過個人對話風格來微調模型，讓 AI 能捕捉到個人特有的語氣和表達方式。

## 資料處理流程概述

整個微調資料準備過程分為以下步驟：

- 第一次使用 LM Studio + Gemma 3 4b 搭配轉換用提示詞處理對話記錄
- 資料格式轉換與初步清洗 
- 使用 OpenAI API 進行二次深度清洗
- 最終資料整理與驗證

## Step 1:資料清洗進化過程

從原始對話到高品質訓練資料，經歷了多次精煉階段，每一步都顯著提升了資料質量：

### 原始資料（聊天記錄）
```
2023/03/09（四）
10:37	Darren	連落洗衣機了嗎
10:42	Chun	有。說會跟我聯繫
10:43	Darren	啥時回來？
10:43	Darren	會
10:43	Chun	等師傅打來啊
10:44	Chun	不知道哦。還沒接到電話
```

### 初步轉換（LM Studio + Gemma 3 4b 搭配轉換用提示詞）

提示詞設計

```
## 核心目標
將對話提煉為高品質訓練樣本，嚴格捕捉 Darren 的語氣、口頭禪與對話風格，排除干擾元素。

## 角色定義
- **Darren（丈夫）**：模型模仿對象，回應作為 **output**
- **Chun（妻子）**：對話引導者，話語作為 **input**

## 語言特徵
Darren 特徵：
- 口頭禪："恩恩"、"好der"、"看你"
- 標點習慣："~"、"! "
- 句式：簡短直接
```

```json
{"instruction":"惠而浦安裝","input":"你在家？你要可以嗎？","output":"我可以來了，你慢慢來"}
{"instruction":"生活協調","input":"連落洗衣機了嗎 Chun會跟我聯繫嗎？","output":"都可以欸，看你想吃什麼"}
```

👆 問題：內容混淆、上下文丟失、回應不匹配

### 程式碼處理

針對 Gemma 3 4b 仍會產生如 \n 或其他亂碼符號以程式過濾

👆 改進：移除了特殊符號、表情，但對話上下文仍不完整

## Step 2:使用 OpenAI API 進行深度資料清洗

為提高訓練資料品質，使用 OpenAI API 進行更精準的資料清洗與改進，針對內容進行意圖分類、格式標準化以及上下文關聯性處理。

API 處理流程：
```csharp
async Task<string> ProcessDataWithOpenAI(
    HttpClient client, 
    string endpoint, 
    TrainingData data, 
    string systemPrompt, 
    string userPromptTemplate,
    string modelName)
{
    // 準備使用者提示詞
    string userPrompt = userPromptTemplate
        .Replace("{instruction}", data.Instruction)
        .Replace("{input}", data.Input)
        .Replace("{output}", data.Output);
    
    // 發送請求至 OpenAI API
    var request = new OpenAIRequest {
        model = modelName,
        messages = new List<Message> {
            new Message { role = "system", content = systemPrompt },
            new Message { role = "user", content = userPrompt }
        }
    };
    
    // 處理回應...
}
```

第二次清洗後的最終成果：
```json
{"instruction":"安裝家電","input":"你在家嗎？惠而浦要安裝了","output":"恩恩，我可以來了，你慢慢來"}
{"instruction":"生活協調","input":"連落洗衣機了嗎 Chun會跟我聯繫嗎？","output":"恩恩，我會去看看，不用擔心"}
{"instruction":"討論晚餐","input":"晚上想吃什麼？想去吃火鍋","output":"都可以欸，看你想吃什麼"}
{"instruction":"生活協調","input":"你要下班了嗎？我來接你","output":"恩恩好der，可以來了，你慢慢來"}
```
👆 成果：保留了特徵性語氣（「恩恩」、「好der」）、清晰的對話意圖、內容連貫性

這次處理明顯改善了資料品質：
- 保留了特徵性語氣「恩恩」、「好der」等口頭禪
- 對話意圖更加清晰，通過 instruction 標籤精確分類
- 內容上下文連貫性大幅改善
- 人物對話風格更加一致和自然

處理過程中的資料統計：
- 處理資料：569 筆
- API 使用總量：501,376 tokens / 596 請求
- 花費：$0.81 USD (總預算 $10 USD)

## 資料清洗的關鍵價值發現

在多層次清洗過程中，得到了幾個關鍵洞察：

1. **提示詞設計至關重要**
   - 精確的提示詞能引導 API 理解如何保留人物特色語氣
   - 細節處理指南（如保留特定口頭禪）顯著提高資料品質

2. **資源與成本管理**
   - 本地模型處理初步清洗，保護隱私的同時降低成本
   - OpenAI API 用於高質量精煉，$0.81 投資獲得更佳樣本

3. **資料品質進階**
   - 簡單移除雜訊 → 保留個性化特徵 → 語意連貫性提升
   - 從原始對話到訓練樣本，信噪比逐步提高
   - 標準化處理實現了可擴展的資料清洗流程

4. **語言特徵捕捉技巧**
   - 從機械式清洗到語意理解式清洗的進化
   - 記錄口頭禪、標點習慣等微妙特徵並融入提示詞
   - 去除特定時間地點，保留普適性表達

## 目前進度與後續計劃

已完成高品質訓練資料的準備工作，獲得標準化且捕捉個人語氣特色的對話樣本，為實際模型微調奠定了堅實基礎。

後續計劃：
- 使用清洗後的資料進行實際模型微調
- 建立評估指標，測試微調效果
- 探索少量樣本（few-shot）微調技術
- 考慮建立自動化資料清洗管道

[返回主頁](index.html)