package com.example.subkill;

import com.google.gson.Gson;
import com.google.gson.GsonBuilder;
import net.fabricmc.api.DedicatedServerModInitializer;
import net.fabricmc.fabric.api.event.lifecycle.v1.ServerLifecycleEvents;
import net.minecraft.server.MinecraftServer;

import java.io.IOException;
import java.io.Reader;
import java.io.Writer;
import java.net.URI;
import java.net.URLEncoder;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.logging.Logger;

/**
 * 監控 YouTube 頻道總訂閱數，超過歷史最高紀錄時，在伺服器內部執行指令（預設 kill @a）。
 *
 * 防刷訂閱：只記錄「歷史最高訂閱數」，退訂造成的下降不算數，
 *          重新訂閱最多打平舊紀錄、不會超過，所以不會重複觸發。
 *
 * 注意：server.getCommandManager() / executeWithPrefix() 等 API 名稱依 Minecraft 版本
 * 及使用的 mappings（Yarn / Mojmap）可能略有差異。若編譯時找不到這個方法，
 * 到 IDE 裡對 server 物件按自動完成，找對應版本裡「以主控台身份執行指令字串」的方法替換即可，
 * 邏輯本身不用改。
 */
public class SubKillMod implements DedicatedServerModInitializer {
    public static final String MOD_ID = "subkill";
    private static final Logger LOGGER = Logger.getLogger(MOD_ID);
    private static final Gson GSON = new GsonBuilder().setPrettyPrinting().create();
    private static final HttpClient HTTP = HttpClient.newHttpClient();

    private ScheduledExecutorService scheduler;
    private Config config;
    private State state;
    private String accessToken;
    private long accessTokenExpiry = 0;
    private long lastTriggerTime = 0;

    @Override
    public void onInitializeServer() {
        ServerLifecycleEvents.SERVER_STARTED.register(this::onServerStarted);
        ServerLifecycleEvents.SERVER_STOPPING.register(server -> {
            if (scheduler != null) scheduler.shutdownNow();
        });
    }

    private void onServerStarted(MinecraftServer server) {
        try {
            config = loadOrCreateConfig();
        } catch (IOException e) {
            LOGGER.severe("[SubKill] " + e.getMessage());
            return;
        }
        if ("your_refresh_token".equals(config.refreshToken)) {
            LOGGER.warning("[SubKill] 尚未設定 refreshToken，請編輯 config/subkill-mod.json 後重啟伺服器。");
            return;
        }

        state = loadState();
        scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.scheduleWithFixedDelay(() -> pollOnce(server),
                5, config.pollIntervalSeconds, TimeUnit.SECONDS);
        LOGGER.info("[SubKill] 已啟動，每 " + config.pollIntervalSeconds + " 秒檢查一次訂閱數。");
    }

    private void pollOnce(MinecraftServer server) {
        try {
            long current = fetchSubscriberCount();

            if (!state.initialized) {
                state.highestCount = current;
                state.initialized = true;
                saveState();
                LOGGER.info("[SubKill] 初始化完成，目前訂閱數 " + current + " 設為基準，之後超過才會觸發。");
                return;
            }

            if (current <= state.highestCount) {
                return;
            }

            long gained = current - state.highestCount;
            LOGGER.info("[SubKill] 訂閱數增加：" + state.highestCount + " -> " + current + "（+" + gained + "）");
            state.highestCount = current;
            saveState();

            long now = System.currentTimeMillis();
            if (now - lastTriggerTime < config.triggerCooldownMs) {
                LOGGER.info("[SubKill] 冷卻中，這次先不觸發指令（紀錄已更新，不會遺漏）。");
                return;
            }
            lastTriggerTime = now;

            server.execute(() -> {
                server.getCommandManager().executeWithPrefix(server.getCommandSource(), config.killCommand);
            });
        } catch (Exception e) {
            LOGGER.warning("[SubKill] 輪詢失敗：" + e.getMessage());
        }
    }

    // ---------- YouTube API ----------

    @SuppressWarnings("unchecked")
    private long fetchSubscriberCount() throws IOException, InterruptedException {
        String token = getAccessToken();
        HttpRequest req = HttpRequest.newBuilder()
                .uri(URI.create("https://www.googleapis.com/youtube/v3/channels?part=statistics&mine=true"))
                .header("Authorization", "Bearer " + token)
                .GET().build();
        HttpResponse<String> res = HTTP.send(req, HttpResponse.BodyHandlers.ofString());
        if (res.statusCode() != 200) {
            throw new IOException("YouTube API 回傳錯誤 " + res.statusCode() + ": " + res.body());
        }
        Map<String, Object> json = GSON.fromJson(res.body(), Map.class);
        List<Object> items = (List<Object>) json.get("items");
        if (items == null || items.isEmpty()) throw new IOException("查無頻道資料，請確認 refreshToken 對應的帳號正確");
        Map<String, Object> item = (Map<String, Object>) items.get(0);
        Map<String, Object> statistics = (Map<String, Object>) item.get("statistics");
        return Long.parseLong((String) statistics.get("subscriberCount"));
    }

    @SuppressWarnings("unchecked")
    private String getAccessToken() throws IOException, InterruptedException {
        long now = System.currentTimeMillis();
        if (accessToken != null && now < accessTokenExpiry - 30_000) {
            return accessToken;
        }
        String form = "client_id=" + urlEncode(config.clientId)
                + "&client_secret=" + urlEncode(config.clientSecret)
                + "&refresh_token=" + urlEncode(config.refreshToken)
                + "&grant_type=refresh_token";
        HttpRequest req = HttpRequest.newBuilder()
                .uri(URI.create("https://oauth2.googleapis.com/token"))
                .header("Content-Type", "application/x-www-form-urlencoded")
                .POST(HttpRequest.BodyPublishers.ofString(form))
                .build();
        HttpResponse<String> res = HTTP.send(req, HttpResponse.BodyHandlers.ofString());
        if (res.statusCode() != 200) {
            throw new IOException("換取 access token 失敗 " + res.statusCode() + ": " + res.body());
        }
        Map<String, Object> json = GSON.fromJson(res.body(), Map.class);
        accessToken = (String) json.get("access_token");
        double expiresIn = (Double) json.get("expires_in");
        accessTokenExpiry = now + (long) (expiresIn * 1000);
        return accessToken;
    }

    private static String urlEncode(String s) {
        return URLEncoder.encode(s, StandardCharsets.UTF_8);
    }

    // ---------- 設定與狀態存取 ----------

    private static Path configPath() {
        return Path.of("config", "subkill-mod.json");
    }

    private static Path statePath() {
        return Path.of("config", "subkill-state.json");
    }

    private Config loadOrCreateConfig() throws IOException {
        Path path = configPath();
        if (!Files.exists(path)) {
            Config def = new Config();
            Files.createDirectories(path.getParent());
            try (Writer w = Files.newBufferedWriter(path)) {
                GSON.toJson(def, w);
            }
            throw new IOException("已建立預設設定檔 " + path + "，請填入 clientId / clientSecret / refreshToken 後重啟伺服器。");
        }
        try (Reader r = Files.newBufferedReader(path)) {
            Config c = GSON.fromJson(r, Config.class);
            return c != null ? c : new Config();
        }
    }

    private State loadState() {
        Path path = statePath();
        if (Files.exists(path)) {
            try (Reader r = Files.newBufferedReader(path)) {
                State s = GSON.fromJson(r, State.class);
                if (s != null) return s;
            } catch (IOException ignored) {
            }
        }
        return new State();
    }

    private void saveState() {
        try {
            Files.createDirectories(statePath().getParent());
            try (Writer w = Files.newBufferedWriter(statePath())) {
                GSON.toJson(state, w);
            }
        } catch (IOException e) {
            LOGGER.warning("[SubKill] 無法儲存狀態：" + e.getMessage());
        }
    }

    // ---------- 資料結構 ----------

    static class Config {
        String clientId = "your_client_id";
        String clientSecret = "your_client_secret";
        String refreshToken = "your_refresh_token";
        String killCommand = "kill @a";
        int pollIntervalSeconds = 30;
        long triggerCooldownMs = 5000;
    }

    static class State {
        boolean initialized = false;
        long highestCount = 0;
    }
}
