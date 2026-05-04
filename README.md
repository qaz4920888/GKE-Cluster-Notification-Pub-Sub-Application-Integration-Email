GKE Cluster Notification + Pub/Sub + Application Integration → Email
## 一、總體目標

當 GKE 發送通知事件（如升級、Security Bulletin），透過 Pub/Sub 接收，Application Integration 將其處理後用 Email 寄送通知給特定收件者。

---

## 🏗️ 二、資源架構

- **GKE Cluster**：啟用 `-notification-config` 發送通知到指定 Pub/Sub topic。
- **Pub/Sub Topic**：接收 GKE 發出的事件。
- **Application Integration**：由 Pub/Sub 觸發，進行資料解析與轉發。
    - **Data Mapping Task**：取出 `message.data` 與 `attributes`。
    - **Data Transformer Task**：拼接 `full_message` 通知內容。
    - **Send Email Task**：將 `full_message` 傳送至指定 Email。

---

## ⚙️ 三、實作流程詳細步驟

### 1️⃣ 建立 Pub/Sub Topic

```bash

gcloud pubsub topics create gke-cluster-events

控制台
直接建立即可 主要是要到Cluster 啟用通知在選定該pub sub
```

### 2️⃣ GKE 啟用通知功能
gcloud container clusters update cluster-2 \
  --region=us-central1 \
  --notification-config=pubsub=ENABLED,pubsub-topic=projects/<your-project-id>/topics/gke-cluster-events,filter=UpgradeEvent|SecurityBulletinEvent
<img width="1057" height="609" alt="image" src="https://github.com/user-attachments/assets/dc0ab955-1867-4209-86e7-a978f5923c7f" />
🔍 可改成接收所有通知：filter=（留空）
### 3️⃣ 建立 Application Integration

- 前往 GCP Console → Application Integration
- 點選 **"Create Integration"**
- 命名為 `Testgke`，選擇 `Cloud Pub/Sub Trigger`。
- 把地區選好即可開始

---

### 4️⃣ 建立 Trigger：Cloud Pub/Sub

- 選擇剛剛的 `gke-cluster-events` Topic
- 按 `+ Triggers` → 名稱：`CloudPubSubMessage`
    - 建立後填上右邊
    
    projects/YOUR_PROJECT_ID/topics/gke-notifications
    
    - 再加上SA
    - 至少要有: Application integration invoker
      <img width="1911" height="297" alt="image" src="https://github.com/user-attachments/assets/6a7edba6-7b90-4459-9252-cfc4eb3147b8" />
      <img width="1519" height="738" alt="image" src="https://github.com/user-attachments/assets/2d741f02-8eb3-48f9-82f3-ab2ff3b7062b" />
      ### 5️⃣ 建立 Task：Data Mapping

按 `+` 新增 Task → 選擇 `Data Mapping`，接著：
<img width="1180" height="680" alt="image" src="https://github.com/user-attachments/assets/289e0465-afea-4131-80f7-b7bbe94022c9" />
<img width="1915" height="869" alt="image" src="https://github.com/user-attachments/assets/74bc1fcf-320b-4bc5-8792-2f0b786084eb" />
### ⬅️ Input 欄位：

- `CloudPubSubMessage.data` → 解析為 `message_data`
- 參考圖片(一)

- `CloudPubSubMessage.attributes` → 命名為 `attributes`（JSON）(這邊要貼schema(圖片))
- Name: `attributes`
- Variable Type: `None`
- Data Type: `JSON`
- 選擇：`Enter a JSON schema`
- 貼上這段 schema：
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "cluster_name": { "type": "string" },
    "payload": { "type": "string" },
    "project_id": { "type": "string" },
    "cluster_location": { "type": "string" },
    "type_url": { "type": "string" }
  }
}
<img width="1253" height="721" alt="image" src="https://github.com/user-attachments/assets/ca3c20fb-8a24-4db9-a715-d44fa41a071f" />
- `attributes.cluster_name` → `cluster_name`
- `attributes.project_id` → `project_id`
- `attributes.cluster_location` → `cluster_location`
- `attributes.type_url` → `type_url`
- `attributes.payload` → `payload`

  ### ➡️ Output 變數（記得按「create a new one」）：

- 類型：`Output from Integration`
- 資料型別：全部為 `String`（除 `attributes` 是 JSON）
- 變數命名：
    - `message_data`
    - `cluster_name`
    - `project_id`
    - `cluster_location`
    - `type_url`
    - `payload`
    - `attributes`
 
      ✅ 完成後按右上角 Save
      <img width="1113" height="448" alt="image" src="https://github.com/user-attachments/assets/8bd8d317-293e-4d6b-ab49-de6c94549164" />
      <img width="577" height="728" alt="image" src="https://github.com/user-attachments/assets/0cb52b7b-032d-4332-b04b-9c7e79fd001d" />
<img width="1037" height="431" alt="image" src="https://github.com/user-attachments/assets/1003e615-79e2-49b4-a6f1-37c797713cfc" />
6️⃣ 建立 Task：Data Transformer (Preview)
- 新增 → 選擇 **Data Transformer**
- 點選 `Open Data Transformer Editor`
- 要先添加DIAGRAM加上Full message參數 (圖片)
- 按 `Script` → 切換到程式碼模式
- 新增變數：`full_message`（Output from Integration，String）
- <img width="1533" height="861" alt="image" src="https://github.com/user-attachments/assets/60e08078-a8d1-4f6d-8369-68e7322637f4" />
<img width="1553" height="797" alt="image" src="https://github.com/user-attachments/assets/d701faa5-0045-4392-a4b6-f6630700bf71" />
<img width="1553" height="797" alt="image" src="https://github.com/user-attachments/assets/cb9ed77c-ebc6-4096-a9c7-c04ff3285df2" />

<img width="1553" height="797" alt="image" src="https://github.com/user-attachments/assets/0642a752-05f0-43ac-bbbb-96f9aefe4576" />
<img width="1911" height="892" alt="image" src="https://github.com/user-attachments/assets/8cf554ba-e5a2-455a-a892-5b33127012b6" />
<img width="533" height="922" alt="image" src="https://github.com/user-attachments/assets/cd2f1ac1-aee9-4ab4-a64c-fa1863fd60e2" />
編寫程式碼
local message_data = std.extVar("message_data");
local cluster_name = std.extVar("cluster_name");
local project_id = std.extVar("project_id");
local cluster_location = std.extVar("cluster_location");
local type_url = std.extVar("type_url");
local payload = std.extVar("payload");

{
  full_message: "📣 GKE 通知 📣\n\n" +
                "📌 Project ID: " + project_id + "\n" +
                "📌 Cluster Name: " + cluster_name + "\n" +
                "📌 Location: " + cluster_location + "\n" +
                "📌 Event Type: " + type_url + "\n" +
                "📌 Message: " + message_data + "\n\n" +
                "📦 Payload:\n" + std.manifestJson(std.parseJson(payload))
}
✅ 完成後按右上角 Save

### 7️⃣ 建立 Task：Send Email

- 收件人：`example@example.com`
- 主旨：`GKE Notification`
- Body Format：Plain Text
- Body in Plain Text → 選擇變數 `full_message`

✅ 儲存後連接至 Data Transformer。
<img width="1910" height="869" alt="image" src="https://github.com/user-attachments/assets/05402ac0-db7a-4f49-a79a-2e7549b6155d" />
<img width="1917" height="749" alt="image" src="https://github.com/user-attachments/assets/2d8e9075-525a-40a2-a1e4-9348b7710821" />
### 8️⃣ 發布整合流程

- 點右上角 `Publish`
- 等待「Published version」出現（最多約 10 分鐘）
### 9️⃣ 測試

- 點選右上 `Test`
- 在右側測試介面 → 加入 JSON：

- {
  "data": "節點已升級至 1.33.0",
  "attributes": {
    "project_id": "my-gcp-project",
    "cluster_name": "cluster-2",
    "cluster_location": "us-central1",
    "type_url": "UpgradeEvent",
    "payload": "{\"nodePool\": \"default-pool\", \"version\": \"1.33.0-gke.2248000\"}"
  }
}
- 按 `Test Integration`

✅ 收到 email，內容應符合 `full_message` 格式。

📬 成品 Email 樣本

⚠️ GKE 通知 ⚠️

* Project ID: my-gcp-project
* Cluster Name: cluster-2
* Location: us-central1
* Event Type: UpgradeEvent

* Message: 節點已升級至 1.33.0

📦 Payload:
{
  "nodePool": "default-pool",
  "version": "1.33.0-gke.2248000"
}

<img width="1054" height="471" alt="image" src="https://github.com/user-attachments/assets/62a27d34-c5dd-4c5a-91ab-cf948e358e63" />

## 📌 補充筆記

- Pub/Sub 不會主動送通知 email，需要透過 Application Integration 或 Cloud Functions 實作轉發。
- Application Integration 不會因為用到 Send Email 而額外計費，只要使用的是 GCP 自帶的 Gmail service（每日限制）。
- Data Transformer 中使用 `std.extVar()` 對應到 Data Mapping 裡的變數名稱。
