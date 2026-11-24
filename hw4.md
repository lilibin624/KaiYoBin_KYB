# 作業4：UML 類別圖、循序圖與活動圖

系統分析與設計115-1 第二組｜專題：HoloLens 2 聽障輔助溝通系統

## 一、UML 類別圖（Class Diagram）

```mermaid
classDiagram
    direction LR

    class RawMicCapture {
        -string speechChannels
        -bool autoGain
        -float targetLevelDb
        -float maxGain
        +event OnRawAudio(float[] samples, int channels, int sampleRate)
        +event OnSpeechPcm16k(byte[] pcm)
        +StartCapture() void
        +StopCapture() void
        -ApplyAutoGain(float[] samples) void
    }

    class DirectionEstimator {
        -PairModel[] pairs
        -float frontHalfWidthDeg
        -float wearerThresholdDb
        -List~Estimate~ timeline
        +TryGetDirection(DateTime from, DateTime to, out Dir dir, out bool wearer) bool
        -OnRawAudio(float[] samples, int channels, int sampleRate) void
        -CrossLag(int chA, int chB, out float lag, out float conf) void
        -EstimateAzimuth(Estimate est) void
        -AzimuthToDir(float azimuth) Dir
    }

    class PairModel {
        +int chA
        +int chB
        +float a
        +float b
        +float offset
    }

    class Estimate {
        +DateTime time
        +Dir dir
        +float azimuth
        +bool wearer
    }

    class Dir {
        <<enumeration>>
        Unknown
        Left
        Front
        Right
        Back
    }

    class SpeechTest {
        -SpeechRecognizer recognizer
        -PushAudioInputStream pushStream
        -int lineBreakChars
        -bool showDirection
        +StartRecognition() void
        +StopRecognition() void
        -OnRecognizing(string text) void
        -OnRecognized(string text) void
        -AddDirectionTag(string line, Dir dir, bool wearer) string
        -IsMutingMic() bool
        +event OnSentenceFinished(TranscriptEntry entry)
    }

    class SpeechConfig {
        <<static>>
        +string Key
        +string Region
    }

    class GeminiConfig {
        <<static>>
        +string ApiKey
        +string Model
    }

    class TranscriptEntry {
        +DateTime time
        +string text
        +Dir direction
        +bool isWearer
    }

    class TranscriptRecorder {
        -List~TranscriptEntry~ entries
        +Add(TranscriptEntry entry) void
        +GetRange(int minutes) List~TranscriptEntry~
        +GetAll() List~TranscriptEntry~
    }

    class ConversationSummarizer {
        -float timeoutSeconds
        +Summarize(int minutes) Task~string~
        -BuildPrompt(List~TranscriptEntry~ entries) string
    }

    class SmartReplyManager {
        -string[] fixedReplies
        +OnSentenceFinished(TranscriptEntry entry) void
        +OnReplySelected(int index) void
        -ShowGenerating(bool visible) void
    }

    class SmartReplySuggester {
        -float timeoutSeconds
        -int requestId
        +RequestReplies(string context) Task~string[]~
        -BuildPrompt(string context) string
        -FilterReplies(string[] raw) string[]
    }

    class SmartReplySpeaker {
        +bool IsSpeaking
        +Speak(string text) Task
    }

    class EnvironmentSoundDetector {
        -float confidenceThreshold
        -string[] targetClasses
        -OnRawAudio(float[] samples, int channels, int sampleRate) void
        -Classify(float[] frame) SoundEvent
        +event OnSoundDetected(SoundEvent e)
    }

    class SoundEvent {
        +string label
        +float confidence
        +Dir direction
        +DateTime time
    }

    class AlertPresenter {
        +ShowAlert(SoundEvent e) void
        +Dismiss() void
    }

    RawMicCapture --> SpeechTest : 提供 16kHz 語音
    RawMicCapture --> DirectionEstimator : 提供多聲道音訊
    RawMicCapture --> EnvironmentSoundDetector : 提供多聲道音訊
    DirectionEstimator "1" *-- "4" PairModel
    DirectionEstimator "1" *-- "*" Estimate
    Estimate --> Dir
    SpeechTest --> DirectionEstimator : 查詢方向
    SpeechTest ..> SpeechConfig : 讀取金鑰
    SpeechTest --> TranscriptRecorder : 存入字幕
    SpeechTest --> SmartReplyManager : 一句結束通知
    SpeechTest --> SmartReplySpeaker : 朗讀時靜音麥克風
    TranscriptRecorder "1" o-- "*" TranscriptEntry
    ConversationSummarizer --> TranscriptRecorder : 取出紀錄
    ConversationSummarizer ..> GeminiConfig
    SmartReplyManager --> SmartReplySuggester
    SmartReplyManager --> SmartReplySpeaker
    SmartReplySuggester ..> GeminiConfig
    EnvironmentSoundDetector --> DirectionEstimator : 查詢方向
    EnvironmentSoundDetector ..> SoundEvent : 產生
    AlertPresenter ..> SoundEvent : 顯示
    EnvironmentSoundDetector --> AlertPresenter
```

類別分工：
- **擷取層**：`RawMicCapture` 以 HoloLens 2 原始麥克風模式擷取多聲道音訊，分送給字幕、方向與環境音三個模組。
- **字幕與方向（李文豪）**：`SpeechTest`、`DirectionEstimator`、`PairModel`、`Estimate`、`Dir`。
- **紀錄、摘要與智慧回覆（高宗佑）**：`TranscriptRecorder`、`TranscriptEntry`、`ConversationSummarizer`、`SmartReplyManager`、`SmartReplySuggester`、`SmartReplySpeaker`。
- **環境音（葉育愷）**：`EnvironmentSoundDetector`、`SoundEvent`、`AlertPresenter`。
- `SpeechConfig`、`GeminiConfig` 存放 API 金鑰，不放進版本控制。

---

## 二、UC1 檢視即時字幕（含說話方向）

### 循序圖

```mermaid
sequenceDiagram
    autonumber
    actor P as 對話者
    participant Mic as RawMicCapture
    participant Dir as DirectionEstimator
    participant ST as SpeechTest
    participant AZ as Azure Speech
    participant TR as TranscriptRecorder
    actor U as 聽障使用者

    P->>Mic: 說話
    loop 每約 0.04 秒一批音訊
        Mic->>Dir: OnRawAudio(多聲道音訊)
        Dir->>Dir: 計算時間差，估計方位與是否為配戴者
        Mic->>ST: OnSpeechPcm16k(語音)
        ST->>AZ: 寫入語音串流
    end
    AZ-->>ST: Recognizing(辨識中文字)
    ST-->>U: 字幕面板即時更新
    AZ-->>ST: Recognized(最終文字)
    ST->>Dir: TryGetDirection(句子開始, 句子結束)
    Dir-->>ST: 方向、是否為配戴者、一致率
    alt 配戴者本人
        ST->>ST: 標示「我」、改為灰色
    else 一致率足夠
        ST->>ST: 加上 ←左 / ↑前 / →右 / ↓後
    else 一致率不足
        ST->>ST: 不加方向符號
    end
    ST-->>U: 顯示最終字幕並斷句換行
    ST->>TR: Add(時間, 文字, 方向)
```

### 活動圖

```mermaid
flowchart TD
    S((●)) --> A[擷取原始麥克風多聲道音訊]
    A --> B{音量超過門檻？}
    B -- 否 --> A
    B -- 是 --> FORK[[同時進行]]
    FORK --> C[送 16kHz 語音至 Azure Speech]
    FORK --> D[估計方位並存入時間軸]
    C --> E[顯示辨識中文字]
    E --> F{一句結束？}
    F -- 否 --> C
    F -- 是 --> G[查詢這句期間的方向]
    D --> G
    G --> H{配戴者本人？}
    H -- 是 --> I[標示「我」並改灰色]
    H -- 否 --> J{方向一致率足夠？}
    J -- 是 --> K[加上方向符號]
    J -- 否 --> L[不加方向符號]
    I --> M[顯示最終字幕並換行]
    K --> M
    L --> M
    M --> N[存入對話紀錄]
    N --> O((◉))
```

---

## 三、UC5 選擇智慧回覆

### 循序圖

```mermaid
sequenceDiagram
    autonumber
    participant ST as SpeechTest
    participant M as SmartReplyManager
    participant SG as SmartReplySuggester
    participant GM as Gemini API
    participant SP as SmartReplySpeaker
    actor U as 聽障使用者

    ST->>M: OnSentenceFinished(對方的話)
    M-->>U: 顯示「產生回覆中…」，按鈕維持固定句
    M->>SG: RequestReplies(對話內容)
    SG->>GM: 送出提示詞（12 秒逾時）
    alt 12 秒內成功
        GM-->>SG: 建議回覆文字
        SG->>SG: 移除標點、過濾過短句子
        SG-->>M: 4 句建議回覆
        alt 期間沒有新的一段話
            M-->>U: 按鈕換成建議回覆
        else 已有新的一段話
            M->>M: 丟棄舊結果
        end
    else 逾時或錯誤
        SG-->>M: 失敗
        M-->>U: 維持固定回覆句
    end
    M-->>U: 隱藏「產生回覆中」
    U->>M: OnReplySelected(第 n 句)
    M->>SP: Speak(回覆文字)
    SP->>ST: IsSpeaking = true
    ST->>ST: 改寫入靜音，不辨識朗讀聲
    SP-->>U: 喇叭朗讀回覆
    SP->>ST: IsSpeaking = false（再等 0.3 秒恢復）
```

### 活動圖

```mermaid
flowchart TD
    S((●)) --> A[對方說完一段話]
    A --> B[顯示「產生回覆中…」<br>按鈕維持固定句]
    B --> C[送對話內容至 Gemini]
    C --> D{12 秒內成功回傳？}
    D -- 否 --> E[記錄警告，維持固定句]
    D -- 是 --> F[移除標點、過濾少於 2 字的句子]
    F --> G{期間有新的一段話？}
    G -- 是 --> H[丟棄舊結果]
    G -- 否 --> I[按鈕換成 4 句建議回覆]
    E --> J[隱藏「產生回覆中」]
    H --> J
    I --> J
    J --> K{使用者點選回覆？}
    K -- 否 --> O((◉))
    K -- 是 --> L[暫停把麥克風送去辨識]
    L --> M[語音合成朗讀該句]
    M --> N[朗讀結束 0.3 秒後恢復辨識]
    N --> O
```

---

## 四、UC4 查詢對話摘要

### 循序圖

```mermaid
sequenceDiagram
    autonumber
    actor U as 聽障使用者
    participant UI as 手部選單／回顧面板
    participant TR as TranscriptRecorder
    participant CS as ConversationSummarizer
    participant GM as Gemini API

    U->>UI: 點選「回顧」
    UI->>TR: GetAll()
    TR-->>UI: 對話紀錄
    UI-->>U: 顯示對話紀錄
    U->>UI: 選擇摘要範圍（10／30／60 分鐘）
    UI->>CS: Summarize(分鐘數)
    CS->>TR: GetRange(分鐘數)
    TR-->>CS: 該時段字幕
    alt 沒有任何字幕
        CS-->>UI: 空結果
        UI-->>U: 顯示「這段時間沒有對話紀錄」
    else 有字幕
        CS->>GM: 送出摘要提示詞
        alt 成功
            GM-->>CS: 摘要文字
            CS-->>UI: 摘要
            UI-->>U: 顯示摘要
        else 失敗或逾時
            GM-->>CS: 錯誤
            CS-->>UI: 失敗
            UI-->>U: 顯示「摘要失敗，請稍後再試」
        end
    end
```

### 活動圖

```mermaid
flowchart TD
    S((●)) --> A[使用者開啟手部選單點選「回顧」]
    A --> B[顯示對話紀錄面板]
    B --> C[使用者選擇 10／30／60 分鐘]
    C --> D[取出該時段字幕]
    D --> E{有字幕？}
    E -- 否 --> F[顯示「這段時間沒有對話紀錄」]
    E -- 是 --> G[送至 Gemini 摘要]
    G --> H{成功？}
    H -- 是 --> I[顯示摘要結果]
    H -- 否 --> J[顯示「摘要失敗，請稍後再試」<br>原始紀錄仍可瀏覽]
    F --> O((◉))
    I --> O
    J --> O
```
