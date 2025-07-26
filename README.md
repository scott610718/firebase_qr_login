# Firebase QR Code Login (跨裝置登入範例)

這個專案示範如何使用 **Firebase Authentication** 和 **Cloud Firestore** 在桌機與手機之間建立掃描 QR Code 的跨裝置登入流程。桌機端產生一個包含隨機 `sessionId` 的 QR Code，手機掃描後使用 Google 帳號在 Firebase 登入，並將憑證寫入 Firestore。桌機端監聽同一 `sessionId` 的文件，一旦收到手機端寫入的登入資訊，就使用相同的 Google 憑證登入。

> **重要注意事項**
>
> * 此範例僅用於展示如何將 Firebase 用作跨裝置登入的後端，實際登入的是 **你的 Firebase 應用程式**，而非直接登入 Gmail、YouTube 等 Google 服務。要在瀏覽器中登入 Google 帳號，請使用 Google 官方的「使用手機登入」或 Passkey 機制。
>
> * 在生產環境中不建議將 `idToken` 或 `accessToken` 直接存入資料庫。更安全的做法是透過後端產生自訂憑證，或在桌機端重新走完整的 Google OAuth 流程。此範例為展示概念而簡化實作，請務必注意權限與有效期限設定。

## 專案結構

```
firebase_qr_login/
├── desktop.html      # 桌機登入頁面：產生 sessionId、顯示 QR Code、監聽 Firestore
├── mobile.html       # 手機登入頁面：解析 sessionId、Google 登入、將憑證寫入 Firestore
├── firestore.rules   # Firestore 安全規則範例
└── README.md         # 本說明文件
```

## 前置作業

1. **建立 Firebase 專案** – 到 [Firebase Console](https://console.firebase.google.com) 建立專案並啟用 *Authentication* 與 *Cloud Firestore*。
2. **啟用 Google 登入** – 在 Authentication → Sign‑in method 中啟用 Google，並將 GitHub Pages 或你要部署的網域加入「授權網域」。
3. **取得 Firebase 設定** – 在專案設定中找到「Firebase SDK snippet」，選擇 *Config*，把 `apiKey`、`authDomain`、`projectId`、`storageBucket`、`messagingSenderId`、`appId` 等資訊記錄下來，稍後會填入 HTML 中。

4. **建立 Google OAuth Client ID** – 在 [Google Cloud Console](https://console.cloud.google.com/apis/credentials) 的「憑證」頁面，選擇你的專案並按「建立憑證」→「OAuth 用戶端 ID」。在應用程式類型選擇「Web 應用程式」，並於「授權的 JavaScript 來源」欄位加入你的網站網域（例如 `https://yourname.github.io`），開發階段可以加上 `http://localhost`【891949438378477†L105-L124】。建立後會得到一個 `client ID` 形如 `1234567890-abc123def456.apps.googleusercontent.com`，稍後會在手機端畫面由使用者輸入。

5. **設定 Firestore 安全規則** – 在 Firestore 的「規則」頁面貼上本專案提供的 [`firestore.rules`](firestore.rules) 內容，並發布規則，以限制只有已登入的用戶可以建立 session，但任何人都可以讀取 session（因為桌機端尚未登入）。安全規則中的 `request.auth` 代表請求者的身份；範例中 `allow read, write: if request.auth != null;` 表示只有登入使用者才能對文件讀寫【462772808244894†L1320-L1345】。

## 目錄

### 1. 使用流程概述

* **桌機端 (desktop.html)**
  1. 載入 Firebase SDK，產生一個隨機的 `sessionId`。
  2. 將 `sessionId` 置入 `mobile.html` 的網址中，例如：`mobile.html?session=abc123`。
  3. 使用第三方 QR Code 函式庫將此網址轉成 QR Code 顯示在畫面上。
  4. 監聽 Firestore `sessions/{sessionId}` 文件，等待手機端寫入憑證。
  5. 收到憑證後用 `GoogleAuthProvider.credential(idToken, accessToken)` 建立憑證並呼叫 `signInWithCredential()` 登入 Firebase【343222603138510†L1642-L1648】。
  6. 登入成功後顯示使用者名稱、Email 及頭像，並選擇性地刪除 Firestore 中的 session 文件。

* **手機端 (mobile.html)**
  1. 解析網址中的 `session` 參數，取得 `sessionId`。
  2. 提供「Google 登入」按鈕，呼叫 `signInWithPopup()` 或 `signInWithRedirect()` 取得使用者授權，從結果中取得 `credential.accessToken` 以及 `result.user` 物件【343222603138510†L1400-L1420】。
  3. 登入成功後呼叫 `user.getIdToken(true)` 取得 ID token【329245863243657†L1398-L1414】。
  4. 將 `idToken`、`accessToken`、`uid`、`displayName`、`email`、`photoURL` 等資訊寫入 Firestore `sessions/{sessionId}` 文件。
  5. 顯示提示訊息，告知使用者可以回到桌機。

### 2. Firebase Firestore 規則 (`firestore.rules`)

檔案 [`firestore.rules`](firestore.rules) 提供了基本的安全設定：

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // sessions 集合
    match /sessions/{sessionId} {
      // 僅允許已登入使用者建立新的 session 文件
      allow create: if request.auth != null;
      // 允許任何使用者讀取 session 文件（桌機尚未登入）
      allow read: if true;
      // 僅允許擁有者刪除自己的 session 文件
      allow delete: if request.auth != null && request.auth.uid == resource.data.uid;
      // 禁止更新，防止覆寫憑證
      allow update: if false;
    }
  }
}
```

範例中使用的 `request.auth` 變數是 Firebase 安全規則判斷用戶身份的重要來源，當它非 null 時代表使用者已登入【462772808244894†L1320-L1345】。你可以依需求調整這些規則，例如限制讀取必須在短時間內或加入其他欄位驗證。

### 3. 部署與使用

1. **填入 Firebase 設定** – 打開 `desktop.html` 和 `mobile.html`，將 `firebaseConfig` 內的值替換為你自己的 Firebase 專案設定。不要在程式碼中硬編碼 Google OAuth Client ID。
2. **推送到 GitHub Pages** – 將整個 `firebase_qr_login` 資料夾放到 GitHub 儲存庫並啟用 GitHub Pages，或使用其他靜態網頁托管服務。確保你的網域已加入 Firebase Authentication 的授權網域列表。
3. **開啟桌機端頁面** – 在桌機瀏覽器開啟 `desktop.html`。這會產生 QR Code，掃描即可在手機端打開 `mobile.html?session=<sessionId>`。
4. **手機輸入 Client ID 並登入** – 手機端載入 `mobile.html` 後會先要求輸入 Google OAuth Client ID。輸入後會顯示「登入」按鈕；點擊並使用 Google 帳號登入即可授權並寫入 Firestore，桌機端便會自動登入。

## 結語

這個範例利用 Firebase 提供的工具實現了簡易的跨裝置登入流程。它遵循 OAuth 2.0 裝置授權流程的精神，即在另一裝置上輸入驗證碼並取得授權【502017976296811†L247-L309】。在實務應用中，你可以把 session 和憑證的管理移至雲端函式，並在桌機端透過自訂憑證登入，以提升安全性。當在正式環境部署時務必開啟 HTTPS、限制憑證讀取及過期時間，並考慮使用 WebSocket 取代輪詢，提高即時性與效率。