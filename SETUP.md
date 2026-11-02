# Firebase 第一次部署｜給 Claude 的施工說明

你正在帶一位**從沒部署過 Firebase、也不一定懂程式**的使用者，把網頁放上網。
請一步一步帶，一次只叫他做一件事，用白話講，不要叫他自己判斷技術問題。

完工定義：使用者拿到真實網址、用手機和電腦都打開看過、程式碼有備份到私人 GitHub。

## 七條鐵律

1. **環境不對就停**：`firebase` 要在使用者自己的電腦上跑（見第 0 步）。
2. **每次部署都寫明專案**：一律帶 `--project <專案ID>`，部署前先念給使用者確認。
3. **鑰匙不進對話、不進 repo、不進 `public/`**：服務帳戶 JSON、`.env`、LINE／Gemini／OpenAI 金鑰都一樣。使用者貼給你就請他作廢重發。唯一例外：第 3 步的一次性登入碼。
4. **`public/` 裡的東西等於公開給全世界**：部署前列出檔案清單給使用者看，Excel、CSV、資料匯出檔不准進去。
5. **不准升級 Blaze 付費方案**，除非使用者聽完第 7 步說明並明確同意。
6. **資料庫規則絕不開放**：不准 `allow read, write: if true`，也不要長期用「測試模式」。
7. **沒看到就不說成功**：「Deploy complete」不算完工，要實際打開網址看到內容。

## 第 0 步：確認環境

先問使用者「你的電腦是 Windows 還是 Mac？」，再用 `uname -a`（PowerShell 用 `$PSVersionTable.OS`）看你的 shell 跑在哪裡。

- shell 的作業系統和使用者一致 → 繼續。`node -v` 找不到指令只代表還沒裝，第 2 步會處理。
- 使用者是 Windows／Mac，但 shell 是 Linux，或你在 Cowork、網頁版這類雲端沙盒 → **停止**，告訴他：

  > 這件事要在你自己的電腦上做，登入狀態和檔案才留得住。請打開 Claude 桌面版的 **Code** 分頁，選 Local，資料夾選你的專案，再把同一句話貼過去。

## 第 1 步：問三件事

1. 這個網頁只是給人看，還是要存資料、要登入、要串通知？
2. 網頁檔案在電腦的哪個資料夾？
3. 要用哪個 Google 帳號？（建議實驗用的帳號，別和公司重要帳號混用）

分流：

| 需求 | 做法 |
|---|---|
| 只給人看 | 只用 Hosting，走完第 2～6 步 |
| 要存資料或登入（多人共用一份、要帳號） | 第 2～6 步 + 第 8 步（資料庫、登入、規則都用指令接好） |
| 要定時、串 LINE／金流、藏金鑰的後端 | 需要 Cloud Functions，先走第 7 步 |

## 第 2 步：準備工具

- 沒有 Node.js：請使用者到 <https://nodejs.org> 裝 LTS 版。
- 下面的 `firebase` 都代表 `npx -y firebase-tools@latest`，不用全域安裝。
- Windows 的 PowerShell 擋 `npx` 時改用 `npx.cmd`。

## 第 3 步：登入

```bash
firebase login --non-interactive
```

把印出的網址給使用者，請他登入後把授權碼貼回來，再跑 `firebase login <授權碼>`。
Windows 在連結印出後可能跳 `Assertion failed`，連結有印出就先繼續。

這條走不通時，請他自己開終端機跑 `npx -y firebase-tools@latest login`，瀏覽器登入完回來說一聲。
用 `firebase login:list` 確認帳號正確。

## 第 4 步：建專案

新帳號要先同意條款，所以第一個專案請使用者自己建：

1. 打開 <https://console.firebase.google.com>，新增專案。
2. 取名時留意專案 ID，**它會變成網址，之後不能改**。
3. Google Analytics 關掉。不需要綁信用卡。

接著確認：

```bash
firebase projects:list
firebase hosting:sites:list --project <專案ID>
```

新專案通常已自動有網站。清單是空的才跑 `firebase hosting:sites:create <專案ID> --project <專案ID>`；回 already exists 就當作已有。

## 第 5 步：放設定檔

⚠️ **不要跑 `firebase init`**，它會停在互動問答，直接把下面三個檔案寫到專案根目錄。

網頁檔放進 `public/`，入口要是 `public/index.html`。React、Vite 這類先 `npm run build`，再把 `public` 改成 `dist`。

`firebase.json`：

```json
{
  "hosting": {
    "public": "public",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "headers": [
      { "source": "/", "headers": [{ "key": "Cache-Control", "value": "no-cache, no-store, must-revalidate" }] },
      { "source": "**/*.@(html|js|css)", "headers": [{ "key": "Cache-Control", "value": "no-cache, no-store, must-revalidate" }] }
    ]
  }
}
```

`.firebaserc`：

```json
{ "projects": { "default": "<專案ID>" } }
```

`.gitignore`：

```gitignore
node_modules/
.firebase/
.env
.env.*
*-firebase-adminsdk-*.json
service-account*.json
*.log
```

快取設定不能省，少了它使用者會一直看到舊版。

## 第 6 步：部署與驗收

1. 列出 `public/` 檔案給使用者看（鐵律 4）。
2. 部署：

```bash
firebase deploy --only hosting --project <專案ID>
```

3. 驗收：
   - 用 `curl -I https://<專案ID>.web.app/`（Windows 用 `curl.exe -I`）確認 200，且 `Cache-Control` 含 `no-cache`。
   - 有 Playwright 就用無頭瀏覽器開桌機和手機寬度，`pageerror` 要是 0，主按鈕要按得動。
   - 請使用者用手機和電腦各開一次。
4. 把網址單獨一行給他。

## 第 7 步：要花錢的東西（只有需要時才做）

Cloud Functions 需要 Blaze 方案，要綁信用卡。升級前先白話講清楚：

- Blaze 是用多少付多少，**預算警示只會通知，不會自動停止收費**。
- 請使用者設預算警示，並先估這次改動的讀寫量。
- 寫 Cloud Function 時，trigger **絕不能寫回觸發自己的文件**而沒有防迴圈。曾有人這樣 4 天燒掉上千美元。
- 排程至少間隔 5 分鐘，不要設 min-instances。
- 金鑰用 `firebase functions:secrets:set`，不寫進前端。

使用者同意了才繼續。

## 第 8 步：要存資料或登入時（全部用指令，不叫使用者去主控台點）

下面每一步都在 2026-11-02 用免費 Spark 專案實測過。順序不能換：先建網頁 App，再建資料庫，再開登入，最後寫規則。

### 8.1 建網頁 App、拿連線設定

```bash
firebase apps:create WEB <名稱> --project <專案ID>
firebase apps:sdkconfig WEB --project <專案ID>
```

第二條會印出 `projectId`、`appId`、`apiKey`、`authDomain`……這組就是網頁連 Firebase 的 `firebaseConfig`。**放進前端是正常的**（apiKey 不是密碼），真正的防線是 8.4 的規則。只部署 Hosting 不會自動產生這組設定，一定要跑這步。

### 8.2 建資料庫（用部署建，不要用 databases:create）

新專案的資料庫服務還沒開，直接跑 `firestore:databases:create` 會回 `403 Cloud Firestore API has not been used`。正確做法是把地區寫進 `firebase.json`，讓部署指令自己開服務、建資料庫、上規則：

```json
{
  "hosting": { ...第 5 步的內容... },
  "firestore": {
    "database": "(default)",
    "location": "asia-east1",
    "rules": "firestore.rules"
  }
}
```

先照 8.4 寫好 `firestore.rules`，再跑：

```bash
firebase deploy --only firestore --project <專案ID>
firebase firestore:databases:get "(default)" --project <專案ID>
```

第二條要看到 `Location: asia-east1`（台灣）。⚠️ `location` 不能省：少了它資料庫會建在美國 `nam5`，而且建了改不了，只能刪掉重建。

### 8.3 開登入（Google 登入，一條指令）

新手一律用 Google 登入：不用設密碼、信箱一定是驗證過的。在 `firebase.json` 最外層加：

```json
"auth": {
  "providers": {
    "googleSignIn": {
      "oAuthBrandDisplayName": "<網頁名稱>",
      "supportEmail": "<使用者的 Google 信箱>"
    }
  }
}
```

```bash
firebase deploy --only auth --project <專案ID>
```

看到 `Auth providers enabled: Google sign-in` 才算開了。沒開的話網頁登入會報 `CONFIGURATION_NOT_FOUND`。
不要開 Email／密碼註冊：開了任何人都能自己註冊進來，只能靠規則擋，多一層風險。

### 8.4 規則：先問「誰能看、誰能改」，再選一種

第 1 步問過使用者誰要用。三種情況，選一種寫進 `firestore.rules`：

**A. 只有自己用**（每個人只看得到自己的資料）：

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{uid}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

**B. 固定幾個人共用一份資料**（最常見：排班表全店看同一份、只有老闆能改）：

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function signedIn() {
      return request.auth != null && request.auth.token.email_verified == true;
    }
    function canView() {
      return signedIn() && request.auth.token.email in ['a@gmail.com', 'b@gmail.com'];
    }
    function canEdit() {
      return signedIn() && request.auth.token.email in ['a@gmail.com'];
    }
    match /<共用集合>/{doc} {
      allow read: if canView();
      allow write: if canEdit();
    }
  }
}
```

名單寫真實信箱，由使用者唸給你。人員變動就改名單重新部署規則，新手這樣最不會出錯。

**C. 名單要常常改、不想每次部署**：把名單放進資料庫的 `members` 集合，規則用 `exists()` 查。這條比較進階，使用者的人數穩定就先用 B。

⛔ 任何情況都不准 `allow read, write: if true`，也不准用「測試模式」撐著。

寫好規則後重新部署：`firebase deploy --only firestore:rules --project <專案ID>`。

### 8.5 網頁接線

用官方 modular SDK，從 CDN 載入即可，不用打包工具。骨架：

```html
<script type="module">
import { initializeApp } from 'https://www.gstatic.com/firebasejs/11.0.2/firebase-app.js';
import { getAuth, GoogleAuthProvider, signInWithPopup, onAuthStateChanged } from 'https://www.gstatic.com/firebasejs/11.0.2/firebase-auth.js';
import { getFirestore, collection, getDocs, setDoc, doc } from 'https://www.gstatic.com/firebasejs/11.0.2/firebase-firestore.js';

const app = initializeApp({ /* 8.1 印出的 firebaseConfig */ });
const auth = getAuth(app);
const db = getFirestore(app);
// 登入按鈕 → signInWithPopup(auth, new GoogleAuthProvider())
// onAuthStateChanged(auth, user => { 登入後才讀寫 collection(db, '<共用集合>') })
</script>
```

讀寫失敗時把錯誤碼顯示在畫面上（`permission-denied` 代表不在名單），不要默默吞掉。

### 8.6 驗收（沒做完不准說「接好了」）

1. 沒登入的人讀不到：
   ```bash
   curl -s "https://firestore.googleapis.com/v1/projects/<專案ID>/databases/(default)/documents/<共用集合>?key=<apiKey>"
   ```
   要看到 `PERMISSION_DENIED`。
2. 請使用者用**名單上的** Google 帳號登入網頁，真的寫一筆、再重新整理看得到。
3. 用一個**不在名單上**的 Google 帳號（家人的、或另一個信箱）登入，要看到 `permission-denied`，而且畫面有講清楚。
4. 有 Playwright 就無頭開一次，`pageerror` 要是 0。
5. 真實個資（名字、電話、排班）要先跟使用者確認名單對了，才放真資料。

Google 登入的跳窗只能真人點，無頭測試測不到；第 2、3 步一定要使用者本人實際操作。

要上傳檔案的 Cloud Storage 需要 Blaze，先走第 7 步。

## 第 9 步：備份到 GitHub

沒設過 Git 身分時（`git config user.name` 是空的），請使用者提供暱稱，email 建議用 GitHub 的 noreply 信箱，別讓私人信箱進 commit 紀錄。

`git init` 後先 `git status` 檢查，確定沒有鑰匙和資料檔才加入。

```bash
git init
git status
git add .
git commit -m "first deploy"
gh repo create <名稱> --private --source=. --push
```

沒有 `gh` 時請使用者裝 GitHub CLI 並跑 `gh auth login`。**預設私人**，使用者明確要求才公開；公開過的鑰匙視同外洩。

## 選配：push 就自動上線

使用者要的話再做。這個指令是互動式的，**請他自己在終端機跑**：

```bash
npx -y firebase-tools@latest init hosting:github
```

它會建立部署用的服務帳戶，把金鑰加密存進 GitHub Secret，並產生 `.github/workflows/`。
這些檔案要 commit 後推上 GitHub 才會生效，再到 repo 的 Actions 頁面確認綠燈。預覽網址接的也是**真實資料庫**。

## 之後每次修改

改檔 → 本機看過 → `firebase deploy --only hosting --project <專案ID>` → 驗收 → `git commit`、`git push`。
開了 Service Worker 或 PWA 的話，還要更新版本號，否則手機會卡舊版。

## 常見錯誤

| 看到 | 處理 |
|---|---|
| `Failed to authenticate` | 重走第 3 步 |
| 登入時 `Premature close`、`credentials no longer valid` | 多半是 Node 版本問題（2026 年曾有 24.17、22.23 的回報），換另一個 LTS 小版本再試 |
| `403`、`permission denied` | 帳號不對，用 `firebase login:list` 檢查 |
| `CONNECT tunnel failed`、proxy 回 403 | 你在雲端沙盒，回第 0 步 |
| 找不到 site | `firebase hosting:sites:create` |
| 網站是舊版 | 檢查 `firebase.json` 的快取設定，再開無痕視窗 |
| 網頁空白 | 看瀏覽器主控台錯誤，檢查 `public` 路徑和資源路徑 |
| `403 Cloud Firestore API has not been used` | 新專案還沒開資料庫服務，不要去主控台點，照第 8.2 步用 `deploy --only firestore` 建 |
| `CONFIGURATION_NOT_FOUND` | 登入功能還沒開，照第 8.3 步 `deploy --only auth` |
| 登入成功但 `permission-denied` | 這個人不在規則的名單裡，或信箱未驗證；改名單，不准改成 `if true` |
| 資料庫建在 `nam5` | `firebase.json` 少寫 `location`，資料庫刪了重建（第 8.2 步） |
| 卡住超過 2 次 | 先查錯誤全文，不要亂改，也不要改用更寬的權限硬過 |

## 交付時回報

用白話告訴使用者：網址、專案 ID、登入的 Google 帳號、GitHub 位置、是否有任何會收費的東西、下次怎麼修改。
