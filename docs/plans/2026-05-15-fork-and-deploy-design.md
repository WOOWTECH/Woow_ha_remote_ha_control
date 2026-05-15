# Fork remote_homeassistant 專案與測試站部署設計文件

**日期：** 2026-05-15
**專案：** Woow_ha_remote_ha_control

---

## 一、專案背景

### 上游專案

- **名稱：** `custom-components/remote_homeassistant`
- **GitHub：** https://github.com/custom-components/remote_homeassistant
- **版本：** v4.6（最新）
- **授權：** Apache-2.0
- **維護者：** Lukas Hetzenecker

### 功能概述

Remote Home-Assistant 是一個 Home Assistant 自訂整合元件，用於將多個 HA 實例透過 WebSocket API 串聯。主要功能：

- 主實例透過 WebSocket 連接遠端 HA 實例
- 遠端實體狀態同步至主實例
- 服務呼叫自動轉發回原始遠端實例
- 可依 domain 或 entity ID 篩選/排除實體
- 連線中斷時自動清除已發佈狀態
- 支援 UI Config Flow 及 YAML 設定
- 主實例和遠端實例**都需要安裝**此元件

### Fork 目標

WOOWTECH 團隊需要長期維護此元件，以便：
1. 根據公司需求客製化功能
2. 修復上游可能不再處理的 bug
3. 與公司內部 HA 生態系統整合

---

## 二、Fork 策略

### 採用方案：完整歷史 Fork

將上游完整 git 歷史（所有 commit、branch、tag）合併至 WOOWTECH 組織的 repo。

**優點：**
- 保留完整開發歷史，便於追蹤變更
- 可隨時 `git merge upstream/master` 同步上游更新
- 保留所有版本標籤（v1.0 ~ v4.6）用於回溯

### Git Remote 配置

| Remote | URL | 用途 |
|--------|-----|------|
| `origin` | `https://github.com/WOOWTECH/Woow_ha_remote_ha_control.git` | WOOWTECH 主 repo（推送目標） |
| `upstream` | `https://github.com/custom-components/remote_homeassistant.git` | 上游原始 repo（同步來源） |

### 分支策略

| 分支 | 用途 |
|------|------|
| `main` | 穩定版本，包含上游完整歷史 + WOOWTECH 自訂修改 |
| `vk/*` | 功能開發/問題修復工作分支 |
| `upstream/master` | 上游追蹤（僅讀取） |

### 同步上游流程

```bash
git fetch upstream
git checkout main
git merge upstream/master
git push origin main
```

---

## 三、專案檔案結構

```
Woow_ha_remote_ha_control/
├── .github/workflows/
│   ├── hassfest.yml          # HA 整合驗證 CI
│   └── validate.yml          # HACS 驗證 CI
├── custom_components/
│   └── remote_homeassistant/
│       ├── __init__.py       # 主模組（811 行，WebSocket 連線核心）
│       ├── config_flow.py    # UI 設定流程（451 行）
│       ├── const.py          # 常數定義
│       ├── manifest.json     # HA 整合宣告
│       ├── proxy_services.py # 服務代理轉發
│       ├── rest_api.py       # REST API 端點
│       ├── sensor.py         # 連線狀態感測器
│       ├── services.yaml     # 服務定義
│       ├── views.py          # HTTP 視圖
│       └── translations/     # 多國語言（en, de, pt-BR, sk）
├── icons/                    # 圖示資源
├── img/                      # 文件截圖
├── docs/plans/               # 設計文件（新增）
├── hacs.json                 # HACS 整合設定
├── LICENSE.md                # Apache-2.0 授權
└── README.md                 # 專案說明
```

---

## 四、測試站部署

### 容器資訊

| 項目 | 值 |
|------|------|
| 容器名稱 | `ha-remote` |
| 映像檔 | `ghcr.io/home-assistant/home-assistant:stable` |
| 端口映射 | `15131 -> 8123` |
| 存取網址 | `http://localhost:15131` |
| 管理員帳號 | `admin` |
| 管理員密碼 | `admin` |
| 設定目錄 | `/home/woowtech-ai-coder/ha-remote-config/` |
| 時區 | `Asia/Taipei` |
| 重啟策略 | `unless-stopped` |

### Podman 執行命令

```bash
podman run -d \
  --name ha-remote \
  --restart unless-stopped \
  -p 15131:8123 \
  -v /home/woowtech-ai-coder/ha-remote-config:/config:Z \
  -e TZ=Asia/Taipei \
  ghcr.io/home-assistant/home-assistant:stable
```

### Custom Component 掛載

設定目錄結構：
```
/home/woowtech-ai-coder/ha-remote-config/
├── custom_components/
│   └── remote_homeassistant/   ← Fork 的程式碼
├── configuration.yaml          ← HA 自動產生
└── ...                         ← HA 自動產生的其他設定檔
```

---

## 五、驗證結果

### Git Fork 狀態

- [x] 上游完整歷史已合併（所有 commit + 25 個 tag）
- [x] 已推送至 `origin/main`（WOOWTECH GitHub）
- [x] 已推送至 `origin/vk/8459-fork-https-githu`（工作分支）
- [x] upstream remote 已設定，可隨時同步

### 測試站狀態

- [x] Podman 容器 `ha-remote` 運行中（port 15131）
- [x] HA 成功啟動
- [x] Onboarding 完成（admin/admin）
- [x] `remote_homeassistant` custom component 已被 HA loader 偵測
- [x] HA 設定檢查通過（`config check: valid`）
- [x] API 正常運作（17 個實體）

### HA 日誌確認

```
WARNING [homeassistant.loader] We found a custom integration
remote_homeassistant which has not been tested by Home Assistant.
```

此 warning 為正常行為，表示 HA 已正確識別並載入此自訂元件。

---

## 六、後續維護指引

### 更新 Custom Component

當 fork repo 有程式碼更新時：

```bash
# 複製最新程式碼到測試站
cp -r custom_components/remote_homeassistant/* \
  /home/woowtech-ai-coder/ha-remote-config/custom_components/remote_homeassistant/

# 重啟容器
podman restart ha-remote
```

### 同步上游更新

```bash
git fetch upstream
git merge upstream/master
# 解決可能的衝突後
git push origin main
```

### 容器管理

```bash
podman stop ha-remote      # 停止
podman start ha-remote     # 啟動
podman restart ha-remote   # 重啟
podman logs ha-remote      # 查看日誌
podman logs -f ha-remote   # 即時追蹤日誌
```
