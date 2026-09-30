# Sub Kill Fabric 伺服器端模組

監控你 YouTube 頻道的總訂閱數，只要超過歷史最高紀錄，就在伺服器內部直接執行指令（預設 `kill @a`）。
是「伺服器端模組」，玩家連線完全不需要安裝這個模組。

## 防刷訂閱邏輯

只記錄「歷史最高訂閱數」，不是目前訂閱數：
- 有人退訂，數字下降，不觸發、也不會扣掉紀錄。
- 重新訂閱最多打平先前的最高紀錄，不會超過，所以不會再次觸發。
- 只有真正超越歷史最高點的淨成長，才會觸發一次指令。

## 安裝步驟

### 1. 用官方範本產生專案骨架

到 https://fabricmc.net/develop/template/，選擇你伺服器對應的 Minecraft 版本、填 Mod ID
（例如 `subkill`）、套件名稱（例如 `com.example.subkill`），產生並下載專案。

這一步一定要用官方產生器，不要自己手動拼 build.gradle，因為 Fabric Loader / Fabric API /
Yarn mappings 的版本號會隨時間更新，用產生器才能保證版本互相搭配、編譯得過。

### 2. 放入模組邏輯

把 `SubKillMod.java` 複製到範本專案的：
```
src/main/java/你的套件路徑/SubKillMod.java
```
如果範本自動產生了範例類別（例如 `ExampleMod.java`），可以直接刪掉或忽略它。

### 3. 修改 fabric.mod.json

打開範本裡的 `src/main/resources/fabric.mod.json`，照 `fabric.mod.json.snippet.txt` 裡的內容，
把 `entrypoints` 改成指向 `SubKillMod`，並加上 `"environment": "server"`。

### 4. 編譯

有兩種方式，擇一即可：

**方式 A：用 GitHub Actions 雲端編譯（推薦，電腦不用裝 Java/Gradle）**

1. 到 GitHub 建立一個新的空 repo（public 或 private 都可以，private 也能免費用 Actions）。
2. 在專案資料夾（範本 + 已放好 `SubKillMod.java`、改好 `fabric.mod.json`、且已包含這裡附的
   `.github/workflows/build.yml` 和 `.gitignore`）裡，用終端機執行：
   ```bash
   git init
   git add .
   git commit -m "init"
   git branch -M main
   git remote add origin 你的repo網址.git
   git push -u origin main
   ```
3. 推上去之後，到 GitHub 網頁上該 repo 的 **Actions** 分頁，會看到一個工作流程自動開始執行，
   等它跑完出現綠色勾勾。
4. 點進那次執行紀錄，下方 **Artifacts** 區塊會有一個 `subkill-mod` 可以下載，
   下載後解壓縮，裡面的 jar 就是編譯好的模組。
5. 之後只要修改程式碼再 `git push`，Actions 就會自動重新編譯一次，不用自己裝環境。

   注意：`.github/workflows/build.yml` 裡設定的是 JDK 21，這對應 1.20.5 以後的 Minecraft
   版本；如果範本產生器顯示你的版本需要 Java 17，把 workflow 檔裡的 `java-version: "21"`
   改成 `"17"` 即可。

**方式 B：在自己電腦本機編譯**

```bash
./gradlew build
```
（Windows 用 `gradlew.bat build`），編譯完成後 jar 檔會在 `build/libs/` 資料夾裡。

### 5. 安裝到伺服器

把編譯出來的 jar 丟進伺服器的 `mods/` 資料夾（伺服器本身要是 Fabric Server，也就是用
`fabric-server-launch.jar` 或官方安裝器裝好的伺服器），重新啟動伺服器。

### 6. 取得並填入 YouTube 授權資訊

第一次啟動時，模組會自動在伺服器的 `config/subkill-mod.json` 產生一份預設設定檔，並在
console 顯示「已建立預設設定檔...請填入...後重啟伺服器」，這時伺服器會正常啟動、只是模組
還不會運作。照 `config-subkill-mod.example.txt` 的說明取得 `clientId`、`clientSecret`、
`refreshToken` 並填入這個設定檔，存檔後重啟伺服器即可開始運作。

## 常見問題

- **編譯時找不到 `executeWithPrefix` 這個方法？** 不同 Minecraft 版本 API 名稱可能略有差異，
  在 IDE 裡對 `server` 物件按自動完成，找「以主控台身份執行一段指令字串」的方法換上去即可，
  其他邏輯不用改。
- **一定要用模組嗎？** 如果覺得寫模組太麻煩，之前給你的外部 Node.js + RCON 方案功能完全一樣，
  兩種方式擇一即可，不用兩個都裝。
- **YouTube API 沒有即時訂閱事件**，這裡一樣是輪詢，`pollIntervalSeconds` 預設 30 秒，
  有偵測延遲是正常的。
