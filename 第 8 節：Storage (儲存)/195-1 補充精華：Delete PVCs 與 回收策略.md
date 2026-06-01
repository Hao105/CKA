# 195-1 補充精華：Delete PVCs 與 回收策略

## 1. 🏷️ 課程定位
- **章節編號與名稱：** 第 8 節：Storage (儲存)
- **影片標題：** 195. Persistent Volume Claims (補充精華：Delete PVCs 與 回收策略)

## 2. 📌 核心概念摘要
Reclaim Policy (回收策略) 定義了當一張 PVC (申請單) 被刪除時，Kubernetes 應該如何處置那塊曾經與它綁定的 PV (實體硬碟) 以及裡面的資料。這是保護企業機敏資料不被誤刪除，或是實現雲端硬碟自動化管理的最後一道防線。

## 3. 📊 流程圖與視覺化重現
我們將截圖中列出的三種策略，轉化為底層運作狀態的變化圖：

```mermaid
graph TD
    A[🗑️ 開發者執行: kubectl delete pvc myclaim] --> B{PV 的 Reclaim Policy 是什麼?}
    
    B -->|1. Retain (保留) | C[🛑 狀態轉為 Released]
    C --> C1[資料原封不動保留 <br/> 管理員需手動清理才能再次使用]
    
    B -->|2. Delete (刪除) | D[🔥 狀態直接消滅]
    D --> D1[PV 被刪除 <br/> 底層雲端硬碟 (如 EBS) 同步銷毀]
    
    B -->|3. Recycle (回收 - 已廢棄) | E[🔄 狀態轉為 Available]
    E --> E1[自動執行 rm -rf 清空資料 <br/> PV 直接開放給下一個 PVC 綁定]

    style C fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
    style D fill:#f8d7da,stroke:#dc3545,stroke-width:2px
    style E fill:#e0e0e0,stroke:#9e9e9e,stroke-dasharray: 5 5
```

## 4. 🔑 知識點擷取 (Detailed Notes)
這張截圖的表格非常經典，請務必掌握這三種策略的底層行為：

- **Retain (保留)：**
  - **觸發機制：** 這是靜態配置 PV 的預設行為。
  - **底層狀態：** PVC 刪除後，PV 的狀態會從 Bound 變成 `Released`。
  - **限制條件：** 處於 Released 狀態的 PV **絕對無法**被新的 PVC 綁定！因為裡面還有上一個使用者的舊資料。管理員必須手動刪除這個 PV 並重新建立，才能再次投入使用。

- **Delete (刪除)：**
  - **觸發機制：** 這是透過 StorageClass 動態配置 (Dynamic Provisioning) 的預設行為。
  - **底層狀態：** PVC 一刪除，PV 連同底層的實體硬碟（如 AWS EBS 或 GCP PD）會被立刻連動刪除，一了百了，節省雲端成本。

- **Recycle (回收 / ⚠️ Deprecated 已廢棄)：**
  - **觸發機制：** 如截圖下方的小終端機所示，它會在底層跑一個簡單的腳本 `rm -rf /scrub/*` 來把資料洗掉。
  - **限制條件：** 現代 Kubernetes 已經強烈不建議甚至不再支援此做法。因為單純的 `rm -rf` 不夠安全，現在的主流做法是直接用 Delete 刪除雲端硬碟，需要時再由 StorageClass 動態開一顆新的。

## 5. 💻 CKA 必備實作指令 (Imperative Commands)
考場上可能會要求你臨時更改某個 PV 的回收策略，這時候用 `patch` 指令會比編輯 YAML 快上非常多：

```bash
# 💡 刪除 PVC (觸發 Reclaim Policy)
kubectl delete pvc myclaim

# 💡 考場技巧：不修改 YAML 檔案，直接透過 patch 動態修改 PV 的回收策略為 Retain
kubectl patch pv <pv-name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# 💡 查詢現有 PV 的回收策略 (注意 RECLAIM POLICY 欄位)
kubectl get pv
```

## 6. 🚀 CKA 考試延伸與 Troubleshooting
### 🎯 考試情境預測：
- **經典考題：** 考題會要求你：「建立一個 PV，並確保當使用它的 PVC 被刪除時，PV 裡面的資料必須被保留，且不可被自動刪除」。這就是在考你是否知道要把 `persistentVolumeReclaimPolicy` 設為 `Retain`。

### 🛑 避坑指南 (Released 的迷思)：
- 很多新手以為 Retain 策略下，PVC 刪除後 PV 就可以直接給下一個人用。大錯特錯！ PV 會變成 `Released`，在沒有經過管理員手動把裡面舊的 `claimRef` (綁定紀錄) 清除並重建之前，它永遠不會變回可被綁定的 `Available`。

### 🔧 Troubleshooting：
- **現象：執行 `kubectl delete pvc <name>` 後，PVC 一直卡在 `Terminating` 刪不掉。**
  - **排查原因：** 這通常是因為**還有 Pod 正在使用（掛載）這張 PVC！**Kubernetes 有一個叫做 Storage Object in Use Protection 的保護機制，為了防止你的應用程式突然崩潰，它會阻止你刪除正在使用中的儲存。
  - **解法：** 先去 `kubectl delete pod` 刪除那個正在使用該 PVC 的 Pod，PVC 就會瞬間自動完成刪除流程。
