# Firebase 第一次部署懶人包

把這份交給 Claude，它會帶你把網頁放上網。不用會寫程式，約 30 分鐘，純展示網頁不用綁信用卡。

## 三步驟

**第 1 步：打開 Claude 桌面版的 Code 分頁**

選 Local，資料夾選你的專案。

不要用 Cowork：部署要用到你電腦上的登入狀態和完整檔案，Cowork 的工作環境在雲端，任務結束就清空。

**第 2 步：把這段話貼給 Claude**

```text
我沒有部署過 Firebase。請先用 curl 把這份說明下載成本機檔案（Windows 用 curl.exe -L -o），再完整讀過一遍，不要用網頁摘要，然後一步一步帶我部署：
https://raw.githubusercontent.com/weifu0716/firebase-first-deploy/main/SETUP.md
需要我登入或點按鈕時，直接告訴我要做什麼。
```

**第 3 步：照 Claude 說的做**

它會請你登入 Google、建一個 Firebase 專案。完成後你會拿到一個 `.web.app` 網址。

## 你要準備

- 一個 Google 帳號
- 一個 GitHub 帳號（用來備份程式碼）
- 你的網頁檔案（還沒有的話，請 Claude 先幫你做一頁）

## 三個提醒

1. 金鑰不要貼進對話，也不要放進網頁資料夾。
2. 放進 `public/` 的東西，全世界都看得到。
3. 升級 Blaze 付費方案前先弄懂怎麼計費，預算警示不會自動停止收費。

## 授權

MIT，歡迎自由使用與修改。
