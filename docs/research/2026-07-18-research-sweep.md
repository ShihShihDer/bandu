# bandu 研究彙整：技術選型・安全篩查・法規・素材・部署模型（2026-07-18）

由 10 個研究 agent 的多維度網路查證彙整而成（6 維度＋審查補漏 3 項，共 267 次查證動作）。
各節含逐項來源連結與信心等級；「查證事實」與「推測」已分別標注。

## 執行摘要（十個決定性結論）

1. **通訊選型定案：自架 LiveKit（Apache-2.0）**——50 併發純語音對 SFU 負載不到單核；官方 egress 支援分軌錄音（志工/孩子各一軌），免轉碼、單機數百併發。Jitsi 因 Jibri「一場錄音一台 8GB VM」在全程存檔前提下出局。
2. **分軌錄音是一石三鳥的架構決定**——直接消滅說話人分離（diarization）問題、講者歸屬 100% 確定、且「脫離平台誘導」訊號主要在成人志工軌（辨識率最高的那軌）。
3. **STT 定案：Breeze-ASR-25（聯發科，繁中 WER 約 8%，Apache-2.0）＋ faster-whisper 夜間批次自架**——兒童語音錯誤率是最大未知（英語研究兒童 WER 可達 25%），上線前必須用 50–100 段真實錄音實測。逐字稿只當索引不當證據，可疑段落一律回聽原音。
4. **AI 篩查採四層管線：規則層（即時）→ LLM 場次評分（批次）→ 跨場次縱向分析 → 人工終審**——Roblox Sentinel（Apache-2.0 開源）是縱向分析的工業先例可直接參考。期望管理：英文回顧式分類 F1 0.99 不代表實戰，早期偵測 SOTA 僅 0.64，中文無公開 benchmark，須自建紅隊測試集。
5. **法規無紅線但有兩條法定時限**——兒少權法 §53 的 24 小時責任通報、性剝削條例 §8 的 180 日資料保留（覆蓋自訂刪除期）。工程解：R2 lifecycle 一般錄音 90 日自刪、命中案件搬 legal-hold 前綴。志工查核 §81 可能不適用一般社團，需函詢＋律師；家長同意書是第一份該付費請律師審的文件（法遵一次性成本估 NT$10,000–30,000）。
6. **部署模型翻案：據點集中上線為預設，在家為例外**——家庭端數據打臉在家模式（弱勢家庭 60.6% 無電腦/平板、家長監管大量落空）；而 1919 陪讀班（233 班）、博幼（17 中心）等課輔據點本來就開晚間時段，「志工下班後」與「據點開放時段」其實對得上。教育部數位學伴 2025 年底結束、2026 由 AI Di+ 承接，其原夥伴學校網絡是機構端切入真空期。
7. **志工供給是最大實驗變數**——台灣不存在「純無償＋固定週次」遠距伴讀先例（數位學伴每節付 549 元且限大學生）。50 對需首學期 130–150 名報名者；先跑 10–15 對實測轉換率與留存。法定 12 小時培訓大半有免費公版（臺北e大基礎 6hr＋衛福部特殊課綱），真正要自製的只有約 3 小時。
8. **錢不是約束、人才是**——全案雲端月成本 10 對約 NT$270–1,240、50 對悲觀不超過 NT$2,500。真正斷點：人工覆核悲觀參數下單人在 29–49 對爆掉，且 24 小時通報單人排班是結構性單點故障——上 50 對前必須有機構社工作第二責任人（寫進 MOU）。
9. **素材架構：索引不重製（link-not-host）**——平台只存書目與進度，內容雙端各開官方來源。共讀主力：圓夢繪本資料庫（逐本 CC）＋國資圖電子書；文化部兒童文化館**不可嵌入**只可連結。金融素養用金管會分齡教材、法治用司法院圖說司法、職探用新北職探 YouTube 官方嵌入。
10. **遊戲室 MVP 三件組**——自建 web shell（LiveKit 語音＋錄音）＋ Excalidraw 白板（MIT，自架關閉分享旁路）＋ 自製同步共讀器（PDF/EPUB 伺服器端預轉圖＋Yjs 同步翻頁指讀）。不採 Minecraft Education（不可嵌入、US$36/人/年、內建聊天是脫離平台破口）。

---

# 第一部：六維度研究

## 通訊技術選型（網路模式）：LiveKit 自架 vs Jitsi+Jibri vs mediasoup vs 商用（Daily/Agora）、低頻寬語音需求、VPS 成本

**結論**：LiveKit 自架（Apache-2.0）是此案最合適的選型：SFU 本體對 50 併發純語音的負載趨近於零（官方 16 核 benchmark 約可承載 3000 audio-only participants），且官方 egress 服務原生支援 audio_only 房間混音錄成 OGG、以及更省資源的 per-track 錄音，直接餵給 STT/LLM 篩查管線。Jitsi+Jibri 路線被「一場錄音 = 一台 8GB+ 專用機」的架構否決（全程存檔等於 25 場併發就要 25 台 VM）；mediasoup 是 library 不是 server，需自寫 signaling 與錄音層，開發量以週計；商用 Daily/Agora 語音費率其實不貴（滿載約 US$170/月、pilot 期免費額度可覆蓋），可當備援但把兒少語音資料交給第三方且長期成本高於自架。硬體上一台台灣本地 2C4G VPS（約 NT$990/月）或東京 Linode/Vultr（US$5–24/月）即可跑完整 stack，志工/孩子端每人約 50–100 kbps 就夠，台灣偏鄉 4G 人口涵蓋率政策目標已達村里 99%。

**建議**：推薦選型：自架 LiveKit（Apache-2.0）單節點 + 官方 egress，跑在一台 2C4G～4C8G VPS 上（台灣本地遠振 NT$990/月，或 Linode/Vultr 東京 US$12–24/月；預算月支出可壓在 NT$500–1,500 含儲存）。理由：(1) 50 併發純語音對 LiveKit SFU 的負載不到單核，硬體門檻是本案所有自架選項中最低；(2) 伺服器端錄音是本案的合規核心，LiveKit 是唯一「開源＋官方維護＋輕量音訊錄製（track egress 免轉碼、數百併發）」三者兼備的選項——Jitsi 的 Jibri 一場錄音要一台 8GB VM，在「每場必錄」前提下直接出局；mediasoup 要自寫 signaling 與錄音層，工程量不符一人公益專案；(3) 兒少語音全程存檔的資料主權敏感，自架讓錄音檔不經第三方 SaaS，只落在自己控制的 VPS 與 S3 相容儲存；(4) 內建 TURN/TLS 走 443，對學校/機構網路穿透友善；(5) 若未來要規模化或不想維運，同一套程式碼可無痛切到 LiveKit Cloud。具體建議：一對一房用 Track egress 分軌錄音（志工/孩子各一軌 OGG），下游 STT 免 diarization、講者歸屬 100% 確定，直接強化 LLM 篩查準確度；機構回放需求用離線 ffmpeg 合軌。備援方案：若初期完全沒有維運量能，可先用 Daily 音訊費率（$0.00099/min，月 1 萬分鐘免費，pilot 期近乎零成本）驗證產品，但上量後月費（滿載約 US$170+錄音另計）將高於自架，且錄音落地第三方需重新做兒少資料法遵評估——建議只當過渡，不當終點。

### 查證細項

- **LiveKit 開源授權與自架能力（查證事實）**（信心：high）
  LiveKit server 為 Apache License 2.0，可自架、無授權費/分鐘費/人數上限；自架與 LiveKit Cloud 使用同一套 SFU、SDK 與 API，之後要遷移雲端不用改應用程式碼。語音房（audio-only room）是核心支援場景。自架所需開放 port：7880（主要）、7881/TCP、UDP 50000–60000，並內建 TURN（TLS 5349 或 443），對學校/機構防火牆與嚴格 NAT 環境友善（TURN/TLS 流量外觀等同 HTTPS）。單節點小型部署可行，Redis 為 production 建議項（egress 則必須共用同一 Redis）。
  來源：<https://github.com/livekit/livekit>、<https://docs.livekit.io/transport/self-hosting/>、<https://docs.livekit.io/transport/self-hosting/deployment/>

- **LiveKit 伺服器端錄音（egress）能力（查證事實）**（信心：high）
  livekit/egress 亦為 Apache-2.0。三種模式皆與本案相關：(1) RoomComposite 支援 audio_only=true，把整房音訊混成單一 OGG 檔，且支援 dual-channel 模式（不同參與者分左右聲道）——對「語音轉逐字稿＋講者歸屬」極有價值；(2) Track egress 匯出單一參與者的原始音軌（WebRTC 音訊即 Opus，存為 OGG），官方文件明言不需轉碼、資源消耗極低，單一 egress instance 可跑數百個併發 TrackEgress job；(3) RoomComposite（含視訊時）每 job 吃 2–6 CPU，官方建議每個 egress instance 至少 4 CPU/4GB。輸出可直接上傳 S3 相容儲存/Azure Blob/GCS。部署需 Docker（--cap-add=SYS_ADMIN）＋與 LiveKit server 共用的 Redis。
  來源：<https://github.com/livekit/egress>、<https://docs.livekit.io/home/self-hosting/egress/>、<https://docs.livekit.io/transport/media/ingress-egress/egress/composite-recording/>

- **LiveKit 自架硬體需求（50 併發語音遠低於單機上限）**（信心：medium）
  官方 benchmark 以 16 核 compute-optimized 機器測試；第三方彙整指出 audio-only 場景 16 核約可承載 ~3,000 參與者（80% CPU）。以此推算（推測）：50 個同時在線的純語音使用者（約 25 個一對一房間）對 SFU 的 CPU 需求遠小於 1 核，2 vCPU/4GB 單機綽綽有餘，真正要留餘裕的是 egress 錄音程序。另一限制「單一房間必須落在單一節點」對一對一房型完全無影響。頻寬面（推測，依 Opus 位元率計算）：50 使用者 × 上下行各 ~40–50 kbps ≈ 伺服器總頻寬 4–5 Mbps、月流量約 50GB 等級，遠低於任何 VPS 流量上限。
  來源：<https://docs.livekit.io/transport/self-hosting/benchmark/>、<https://fazliev.com/blog/livekit-production-guide>、<https://celloip.com/blog/self-hosted-livekit-deployment-guide/>

- **Jitsi + Jibri 路線：錄音架構是致命傷（查證事實）**（信心：high）
  Jitsi Meet 本體對小型會議不重（官方手冊：小型可 4GB RAM、基本 4 核；Prosody 只能用單核）。但錄音元件 Jibri 的架構是「無頭 Chrome 進會議室再整場轉檔」：官方明言 one Jibri instance = one meeting、無 workaround，且 720p 錄製單場就要至少 8GB RAM、需獨立機器/VM。即使通話是純語音，Jibri 仍走視訊合成管線。套到本案「全程存檔」需求（推論）：25 場併發 = 25 台 8GB 專用 VM，維運與成本完全不可行；Jitsi 生態也沒有官方的輕量 per-track 音訊錄製方案。結論：Jitsi 適合「開會不錄」或「偶爾錄一場」，不適合「每場必錄」的合規型平台。
  來源：<https://jitsi.github.io/handbook/docs/devops-guide/devops-guide-requirements/>、<https://github.com/jitsi/jibri>、<https://community.jitsi.org/t/jibri-requirements/104021>

- **mediasoup 自建路線：開發量不符個人公益專案（查證事實＋推論）**（信心：high）
  mediasoup 是 SFU library 而非完整 server：無安裝器、無房間管理、無錄音服務，需自寫 signaling server、前端整合與 worker 進程管理（保活、重啟、房間路由），第三方比較文件一致評估 greenfield 專案到第一通電話以「週」計，錄音還要另外接 GStreamer/FFmpeg 管線。CPU 效率最高、彈性最大，但那是有專職後端團隊時的優勢。對一人出資出力的公益專案（推論）：把工程時數花在 SFU 底層是錯誤配置，應花在安全篩查與機構流程上。
  來源：<https://www.forasoft.com/learn/video-streaming/articles-streaming/sfu-comparison-mediasoup-janus-livekit-jitsi-pion>、<https://trembit.com/blog/choosing-the-right-sfu-janus-vs-mediasoup-vs-livekit-for-telemedicine-platforms/>、<https://mylinehub.com/articles/janus-vs-livekit-vs-mediasoup-webrtc-server-comparison>

- **商用選項價格：Daily / Agora（查證事實＋試算）**（信心：high）
  查證事實：Daily 每月 10,000 participant-minutes 免費，超出後預設（視訊）費率 $0.004/participant-min，純音訊費率 $0.00099/min（session 只要出現過任何 video track 就整段按視訊計價；音訊費率需確認套用）。Agora Voice Calling 每月 10,000 分鐘免費（與其他 RTC 產品共享額度），之後 $0.99/1,000 分鐘。試算（推測）：滿載情境 25 併發房 × 2 人 × 每晚 2 小時 × 30 天 ≈ 180,000 participant-min/月 → Daily 音訊費率約 US$170/月、Agora 約 US$168/月（皆未含雲端錄音另計費用）；pilot 期（如每週 10 場 × 1 小時）約 4,800 分鐘/月，落在兩家免費額度內。結論：價格不是商用路線的主要障礙，資料主權（兒少語音全程存檔放在第三方）與錄音附加費才是。
  來源：<https://www.daily.co/pricing/video-sdk/>、<https://docs.agora.io/en/voice-calling/overview/pricing>、<https://www.agora.io/en/pricing/>

- **純語音一對一的頻寬需求與偏鄉網路可行性（查證事實）**（信心：high）
  Opus 是 WebRTC 強制支援的主要語音編碼（RFC 7874），支援 6–510 kbps 並內建 VBR 自適應；寬頻語音（HD voice）16–24 kbps 已達良好品質，一般 VoIP 建議 24–32 kbps。推測（含 RTP/UDP 封包開銷）：每方向實際線路流量約 40–50 kbps，一對一經 SFU 每端上下行合計 ~100 kbps——即 0.1 Mbps，任何 4G 訊號、甚至弱訊號降到 16 kbps 模式都能維持通話。台灣基礎建設面：NCC 資料顯示五家業者 4G 人口涵蓋率 93.2%–99%，數位發展部並以「偏遠地區村里人口涵蓋率 99% 以上」為頻率使用費係數政策目標，持續改善 87 個偏遠鄉鎮 779 個村里。純語音（而非視訊）的產品決策同時是安全設計與低頻寬設計，兩者互相強化。
  來源：<https://wiki.xiph.org/Opus_Recommended_Settings>、<https://getstream.io/resources/projects/webrtc/advanced/codecs/>、<https://moda.gov.tw/major-policies/reinforce-rural-services/1307>、<https://www.ncc.gov.tw/chncc/app/data/list?id=382>

- **VPS/雲成本估算（50 併發以下）（查證事實＋估算）**（信心：medium）
  台灣本地：遠振資訊台灣 VPS 2 核/4GB/50GB SSD/月流量 2TB 約 NT$990/月（年繳送一個月），本地機房 ping 約 10–20ms；捕夢網同級約 NT$2,700/月。國際近鄰：Linode 東京 Nanode（1C/1GB/1TB 流量）US$5/月，DigitalOcean/Vultr 1C/1GB US$6/月，2C/4GB 級距約 US$12–24/月（台灣到東京 RTT 約 35–50ms，屬推測值，對語音無感）。Oracle Cloud Always Free ARM 已於 2026-06-15 無預警從 4 OCPU/24GB 砍到 2 OCPU/12GB（免費帳戶超額 instance 直接被關機），規格上仍夠跑本案但穩定性風險高，不建議承載正式服務。估算結論（推測）：LiveKit server + egress + Redis 同機跑在一台 2C4G（偏保守可 4C8G），月費 NT$300–1,000 區間即可覆蓋 50 併發純語音＋全程錄音，錄音檔另存 S3 相容儲存。
  來源：<https://host.com.tw/%E5%8F%B0%E7%81%A3VPS%E4%B8%BB%E6%A9%9F>、<https://www.nss.com.tw/virtual-hosting-charging>、<https://joyso.app/knowledgebase/21/>、<https://www.vultr.com/pricing/>、<https://kuberns.com/blogs/linode-pricing/>、<https://www.infoq.com/news/2026/07/oracle-cloud-free-tier-limits/>

- **錄音架構的加分設計：per-track 錄音對 LLM 篩查管線的價值（推測，基於已查證能力）**（信心：medium）
  這是架構建議而非查證事實：與其用 audio_only RoomComposite 混音（每房一個 Chrome 合成程序，資源成本待實測；社群存在 nostrnests/livekit-audio-egress 這類『輕量 audio-only egress』替代品，側面暗示 stock RoomComposite 對純音訊偏重），不如對一對一房型直接用 Track egress 分別錄下志工與孩子兩條 OGG/Opus 音軌。官方文件證實 track egress 不需轉碼、數百併發 job 單機可承載；而分軌錄音讓下游 STT 天然免除 speaker diarization 問題——逐字稿的每一句話講者身分 100% 確定，這對「抓志工誘導脫離平台」的 LLM 篩查是準確度的直接提升。若機構覆核需要「整場回放」，再離線用 ffmpeg 把兩軌合併即可（批次、非即時，成本趨近零）。
  來源：<https://docs.livekit.io/home/self-hosting/egress/>、<https://github.com/livekit/egress>、<https://github.com/nostrnests/livekit-audio-egress>

### 未解問題

- LiveKit audio_only RoomComposite 是否仍為每房啟動一個 headless Chrome、單房實際 CPU/RAM 成本多少？（社群輕量替代品的存在暗示 stock 版偏重）需要 PoC 實測 25 併發錄音在 2C4G 機器上的表現。

- 官方 16 核 ≈ 3,000 audio-only participants 的數字來自第三方對官方 benchmark 的轉述，未在官方頁面逐字驗證；建議用 LiveKit 官方 load-testing 工具在目標 VPS 上實壓 50 併發。

- Daily 的雲端錄音（recording）為另計費項，音訊錄音費率未查證；且『session 出現任何 video track 即整段按視訊計價』的細節會影響備援方案成本。

- Agora 的資料落地位置與供應鏈背景（中國研發主體）對兒少語音資料的法遵/信任影響未查證——若考慮 Agora 需先做這項盡職調查。

- 台灣本地 VPS（遠振/捕夢網）對 UDP 大 port range（50000–60000）是否有限制、跨電信（中華/台哥大/遠傳）peering 品質如何，需實測；若不佳則東京節點反而可能更穩。

- 錄音檔長期儲存成本（S3/Cloudflare R2/Backblaze B2 的單價與台灣存取延遲）不在本維度查證範圍，需另行估算——全程存檔的月增量約為每 participant-hour 10–20MB（Opus 32kbps 估算，屬推測）。

- 偏鄉端實際『家用網路品質』（非涵蓋率）：4G 涵蓋 99% 是政策數字，個別家庭室內訊號、晚間壅塞時段的丟包率需在 pilot 中以 WebRTC stats 實測。

## 語音轉文字（逐字稿管線）

**結論**：Whisper 系（尤其聯發科 Breeze-ASR-25，台灣華語 zh-TW WER 約 8%、Apache 2.0）已足以支撐 bandu 的逐字稿需求，但有三個必須設計繞開的弱點：繁簡輸出不穩、靜音誘發幻覺、兒童語音錯誤率顯著高於成人（英語研究成人 3% vs 兒童 25% WER；3-5 歲中文兒童最佳微調後 CER 仍 27%）。台語與國台語夾雜已有開源解（Breeze-ASR-26 台語模型、數產署 Taiwan Tongues ASR），不必依賴商用服務。成本上商用雲端批次約 $0.24-0.36/音檔小時，在 50 同時在線的量級下每月可達數百美元，而自架 faster-whisper 在單張消費級 GPU 上可達數十倍即時速度、每音檔小時成本可壓到 $0.01-0.06，極低預算下自架是明確方向；說話人分離問題可用「自建遊戲室每人分軌錄音」在架構層直接消滅，不需要 diarization 模型。

**建議**：建議 bandu 的逐字稿管線定案為：「分軌錄音 → 夜間自架批次 faster-whisper（Breeze-ASR-25）→ 繁體正規化 → segment 級時間戳交 LLM 篩查與人工覆核」。具體理由與作法：(1) 遊戲室若採 LiveKit，用 Track Egress 把志工與孩子的音軌分開存檔——這一個架構決定同時解掉 diarization（不需 pyannote）、說話人歸屬（軌道即身分）、與兒童語音弱點的大半衝擊（「脫離平台誘導」訊號主要在成人志工軌，該軌 CER 最低）。(2) ASR 用 Breeze-ASR-25（Apache 2.0、繁體、zh-TW WER 約 8%）跑 faster-whisper + Silero VAD 濾靜音（壓幻覺）+ initial_prompt 與 OpenCC s2twp 雙保險鎖繁體；台語需求實際出現後再掛 Breeze-ASR-26 第二遍跑台語軌。(3) 成本：伴讀是異步篩查場景不需即時轉錄，夜間批次在自有桌機 GPU 邊際成本趨近零，或租 RunPod Secure Cloud 約 $20-180/月即可吃下尖峰假設；同量級走 Google/Azure 批次要 $500-2,000+/月，個人出資撐不住——商用雲端只留作 <100 音檔小時/月的試營運期或故障備援（Google Dynamic Batch $0.24/音檔小時最便宜）。(4) 制度護欄：因 Whisper 幻覺率約 1% 且靜音誘發，人工覆核 SOP 必須規定「可疑段落一律回聽原音檔」，逐字稿只當索引不當證據；這也決定了時間戳只需 segment 級，不必為 WhisperX word 級對齊增加中文對齊模型的維護負擔。(5) 上線前先做 50-100 段真實伴讀錄音的 pilot，人工標註算兒童軌與志工軌的實際 CER，作為 LLM 篩查召回率的地板數據——兒童語音的實際表現是目前最大未知，不宜跳過。

### 查證細項

- **(a) Whisper/faster-whisper 中文準確度**（信心：high）
  查證事實：Whisper large-v3 在 Common Voice 中文（普通話）約 12.8% CER；中日韓因斷詞模糊慣用 CER 而非 WER。針對台灣華語，Common Voice 16 zh-TW 測試集上 Whisper-large-v2 基線約 9.8%，聯發科微調後降到 7.97%（見 (b)）。faster-whisper 是 CTranslate2 重寫，官方宣稱與原版 Whisper 同準確度、GPU 上約 4 倍速、CPU 上 2 倍速，支援 INT8/FP16 量化與批次推論。推測：對「志工念題目、講解功課」這類接近朗讀/教學的成人語音，實際 CER 會優於 Common Voice 均值；口語化、背景噪音（偏鄉家中環境）會往上拉。
  來源：<https://vexascribe.com/how-accurate-is-whisper>、<https://github.com/SYSTRAN/faster-whisper>、<https://huggingface.co/MediaTek-Research/Breeze-ASR-25>

- **(a) 繁體輸出不穩定問題與解法**（信心：high）
  查證事實：Whisper 只有單一語言碼 zh、訓練語料簡繁混雜，因此同一段音檔可能輸出簡體或繁體，語言參數無法控制。社群成熟解法有二：(1) initial_prompt 引導（如「以下是普通話的句子，請以繁體輸出」）；(2) 後處理用 OpenCC 或 zhconv 統一轉繁（OpenCC 的 s2twp 模式可同時做台灣用語轉換）。兩者並用最穩。faster-whisper 同樣支援 initial_prompt。這對 bandu 重要：逐字稿要給機構人工覆核，簡繁跳動會干擾閱讀與關鍵詞篩查（例如「賴」vs「赖」會讓字面比對漏抓）。
  來源：<https://github.com/openai/whisper/discussions/277>、<https://github.com/guillaumekln/faster-whisper/issues/521>、<https://docs.macwhisper.com/article/9-chinese-simplied-and-traditional-troubleshooting>

- **(a) 兒童語音是已證實的系統性弱點**（信心：high）
  查證事實：Whisper 對成人朗讀乾淨語音 WER 約 3%，同條件兒童語音約 25%，落差達 22 個百分點（英語研究）；Kid-Whisper 等研究確認兒童 ASR 是低資源任務，成因是兒童的韻律、發音、語句結構與成人語料差異大。中文方面，ChildMandarin 資料集（3-5 歲、41.25 小時）顯示零樣本表現差，微調後最佳模型（Conformer CTC-AED）CER 也僅到 27.38%，且微調後 Whisper-medium 反而不如 Whisper-small（小資料集過擬合）。微調可將兒童語音錯誤率降 20%-96%。推測（重要推論）：bandu 服務對象多為國小以上，語音成熟度高於 3-5 歲研究對象，實際落差會小於上述數字；且「脫離平台誘導」（加 LINE、私下聊、別跟爸媽說）這類第一優先篩查訊號主要出自志工（成人）之口，成人軌辨識率高，兒童 ASR 弱點對核心安全目標的衝擊有限——但兒童揭露受害或求助的語句仍依賴兒童軌品質，不能忽視。
  來源：<https://the-learning-agency.com/the-cutting-ed/article/how-speech-recognition-systems-struggle-with-childrens-voices/>、<https://ar5iv.labs.arxiv.org/html/2309.07927>、<https://arxiv.org/abs/2409.18584>

- **(a) 幻覺（hallucination）對安全篩查的雙向風險**（信心：high）
  查證事實：Careless Whisper 研究（2024）發現約 1% 的轉錄含整句無中生有的幻覺，其中 38% 含有害內容（暴力、錯誤指涉等）；靜音與長停頓是主要誘因，對停頓較多的說話者（語言障礙、兒童）更嚴重；幻覺常表現為同句重複迴圈。對 bandu 的意涵：伴讀情境有大量安靜寫作業時段，未處理的靜音會誘發幻覺，可能產生「假可疑訊號」（幻覺句被 LLM 篩查誤判）或污染證據鏈。緩解：faster-whisper 內建 VAD（Silero）先濾靜音、Calm-Whisper 等研究方向、以及制度設計上人工覆核必須回聽原音檔而非只看逐字稿。
  來源：<https://arxiv.org/html/2402.08021v1>、<https://arxiv.org/html/2505.12969v1>、<https://github.com/SYSTRAN/faster-whisper>

- **(b) 台灣華語首選：聯發科 Breeze-ASR-25**（信心：high）
  查證事實：Breeze-ASR-25 由 Whisper-large-v2 微調，針對台灣華語與中英夾雜優化，Apache 2.0 授權、繁體輸出。官方 benchmark：CommonVoice16-zh-TW WER 7.97%（比基線好 19%）、中英 code-switching 測試集 CSZS-zh-en 13.01%（好 56%）、長音檔課程 ML-lecture 4.98%。模型 2B 參數（large 級），與 Whisper 生態相容，可轉 CTranslate2 格式跑 faster-whisper。注意：它不支援台語；中文訓練資料全為合成語音。推測：這是 bandu 華語軌的最佳起點——免費、繁體、台灣口音優化、授權乾淨。
  來源：<https://huggingface.co/MediaTek-Research/Breeze-ASR-25>、<https://github.com/mtkresearch/Breeze-ASR-25>、<https://www.mediatek.com/zh-tw/press-room/mediatek-research-unveils-mr-breeze-asr-25-an-open-source-ai-model-for-taiwanese-speech>

- **(b) 台語與國台語夾雜的選項**（信心：high）
  查證事實：(1) 聯發科 Breeze-ASR-26（BreezeASR-Taigi，2026 開源）：首個開源台語 ASR，Whisper-large-v2 基底、約 1 萬小時合成台語語料（含台華夾雜），輸出華語漢字而非台文正字，Taigi benchmark 平均 CER 30.13%（樣本間 14.49%-52.78% 波動大），Apache 2.0；官方承認強地方腔與專有名詞會退化、全合成語料是限制。(2) 數位發展部數位產業署 Taiwan Tongues ASR CE（台灣大哥大維運）：MIT 授權，支援國語/台語/客語/英語/印尼語混講，直接提供 CTranslate2 格式（即 faster-whisper 可直接載入）與微調腳本，但無公開 benchmark 數字。(3) 意傳科技（ithuan.tw）：台語專業公司，提供商用客製 ASR/MT/語料標注，價格未公開。(4) 學界：NUTN（台南大學）Whisper-Taiwanese-model-v0.5、CLiFT-ASR 跨語言微調框架（arXiv 2511.06860）。推測：偏鄉伴讀場景孩子可能國台語夾雜，但主要互動語言仍是華語；台語軌可列為第二階段，屆時 Breeze-ASR-26 輸出華語漢字反而利於 LLM 篩查（不需懂台文正字）。
  來源：<https://huggingface.co/MediaTek-Research/Breeze-ASR-26>、<https://github.com/adi-gov-tw/Taiwan-Tongues-ASR-CE>、<https://www.koc.com.tw/archives/639154>、<https://ithuan.tw/>、<https://huggingface.co/NUTN-KWS/Whisper-Taiwanese-model-v0.5>、<https://arxiv.org/pdf/2511.06860>

- **(c) 商用雲端價格：Google STT 與 Azure Speech**（信心：medium）
  查證事實（第三方 2025-10 查核 + 微軟官方 Q&A 交叉比對；兩家官方定價頁皆為 JS 渲染無法直接抓取）：Google STT V2 標準即時 $0.016/分鐘 = 約 $0.96/音檔小時（含 Chirp 模型）；Dynamic Batch 折 75% 約 $0.004/分鐘 = $0.24/音檔小時（24 小時回件）；每月 60 分鐘免費；以 15 秒為計費單位無條件進位；官方文件確認支援 cmn-Hant-TW（chirp/chirp_2/chirp_3）。Azure Speech：標準即時 $1.00/小時，批次轉錄 $0.36/小時（微軟官方 Q&A 亦引 $0.36/音檔小時），diarization 即時要加 $0.30/小時、批次宣稱內含（但同一官方回答內部有矛盾，見 open questions）；每月 5 音檔小時免費；支援 zh-TW。另有搜尋結果稱 Azure 批次 $0.18/小時，與多數來源矛盾，採 $0.36 為準。推測：以 bandu 尖峰假設（50 組同時在線、每晚 2-3 小時、志工+孩子分軌錄音使音檔時數翻倍），每月可達 2,000-6,000 音檔小時，Google batch 約 $480-1,440/月、Azure batch 約 $720-2,160/月——對個人出資公益專案顯然過重；商用雲端只適合當低量試營運或溢流備援。
  來源：<https://brasstranscripts.com/blog/google-cloud-speech-to-text-pricing-2025-gcp-integration-costs>、<https://brasstranscripts.com/blog/azure-speech-services-pricing-2025-microsoft-ecosystem-costs>、<https://learn.microsoft.com/en-us/answers/questions/5634306/how-much-is-the-pricing-of-using-the-diarization-a>、<https://docs.cloud.google.com/speech-to-text/docs/speech-to-text-supported-languages>

- **(d) 自架 GPU 批次成本：每音檔小時 $0.01-0.06，比雲端低 1-2 個數量級**（信心：medium）
  查證事實：RTX 4090 上優化實作（batched faster-whisper / insanely-fast-whisper）可達 70-100 倍即時速度；faster-whisper 單流也有原版 4 倍速（原版 large 在消費 GPU 上約 5-15 倍即時）。GPU 租價（2026）：Vast.ai 4090 $0.29-0.59/hr、RunPod $0.34（社群雲）-0.59（安全雲）/hr。推算（推測，標明假設）：以保守 20 倍即時計，1 GPU 小時處理 20 音檔小時，租用成本 $0.015-0.03/音檔小時；以 70 倍計則低於 $0.01。尖峰月 6,000 音檔小時只需 60-300 GPU 小時 ≈ $20-180/月；若跑在自有桌機 GPU（夜間批次），邊際成本僅電費（4090 約 350W，每音檔小時約 0.005-0.018 kWh，台電費率下不到 NT$0.1），趨近於零。注意：兒童錄音上租用市集 GPU（Vast.ai 社群主機）有資料保護疑慮，音檔屬敏感個資，自有硬體或有合約保障的雲比較站得住腳——這點是安全維度的推論。
  來源：<https://gigagpu.com/best-gpu-for-whisper/>、<https://github.com/SYSTRAN/faster-whisper>、<https://www.synpixcloud.com/blog/vast-ai-vs-runpod-rtx-4090-pricing>、<https://www.runpod.io/gpu-models/rtx-4090>

- **(e) 時間戳與說話人分離：架構層分軌錄音可直接消滅 diarization 需求**（信心：high）
  查證事實：(1) Whisper/faster-whisper 原生提供 segment 級時間戳（誤差可達 ±0.5 秒）；WhisperX 用 wav2vec2 強制對齊做到 word 級 ±50ms，但預設 torchaudio 對齊模型只有 en/fr/de/es/it，中文須用 HF 上的 wav2vec2 中文模型且需自行驗證品質。(2) 事後 diarization 標準解是 pyannote（speaker-diarization-3.1 / community-1）：MIT 授權但 HF gated（需帳號接受條款），可商用，維護者主推付費版 pyannoteAI。(3) 關鍵替代：bandu 的遊戲室是自建的，若用 LiveKit 等 WebRTC SFU，Track Egress 可把每位參與者的音軌分開錄成獨立檔案（官方明示適用「audit trail + 乾淨音源餵模型」場景），說話人身分由音軌天然決定，完全不需要 diarization 模型，也免除重疊語音、兒童聲紋區分不佳等 diarization 已知難題。推測（強烈推論）：1 對 1 或 1 對 2 伴讀 + 分軌錄音下,「志工軌」逐字稿是篩查主標的，segment 級時間戳已足夠支撐人工覆核跳轉回聽，WhisperX word 級對齊屬過度工程。
  來源：<https://github.com/m-bain/whisperX>、<https://huggingface.co/pyannote/speaker-diarization-3.1>、<https://docs.livekit.io/transport/media/ingress-egress/egress/>、<https://github.com/livekit/egress>

- **（補充）陸系替代模型 SenseVoice/FunASR：CER 亮眼但來源偏頗且不合台灣場景**（信心：low）
  查證事實：FunASR 官方 blog 宣稱 SenseVoice 在 184 段普通話測試 CER 8.0%、CPU 上 20 倍即時，遠優於其測得的 Whisper 系 22-31%——但此為 FunASR 自家評測（Whisper 對照組含 small/base 小模型、CPU 環境），數字偏頗程度未知，且未測台灣華語。推測：SenseVoice/Paraformer 以簡體與大陸口音語料為主，輸出簡體、對台灣用語與繁體場景需額外處理，加上 bandu 的信任敘事（給家長與機構交代資料流）用陸系模型有觀感成本；有 Breeze 系列在，不建議採用，僅列為知悉。
  來源：<https://www.funasr.com/en/blog/funasr-vs-faster-whisper-chinese.html>

### 未解問題

- Breeze-ASR-25/26 對兒童語音（國小年齡層台灣華語）的實際 CER 沒有任何公開數據，必須 pilot 實測；ChildMandarin 研究對象是 3-5 歲，外推有限

- Breeze-ASR-26 訓練語料全為合成台語，真實偏鄉兒童台語（含腔調）表現未知；其輸出為華語漢字，孩子台語原話的語義保真度對篩查召回率的影響需驗證

- Azure 批次轉錄的 diarization 是否真的免費：微軟官方 Q&A 同一回答內既說「batch 內含」又列 $0.30/hr 加購，且批次基價有 $0.18 與 $0.36 兩種說法，正式採用前需用 Azure 定價計算機確認（官方定價頁無法直接抓取）

- Google Dynamic Batch（$0.004/min）是否完整支援 cmn-Hant-TW 的 chirp_2/chirp_3 組合與可用區域未逐項查證

- WhisperX 的中文 word 級對齊實際精度未驗證（DEFAULT_ALIGN_MODELS_HF 中的中文 wav2vec2 模型品質需實測）——但若接受 segment 級時間戳此問題可迴避

- 租用市集 GPU（Vast.ai 社群主機）處理兒童語音檔的法遵與資安是否可接受（個資法、機構端要求），屬安全/法遵維度需另行研究

- 意傳科技商用台語 ASR 的授權價格未公開，若走商用客製需直接洽詢

- 分軌錄音的儲存成本與保存年限（全程語音存檔的量體）未在本維度估算，需併入整體成本維度

## AI 兒少安全篩查（誘騙偵測研究現況、審查 API 中文適用性、規則+LLM 混合偵測、跨場次縱向分析、誤報管理與人工覆核）

**結論**：學術上誘騙偵測在回顧式分類已近飽和（PAN12 上 LLM 可達 F1 0.99），但「即時早期偵測」效果驟降（latency-F1 僅 0.365）且全部研究基於英文誘捕資料，無中文資料集，直接遷移到 bandu 的中文語音轉錄場景不可行。現成審查 API 各有致命限制：Perspective API 將於 2026 年底落日；OpenAI moderation 免費且中文已改善，但其 13 類別不涵蓋「脫離平台」這類表面無害的訊號，也不吃音訊。業界（Roblox Sentinel、Modulate ToxMod）已驗證 bandu 提案的正確方向——規則過濾+跨場次縱向模式分析+人工覆核分流，且 Sentinel 以 Apache-2.0 開源可直接參考，關鍵是把 AI 定位為「高召回率的分流器」而非裁判，兒少安全案件一律人工終審。

**建議**：建議 bandu 採四層管線，全部圍繞「AI 是高召回分流器、人是終審」設計：(1) 規則層（即時）——自訂中文關鍵詞/正則抓 LINE ID、電話數字串、「加賴/私下聊/秘密/別跟爸媽說」句式及其諧音變體，命中即進人工佇列，這層零成本、零延遲、可解釋；(2) LLM 層（場次後批次）——每場逐字稿用一次 LLM 呼叫做 prompt-based 多維風險評分（脫離平台意圖、隔離話術、過度私密提問、性暗示），以 bandu 規模成本極低，不要用 Perspective（2026 年底落日），OpenAI moderation 免費可加掛但只當輔助（其類別抓不到無毒的誘騙訊號）；(3) 縱向層——把每場評分依「志工×孩子配對」跨場次聚合，參考 Apache-2.0 開源的 Roblox Sentinel 的 skewness 統計思路看分布尾端與軌跡（親密度爬升、話題漂移），這是抓「慢速養成型」誘騙的關鍵，也是 ToxMod/Sentinel 已驗證的工業先例；(4) 人工層——依嚴重度排序的覆核佇列，兒少安全案件絕不自動處置，覆核判定回存當 golden set。務必管理期望：文獻上的 F1 0.99 是英文回顧式分類的數字，早期偵測 SOTA 僅 F1 0.64，中文無任何公開 benchmark——上線前應自建中文紅隊對話測試集實測，並在對機構的溝通中把系統定位為「輔助人工的預警網」而非「AI 保證安全」。

### 查證細項

- **(a) PAN12 與回顧式誘騙偵測：技術上已成熟，但資料集有根本侷限**（信心：high）
  查證到的事實：PAN12（2012 年 PAN Sexual Predator Identification 競賽）是該領域標準資料集，誘騙對話來自 Perverted Justice 基金會（成人志工假扮兒童的誘捕紀錄），2012 年冠軍系統 F1 約 0.873；近年方法大幅提升——RoBERTa 訊息級聚合達 F1 0.96，ensemble 達 F0.5 0.9904，2025 年 Frontiers in Pediatrics 研究顯示 LLaMA 3.2 1B 微調後對誘騙者作者辨識達 F1 0.99（SVM baseline 0.94）。該論文作者自承三大限制：資料集老舊（誘騙話術會演化）、僅英文、且來自執法誘捕而非真實平台對話，外推性存疑。推測：對 bandu 而言，「PAN12 上 0.99」不能當作可達成的實戰指標；真實場景是中文、語音轉錄文字、極低基率（絕大多數志工無惡意），實際 precision 會遠低於論文數字。
  來源：<https://www.researchgate.net/publication/233324713_Overview_of_the_International_Sexual_Predator_Identification_Competition_at_PAN-2012>、<https://www.frontiersin.org/journals/pediatrics/articles/10.3389/fped.2025.1591828/full>、<https://www.researchgate.net/publication/341920836_Ensemble_Method_for_Sexual_Predators_Identification_in_Online_Chats>、<https://www.sciencedirect.com/science/article/pii/S0950705122011327>

- **(a) 早期偵測（eSPD）與 LLM 直接用於誘騙防範：效果顯著較差**（信心：high）
  查證到的事實：回顧式分類與「儘早偵測」是兩個難度級別。2025 年 SCoRL 論文（arXiv 2503.06627，RoBERTa+強化學習做 turn-level 早期偵測）雖然是新 SOTA，也只達 latency-F1 0.365、F1 0.641、precision 0.475——比前 SOTA（latency-F1 0.062）好 4 倍以上但絕對值仍低，且 turn-level 標註訓練資料僅 77 段對話；作者明言「太早警報準確度低、等待則喪失及早性」的根本張力。另一方面，2024 年 EICC 研究測了 6,000+ 次 LLM 互動後結論是「沒有任何模型明確適合用於 online grooming prevention」，行為不一致、開源模型甚至可能產生有害回答。推測：bandu 應把偵測設計成「場次後批次分析為主、場內即時只抓高置信度規則命中」，不要承諾即時攔截能力。
  來源：<https://arxiv.org/html/2503.06627v1>、<https://dl.acm.org/doi/10.1145/3655693.3655694>

- **(a/b) 中文誘騙偵測資料集：查無公開資源**（信心：medium）
  查證到的事實：多輪搜尋未找到任何公開的中文（普通話/台灣國語）誘騙偵測資料集或專門研究；非英文工作僅見韓文（KcELECTRA 框架）。survey 文獻指出誘騙對話本質上難以取得（一對一私訊不公開、受害者保護）。這是「查無」的負面發現，信心標 medium。推測：bandu 若要評估自己的偵測管線，必須自建中文測試集——可行做法是用 LLM 依 grooming 階段理論（如 Luring Communication Theory）合成紅隊對話+人工改寫，並混入真實無害的伴讀對話當負樣本。
  來源：<https://www.mdpi.com/2076-3417/15/13/7105>、<https://www.sciencedirect.com/science/article/pii/S0950705122011327>

- **(b) Google Perspective API：2026 年底落日，不應採用**（信心：high）
  查證到的事實：Perspective API 雖支援簡體中文（TOXICITY 等屬性、18+ 語言），但 Jigsaw 已宣布落日——服務只到 2026 年底，新申請與加量請求只受理到 2026 年 2 月，且 Google 明言不提供遷移協助。多個第三方來源交叉證實（官方支援頁面本身載入失敗，未能直接引用原文）。推測：對 2026 年 7 月才在構想階段的 bandu 而言，Perspective 已完全出局。
  來源：<https://medium.com/tisanelabs/goodbye-perspective-api-79da0f237b3f>、<https://www.lassomoderation.com/blog/what-is-perspective-api/>、<https://support.perspectiveapi.com/s/about-the-api-faqs?language=en_US>

- **(b) OpenAI Moderation API：免費、中文已改善，但類別設計天生抓不到「脫離平台」訊號，且不吃音訊**（信心：high）
  查證到的事實：官方文件確認 moderation endpoint 免費，omni-moderation-latest 偵測 13 類別（harassment、hate、illicit、self-harm、sexual、sexual/minors、violence 等），接受文字與圖片但「不分類音訊」（It doesn't classify audio）。多語言方面，OpenAI 宣稱新模型在 40 語言內部評測提升 42%、98% 語言有改善，第三方引述其中文表現已超過舊模型的英文表現（官方頁面回 403，此點僅二手來源，信心略降）；但獨立研究（arXiv 2410.22153）仍認定現有 guardrails「處理多語言毒性內容仍然無效」且易被規避。推測（重要架構含義）：bandu 第一優先要抓的「加LINE／私下聊／別跟爸媽說」在語義上完全無毒、不落入任何 moderation 類別，通用審查 API 對此必然漏報——moderation API 只能當輔助層（抓性暗示/騷擾語言），「脫離平台」偵測必須靠自訂規則+自訂 LLM prompt 分類。
  來源：<https://developers.openai.com/api/docs/guides/moderation>、<https://openai.com/index/upgrading-the-moderation-api-with-our-new-multimodal-moderation-model/>、<https://arxiv.org/abs/2410.22153>、<https://aimoderationtools.com/posts/openai-moderation-api-review/>

- **(c) 規則+LLM 混合（cascade）是 2026 年業界標配，成本可壓到極低**（信心：medium）
  查證到的事實：業界共識架構是分層 cascade——regex/關鍵詞規則層（CPU、<1ms）先擋明顯案例並過濾大量安全內容，只把可疑或不確定的訊息升級給 LLM 深度分析；有產線案例宣稱 97.5% 安全內容走輕量層、僅 2.5% 進 frontier LLM，推論成本降到全 LLM 方案的約 1.5%（單一部落格來源，具體數字信心低，但「cascade 省一到兩個數量級」的方向被多來源支持）。Roblox 實務也用「嚴格過濾器擋 handle、電話號碼、站外邀請」+行為偵測的組合。推測：以 bandu 規模（≤50 同時在線、一場次一小時、轉錄文字量小），即使「每場次全文丟一次便宜 LLM 批次分析」的成本也極低（每月估計在數美元到數十美元級，此為推算非查證），規則層的價值不在省錢而在零延遲抓高置信度訊號（LINE ID 格式、電話數字串、「別跟爸媽說」句式）與可解釋性。
  來源：<https://ethora.com/blog/ai-content-moderation/>、<https://tianpan.co/blog/2026-04-12-llm-content-moderation-at-scale>、<https://getstream.io/blog/nlp-vs-llm-moderation/>、<https://www.aicerts.ai/news/roblox-ai-safety-system-launches-to-protect-kids/>

- **(d) 跨場次縱向分析有直接產品先例：Roblox Sentinel（開源 Apache-2.0）與 Modulate ToxMod Player Risk Categories**（信心：high）
  查證到的事實：(1) Roblox Sentinel 專為「兒少危害早期訊號（如誘騙）」設計，核心方法是收集單一使用者跨時間的訊息、用 embedding 相似度算個別分數、再以統計偏度（skewness）聚合——因為「壞人會把意圖混在大量無害內容中」，看分布尾端而非平均值，並校正發言量以降誤報；2024 年底上線，2025 上半年協助提交約 1,200 件 NCMEC 通報、佔偵測案件 35% 為主動發現；已在 GitHub 開源（Roblox/Sentinel，Apache-2.0，Python library，主打極稀有類別的即時偵測）。(2) Modulate ToxMod 的 child grooming Player Risk Category 明確「分析長期行為模式」，可在尚未出現露骨內容前就因「對未成年者持續提出可疑暗示性問題」而標記；但未公開精度指標與技術細節。(3) 學術端的縱向分析僅限單場對話內的 turn-level 早期偵測（eSPD），跨場次「關係軌跡」分析在學術文獻中未見成熟工作——先例主要在工業界。推測：Sentinel 的 skewness-over-time 思路可直接移植到 bandu 的「志工×孩子配對」層級評分（每場次 LLM 給多維風險分，跨場次聚合看軌跡），但其 embedding 模型對中文的效果未經驗證，需替換成多語言 embedding 並自行校準。
  來源：<https://about.roblox.com/newsroom/2025/08/open-sourcing-roblox-sentinel-preemptive-risk-detection>、<https://github.com/Roblox/Sentinel>、<https://github.com/Roblox/Sentinel/blob/main/LICENSE>、<https://www.modulate.ai/press-releases/toxmod-player-risk-categories>、<https://ir.roblox.com/news/news-details/2025/Roblox-Releases-Early-Warning-System-to-Help-Keep-Children-Safer-Online/default.aspx>

- **(d/補充) 語音層直接偵測：有開源模型但不適合當 bandu 主力**（信心：high）
  查證到的事實：Roblox 另開源 voice-safety-classifier（HuggingFace，CC-BY-SA-3.0，94.6M 參數，WavLM 微調，直接吃音訊），六類別（Profanity/DatingAndSexting/Racist/Bullying/Other/NoViolation），毒性二元化 average precision 94.48%，訓練於 2,374 小時 Roblox 語音；未說明語言支援（推測以英文為主）。ToxMod 也是語音原生（號稱訓練於 1,000 萬+小時音訊、聽語氣與互動動態而非只靠轉錄），但是商用方案、未查到價格。推測：這些模型的類別設計（罵髒話/約會性暗示）與訓練語料（英文遊戲語音）都不對準 bandu 的核心威脅（中文、表面友善的關係培養），bandu 以 ASR 轉錄+文字分析為主軸是正確選擇；語音存檔的價值在人工覆核時聽語氣佐證，而非機器直接分類音訊。
  來源：<https://huggingface.co/Roblox/voice-safety-classifier>、<https://www.modulate.ai/products/toxmod>、<https://gamesbeat.com/modulate-raises-30m-to-detoxify-voice-chat-in-games-with-ai/>

- **(e) 誤報管理與人工覆核分流的業界做法**（信心：medium）
  查證到的事實：業界標準管線為「自動分流→依嚴重度排序的人工審查佇列→最嚴重類別（兒少安全、可信威脅）走專用升級程序且絕不由自動化單獨處理，常含法定通報」；不同政策領域應設不同閾值（依誤報 vs 漏報的相對成本），並以 golden dataset 持續評測、量測 inter-rater reliability、用申訴率/翻案率偵測政策模糊處。Roblox Sentinel 的每個標記案件都由受訓分析師（多為前 CIA/FBI 背景）覆核，覆核結果回饋更新模型範例，形成閉環。LLM 審查的已知風險包括：automation bias（審查員因信任 AI 而怠惰）、model drift、低資源語言的雙向誤差（過度與不足執法並存）。推測：對 bandu 的含義——(1) 兒少安全類 AI 只做排序與召回，終審必須是機構人員；(2) 閾值策略應「規則層高召回、寧可多送人工」，因為 50 人規模下每日場次少、人工覆核量完全可負擔，這是小規模的結構性優勢；(3) 從第一天就要留存覆核判定（accepted/false-positive）當作日後調閾值與換模型的 golden set；(4) 要對覆核人員做「AI 沒標的也可能有事」的訓練以對抗 automation bias。
  來源：<https://about.roblox.com/newsroom/2025/08/open-sourcing-roblox-sentinel-preemptive-risk-detection>、<https://www.musubilabs.ai/blog/the-top-challenges-of-using-llms-for-content-moderation-and-how-to-overcome-them>、<https://techbuzzonline.com/content-moderation-systems-guide/>、<https://www.webpurify.com/blog/how-to-build-a-trust-and-safety-team/>

- **(補充) 「脫離平台」訊號是業界公認的第一優先指標，bandu 的直覺有先例背書**（信心：medium）
  查證到的事實：Roblox 的安全體系明確把「反覆詢問孩子位置」「試圖把對話移往站外（Discord/Snapchat）」列為觸發調查的核心訊號，並以過濾器直接阻擋分享社群帳號、電話、站外邀請；誘騙者的典型路徑正是在主平台找目標後轉移到監管較少的管道。推測：bandu「第一優先抓脫離平台訊號」的設計判斷與最大規模兒童平台的實戰經驗一致；且 bandu 的封閉場景（無私訊、無圖傳、機構中介、成對固定關係）讓「約站外」幾乎是攻擊者唯一出路，訊號集中度比開放平台更高，偵測條件其實更有利。
  來源：<https://www.fileabuselawsuit.com/how-predators-use-roblox-to-target-children/>、<https://www.aicerts.ai/news/roblox-ai-safety-system-launches-to-protect-kids/>、<https://about.roblox.com/resource/child-safety>

### 未解問題

- 中文口語（台灣國語+台語混雜）經 ASR 轉錄後的錯字與諧音規避（+賴、ㄌㄞ、Line 寫成「連」等）對規則層命中率的實際影響——需自建測試集實測，未查到任何公開數據

- prompt-based LLM 分類在中文誘騙情境的實際 precision/recall 完全沒有公開 benchmark；自建紅隊評測集（LLM 合成+人工改寫誘騙對話、真實伴讀對話當負樣本）的方法學與倫理邊界

- Roblox Sentinel 的 embedding 與方法對中文短語音轉錄的遷移效果未知；替換多語言 embedding 後 skewness 聚合是否仍有效需 PoC 驗證

- 全程語音存檔+AI 分析在台灣個資法/兒少相關法規下的法遵要求（告知同意、保存期限、疑似案件的法定通報義務與流程）——屬另一研究維度但直接約束本管線的資料設計

- OpenAI moderation 免費政策與模型行為的長期穩定性；Perspective 落日後低成本中文毒性偵測的替代選項（自架開源 classifier vs 商用 API）比較

- 商用語音原生方案（ToxMod 等）的定價對公益專案是否可及——未查到公開價格；以及 50 人規模是否根本不需要語音原生偵測

- 誤報對志工體驗與留存的影響（被標記調查的志工如何告知、申訴流程）——業界 T&S 文獻多針對大平台，小型機構中介模式的先例未查到

## 安全營運先例與台灣法規（Outschool 等平台 trust & safety、教育部數位學伴 SOP、兒少權法查核、個資法、通保法錄音、志願服務法）

**結論**：bandu 提案的「全程錄音存檔＋平台內通訊＋禁止私下聯絡＋分級停權」與 Outschool 已運作多年的 trust & safety 模型高度一致，且純語音無鏡頭的設計比先例更保守，方向可行；台灣法規面最大的不確定點是志工查核——兒少權法 §81 的法定查核義務僅明文及於「兒童及少年福利機構」負責人與工作人員，若合作機構只是一般立案社會團體，查核將退為自願措施，需律師與主管機關函詢確認。錄音合法性在「平台揭露＋雙方（含法定代理人）事前同意」下依通保法 §29(3) 與刑法 315-1 的既有實務風險很低，真正要花功夫的是個資法端的家長同意書設計（未成年人同意回歸民法行為能力）、以及作為網路平台可能觸發的性剝削條例 §8（先行移除＋資料保留180日）與兒少權法 §53（24小時責任通報）義務動線。

**建議**：整體判斷：bandu 的安全架構提案有堅實先例支撐、無明顯法規紅線，可續推，但要把重心從「技術篩查」再平衡到「法遵動線」。具體建議：(1) 直接移植 Outschool 的三件套——全程錄音限定保存期（如 90 天）與用途、零容忍清單（誘導脫離平台/私下聯絡列為立即終止）、三級分級處置（教育提醒→記點→終止），這些都有公開文件可抄作業；(2) 正視與數位學伴的關鍵差異：官方計畫的安全底座是「兒少端有實體帶班老師在場」，bandu 若讓孩子在家單獨上線，早期（<50 同上）建議要求兒少端由合作機構據點集中上線或家長在場，LLM 篩查作為第二道網而非唯一防線；(3) 志工管理直接套志願服務法現成制度（機構備案＋基礎 6hr＋特殊 6hr 訓練＋紀錄冊＋意外險），特殊訓練時數用來上 bandu 自製的兒少保護與平台規範課；(4) 查核部分不要假設 §81 自動適用——若合作機構非兒少福利機構，改採「自願性要求警察刑事紀錄證明＋切結書＋函詢主管機關可否用不適任資料庫」三件組，並請律師確認要求良民證的個資法（§6 特種個資）適法設計；(5) SOP 必須內建兩條法定時限：兒少權法 §53 的 24 小時責任通報、性剝削條例 §8 的先行移除＋180 日資料保留（這條會覆蓋自訂的錄音刪除期限）；(6) 家長同意書是第一份該請律師看的文件——涵蓋錄音、逐字稿、LLM 分析、境外雲端傳輸（若用）、保存期、調閱權限，並附兒童適齡版說明。預算極低的情況下，(1)(2)(3) 零法律成本可先做，律師費集中花在 (4)(6) 兩點即可。

### 查證細項

- **(a) Outschool 錄影政策——與 bandu 提案幾乎同構的先例**（信心：high）
  查證事實：Outschool 所有直播課程一律錄影（"Outschool records all live classes for safety and quality assurance"），經 Zoom 錄製後上傳自家私有伺服器加密保存 90 天後刪除；用途限教師輔導、客服、合規，明言絕不對外分享；教師「不得以任何理由下載、上傳或儲存課程影片」。私人課程（1:1）也「for safety purposes」錄影。這證明「全程錄影＋限定保存期＋限定用途」是兒童線上教學的成熟業界作法，bandu 的全程語音存檔有直接先例可援。
  來源：<https://support.outschool.com/en/articles/452568-class-recordings>、<https://support.outschool.com/en/articles/2612466-learner-safety-and-privacy-for-teachers>、<https://support.outschool.com/en/articles/579976-learner-safety-and-privacy-for-parents>

- **(a) Outschool 通訊限制與「脫離平台」禁令**（信心：high）
  查證事實：教師「never sharing your personal contact information with parents, legal guardians, or learners」，所有溝通必須留在平台內（家長走 Parent Conversation Tab）；學生與教師間私訊對受信任成人（家長）可見；live chat 須在監控下才開放。另有課前身分核驗：「teacher can visually verify their identity at the start of each class」、學生名字須與帳號相符、旁聽家長應在鏡頭外。bandu 把「加LINE/私下聊」列第一優先偵測訊號，正對應 Outschool 的核心禁令；差異是 Outschool 靠人工＋家長可見性，bandu 想用 LLM 自動篩，屬創新但無直接先例（此句為推測/分析）。
  來源：<https://support.outschool.com/en/articles/2612466-learner-safety-and-privacy-for-teachers>、<https://support.outschool.com/en/articles/579976-learner-safety-and-privacy-for-parents>

- **(a) Outschool 違規分級（STRIKE）與零容忍停權**（信心：high）
  查證事實：三級記點制——第1次記點通知違規細節、第2次書面「最後警告」附安全指引、第3次「無需進一步通知」永久移除；記點「不可移除且不可申訴」；「不當接觸學習者（inappropriate contact with learners）」、武器/毒品/色情內容、背景查核失效屬零容忍、立即移除且「無資格重新申請」。背景查核：所有教師須通過身分驗證＋刑事背景查核（美國用 Checkr、英國用 Accurate，平台付費），須定期更新，逾期未更新或未通過即刻停權。bandu 可直接借用此分級框架，把「誘導脫離平台」列入零容忍清單。
  來源：<https://support.outschool.com/en/articles/11084107-understanding-the-trust-safety-strike-system-for-teachers>、<https://support.outschool.com/en/articles/2701506-educator-background-checks>

- **(b) 教育部數位學伴計畫的安全模型：實體帶班老師＋錄影檔檢核**（信心：medium）
  查證事實：每學期10週、每週2次、每次90分鐘（2節45分）、最晚20:30結束，一對一線上即時陪伴；採「集中管理式」上課——定時、定點、集體，小學伴集中在學習端學校教室，「師長們也會在現場，了解學童上課情形及需求」；教學品質管理含帶班老師填帶班日誌、大學伴填教學日誌、「教學教材與錄影檔檢核」（課程有錄影且事後檢核）。大學伴須「參加培訓、研習、備課」，但具體培訓時數與兒少保護課綱在公開網頁找不到（完整實施計畫藏在公文 PDF 附件，etutor.moe.gov.tw 有 TLS 憑證問題無法直接抓取）。推測/分析：數位學伴的安全核心是「兒少端永遠有實體大人在場」——這與 bandu 設想「孩子在家單獨上線」根本不同，bandu 的 LLM 篩查＋機構覆核正是在補這個缺口，早期可考慮要求兒少端由機構據點或家長在場作為過渡。
  來源：<https://etutor.moe.gov.tw/etutor/plans_introduction.php>、<https://depart.moe.edu.tw/ED2700/News_Content.aspx?n=727087A8A1328DEE&s=52FEAB3043C37A59>、<https://dsa.site.nthu.edu.tw/p/406-1266-234563,r6890.php?Lang=zh-tw>

- **(c) 兒少權法 §81：法定查核義務的內容與適用邊界【需律師確認】**（信心：high）
  查證事實：§81 規範「兒童及少年福利機構之負責人或工作人員」消極資格四款（性侵害/性騷擾/兒少性交易/性剝削犯罪經緩起訴或有罪確定；§49 行為查證屬實；客觀事實有傷害兒少之虞經認定不能執行職務；性侵/性騷/性霸凌查證屬實），聘僱前應檢具名冊、切結書、健檢表、「最近三個月內核發之警察刑事紀錄證明書」報主管機關核准，主管機關應主動查證。§81-1 為課後照顧班/中心版本（教育主管機關管）。但「兒童及少年福利機構」依 §75 限五類（托嬰中心、早療、安置教養、心輔/家諮、其他兒少福利機構）——若 bandu 合作的立案機構只是一般社會團體（多數課輔協會即是），§81 的強制查核義務不必然適用；子法「兒童及少年福利機構人員不適任資格認定資訊蒐集及查詢處理利用辦法」§2 將工作人員定義為「於機構內實際從事照顧、協助照顧、提供服務、擔任行政及其他提供勞務工作之人員」，未明文排除志工但也未明列。台灣目前沒有涵蓋所有接觸兒少者的一般性查核制度（「兒童工作證」仍在公共政策提點子平台連署階段）。需律師確認：志工是否落入辦法 §2、一般立案協會可否申請使用不適任資料庫查詢、以及要求志工提供警察刑事紀錄證明的個資法適法性（前科屬個資法 §6 特種個資，自願提供或需書面同意設計）。
  來源：<https://law.moj.gov.tw/LawClass/LawSingle.aspx?pcode=D0050001&flno=81>、<https://law.moj.gov.tw/LawClass/LawSingle.aspx?pcode=D0050001&flno=75>、<https://law.moj.gov.tw/LawClass/LawSingleRela.aspx?PCODE=D0050209&FLNO=2&ty=L>、<https://lawplayer.com/article/6438ee19e800e5f0b9334ace>、<https://join.gov.tw/idea/detail/ad692fea-fbca-450e-b487-3a7e99053ea8>

- **(c) 附帶義務：兒少權法 §53 責任通報與性剝削條例 §8 平台義務【需律師確認】**（信心：high）
  查證事實：兒少權法 §53——「醫事人員、社會工作人員、教育人員……及其他執行兒童及少年福利業務人員」知悉兒少遭受 §49 行為等情形，「應立即向直轄市、縣（市）主管機關通報，至遲不得超過二十四小時」，通報人身分保密（bandu 的機構覆核人員若屬「執行兒少福利業務人員」即負此義務——LLM 篩出可疑事件後的 SOP 必須內建這條時限）。兒童及少年性剝削防制條例 §8——「網際網路平臺提供者」知有第四章犯罪嫌疑情事，應先行限制瀏覽或移除相關資料，且「犯罪網頁資料與嫌疑人之個人資料及網路使用紀錄資料，應保留一百八十日，以提供司法及警察機關調查」。需律師確認：bandu 這種封閉式語音遊戲室是否構成該條例的「網際網路平臺提供者」、以及 180 日保留義務如何與自訂的錄音刪除期限銜接。
  來源：<https://law.moj.gov.tw/LawClass/LawSingle.aspx?pcode=D0050001&flno=53>、<https://laws.gov.taipei/Law/LawSearch/LawArticleContent/FL002523>、<https://www.lawbank.com.tw/news/NewsContent.aspx?NID=190332.00>

- **(d) 個資法：無兒童專章，未成年同意回歸民法行為能力【需律師確認同意書設計】**（信心：medium）
  查證事實：台灣個資法沒有 COPPA 式的兒童個資專章或年齡門檻；主管機關見解為未成年人個資之蒐集與同意「回歸適用民法關於行為能力的規定」——未滿7歲（無行為能力）須由法定代理人同意；7歲以上限制行為能力人原則上須法定代理人同意，但依其年齡及身分、日常生活所必需等情形可自行同意；14歲以上實務見解傾向依民法一般規定判斷。告知義務（§8）須載明蒐集機關、目的、個資類別、利用期間/地區/對象/方式、當事人權利、不提供之影響，且對兒童須用「符合學童之年齡、生活經驗及理解能力」的語言，不得以贈品不當利誘（§5 誠信原則）。語音錄音檔屬得識別個人之個資。推測/分析：bandu 應對每位兒少取得法定代理人書面明示同意（涵蓋錄音、逐字稿轉換、LLM 分析、保存期限、機構人員調閱範圍）＋對兒童本人做適齡告知，志工端另簽同意書；這與 Outschool 以 parental consent 為基礎的作法一致。需律師確認：同意書條款、逐字稿交 LLM 處理是否為特定目的內利用、若用境外雲端 LLM 之國際傳輸（§21）問題。另 2023 修法後個資保護委員會（籌備處）已設立，兒童個資規範有入法討論，需追蹤修法動態。
  來源：<https://wisebeamlaw.com/pdpa-compliance/>、<https://laws.gov.taipei/Law/LawSearch/LawRelateInterpretation/FL010627?no=19>、<https://law.moj.gov.tw/LawClass/LawAll.aspx?PCode=I0050021>、<https://www.pdpc.gov.tw/News_Html/100/>

- **(e) 通保法與刑法 315-1：「平台揭露＋雙方同意」錄音的合法性——風險低**（信心：high）
  查證事實：通保法 §29(3) 明文「監察者為通訊之一方或已得通訊之一方事先同意，而非出於不法目的者，不罰」——即一方事先同意即可阻卻；刑法 §315-1 處罰「無故」竊錄「他人非公開之活動、言論、談話」，若錄音者為對話一方，他方「無從對參與對話之他方主張具有合理隱私期待」，法院並肯認「保全日後證據」屬合法正當目的。推測/分析：bandu 作為第三方（平台）錄音，依 §29(3) 文義取得「通訊之一方事先同意」即已達法定門檻，而 bandu 設計為服務條款揭露＋志工與兒少法定代理人「雙方」事前書面同意，明顯高於法定最低標準，通保法/刑法層面的違法風險很低；真正的合規重心不在通保法而在個資法端（見上項）。需律師確認：告知與同意文字、以及兒少一方之同意由法定代理人行使的介面設計即可，屬低風險確認項。
  來源：<https://www.legis-pedia.com/article/crime-penalty/626>、<https://lylaw.tw/article-content.asp?ids=40>、<https://www.tph.moj.gov.tw/4421/4475/632364/960875/post>

- **(f) 志願服務法：bandu 合作機構可直接套用的現成制度**（信心：high）
  查證事實：§3 運用單位＝「運用志工之機關、機構、學校、法人或經政府立案團體」（立案協會即可）；§7 應檢具志願服務計畫及立案登記證書送主管機關及目的事業主管機關備案；§9 應辦基礎訓練＋特殊訓練——衛福部 2018 年公告基礎訓練修正為 3 課目 6 小時（志願服務內涵及倫理、經驗分享、法規認識），社會福利類特殊訓練 4 課目 6 小時（含運用單位業務簡介及工作內容說明含實習）；§12 應發給志願服務證及服務紀錄冊並指定專人督導（完成基礎＋特殊訓練方可請領紀錄冊）；§16 「應為志工辦理意外事故保險」，衛福部有年度志工意外團體保險共同供應契約（公告範例：意外身故最高 300 萬、醫療 3 萬、住院日額 2,000，涵蓋服勤前後通勤 2 小時）。推測/分析：這套制度是現成骨架——bandu 志工掛在合作機構名下跑備案、6+6 小時訓練、紀錄冊、保險，行政成本低且完全合規；特殊訓練 6 小時正好可塞 bandu 自訂的兒少保護＋平台安全規範課程。此項法制明確，無需律師即可執行（僅保險投保細節向地方社會局確認）。
  來源：<https://law.moj.gov.tw/LawClass/LawAll.aspx?PCode=D0050131>、<https://dep.mohw.gov.tw/dosaasw/cp-530-41204-103.html>、<https://socbu.kcg.gov.tw/index.php?prog=2&b_id=16&m_id=85&s_id=2393>、<https://sab.tycg.gov.tw/News_Content.aspx?n=7337&s=684687>

### 未解問題

- 教育部數位學伴計畫完整實施計畫 PDF（大學伴培訓時數、兒少保護/性平課綱、錄影檔檢核細節）未能取得——etutor.moe.gov.tw 有 TLS 憑證問題、公文附件未公開直鏈，需向教育部資科司或任一夥伴大學索取

- 兒少權法 §81 及其子法（D0050209 §2）之「工作人員」是否涵蓋志工、一般立案社會團體（非 §75 兒少福利機構）可否申請不適任資料庫查詢——需函詢衛福部社家署並由律師確認

- 要求志工提供警察刑事紀錄證明（良民證）之個資法 §6 特種個資適法設計（自願提供 vs 書面同意 vs 法定依據）——需律師

- 兒少語音逐字稿交 LLM 分析是否屬特定目的內利用、使用境外雲端 LLM 之個資國際傳輸（§21）限制——未查證，需律師

- bandu 的封閉式語音遊戲室是否構成性剝削條例 §7/§8 之「網際網路平臺提供者」而負移除與 180 日資料保留義務——需律師

- 個資保護委員會正式成立後是否推出兒童個資專章（修法草案動態）——需持續追蹤

- Outschool 官方 trust 頁（outschool.com/trust）為 JS 渲染無法抓取全文，教師背景查核之「定期重驗頻率」具體年限未能從公開支援文件確認

## 教學主題與免費素材（分齡）：共讀、金融素養、法律素養、職業探索之免費合法素材盤點與授權查證

**結論**：共讀素材方面，最知名的文化部「兒童文化館」繪本動畫經查證明確「僅限該站內使用、需事前書面同意」，不可嵌入 bandu，只能超連結；真正可安全使用的是逐本標示創用 CC 的國資圖「圓夢繪本資料庫」與人人可免費辦證的國資圖電子書平台（各自借閱模式）。金融素養有金管會金融智慧網分齡專區（國小/國中）與銀行局國小 3-6 年級起的六大主題教材，政府自產內容原則上適用「政府網站資料開放宣告」（相容 CC BY 4.0）；法律素養（民間公民與法治教育基金會教材包、司法院圖說司法動畫）與職業探索（新北職探中心 7 職群 YouTube 數位課程、教育部 CIRN 生涯專區）素材充足且免費。最關鍵的授權結論：政府開放宣告是通則但「個站聲明優先」且第三方著作除外，NGO 教材多為「免費下載但保留所有權利」——bandu 應採「連結/嵌入官方來源、雙端各開素材、語音同步共讀」架構，避免任何重製上架，需內嵌者逐一取得書面授權。

**建議**：建議 bandu 把素材架構定為「索引不重製」（link-not-host）：平台只存書目/連結/進度元資料（兒童文化館書目屬開放資料可自由用），內容一律由志工與孩子雙端各自開啟官方來源，遊戲室純語音同步討論——這與純語音無圖傳的安全架構天然吻合，且把著作權風險降到近零。具體優先序：(1) 共讀主力用「圓夢繪本資料庫」（逐本 CC、免登入）與國資圖電子書平台（onboarding 時協助孩子與志工各辦一張免費借閱證，未成年辦證的家長同意流程順勢併入機構知情同意）；兒童文化館只做站外連結，絕不內嵌。(2) 金融（國小 3 年級起）與法治用政府自產教材（金融智慧網分齡區、司法院圖說司法），使用前逐站確認開放宣告後在標示來源下使用；(3) 職探用新北職探 YouTube 影片之官方嵌入或連結（國小高年級～國中）。(4) 對 LRE、FINLEA 這類「免費下載但保留權利」的 NGO 教材，以公益平台名義發函洽談書面授權——它們本以推廣為目的，成功率高，但取得前只連結。(5) 建一張素材授權台帳（每筆：來源、適齡、授權條款、可否內嵌、確認日期），當作日後擴充素材的守門機制；早期 50 人以下規模，人工維護足矣。

### 查證細項

- **（a）文化部兒童文化館：內容豐富但授權最嚴——不可在第三方平台內使用**（信心：high）
  查證到的事實：兒童文化館（1999 年設立）有「閱讀花園/繪本花園」每月新繪本動畫、聽書、聽唱兒歌等區，適齡約學齡前至國小低中年級。其「著作權宣告及授權說明」明載：內容係「經各著作財產權人同意合法使用」、未採創用 CC 或政府資料開放授權；「繪本動畫與有聲書僅限在兒童文化館網站內使用」，轉載/重製/公開傳輸需事前書面同意，「聽唱兒歌」需另依文化部授權利用及收費要點申請；但「允許自由以超連結方式連結本網站」。另政府資料開放平臺上開放的只有「繪本花園動畫書目」的書目資料（政府資料開放授權條款第 1 版），不含內容本身。結論：bandu 只能做站外連結＋自建書目索引（書目資料可用），不能把動畫嵌進遊戲室。
  來源：<https://children.moc.gov.tw/index>、<https://children.moc.gov.tw/copyright>、<https://data.gov.tw/dataset/24968>

- **（a）國立公共資訊圖書館電子書服務平臺：全國免費、量大，適合「各自借閱＋語音共讀」模式**（信心：high）
  查證到的事實：平台約有 2.7 萬種、25 萬冊以上正體中文電子書，全國民眾持任一國內公共圖書館借閱證即可透過「一證通」註冊使用；國資圖提供網路辦證（本國人及持居留證外籍人士，審核約 3 個工作天，可選實體或數位證）。推測：電子書為 DRM 逐人借閱授權，bandu 不可能取得內容重製權嵌入平台；但因辦證免費全國通行，「志工與孩子各自持證借同一本書、在純語音房內同步翻頁共讀」是零授權風險的可行模式（志工端成本為零）。
  來源：<https://www.nlpi.edu.tw/ReaderService/LoanService/Collection01/Loan03.htm>、<https://allpass.nlpi.edu.tw/>、<https://lib.bocach.gov.tw/News_Content.aspx?n=893&s=41419>

- **（a）圓夢繪本資料庫：逐本標示創用 CC，是共讀素材中授權最乾淨的本土來源**（信心：high）
  查證到的事實：國資圖「圓夢繪本資料庫」收錄學生創作繪本（實測樣本：桃園市八德國中學生創作之英文繪本，頁面明確標示 CC BY-NC-ND 4.0；搜尋結果顯示館內另有 BY-NC-SA 2.5 台灣等不同條款，逐本標示），可直接線上閱讀，樣本頁未見登入要求。適齡：內容為國小～國中學生創作，題材貼近同齡孩子，低中高年級與國中皆有可用作品。推測：BY-NC-ND/BY-NC-SA 之「非商業性」與公益免費平台相容；ND 條款下不可改作（不可重新配音改編），依條款重製與公開傳輸原則上可行，但保守作法仍以連結原站＋各自開頁為主。
  來源：<https://storybook.nlpi.edu.tw/book-single.aspx?BookNO=1499>、<https://ers.nlpi.edu.tw/>

- **（a/e）教育大市集：教育部彙整 15 萬筆資源、多採創用 CC 但逐項為準**（信心：high）
  查證到的事實：教育部「教育大市集」整合 22 縣市教育單位、部屬機構及民間單位資源，累積超過 15 萬筆，國中小高中職學習單/簡報/影片超過 11 萬筆；官方 FAQ 說明資源「授權方式多採用創用 CC 授權，但使用上請依實際授權方式為準」（即逐項標示、逐項判斷）。這是 bandu 找學習單型輔材（各科、各年級皆有）的主要合法來源，選用時只取 CC 標示明確且允許重製之項目。
  來源：<https://market.cloud.edu.tw/>、<https://market.cloud.edu.tw/about/faq.jsp?faq_type=3>

- **（a）CC 授權國際繪本平台 StoryWeaver：CC BY 4.0，繁中供應量未能驗證**（信心：low）
  查證到的事實：Pratham Books 的 StoryWeaver 提供數千本免費童書、380 種以上語言，授權為 CC BY 4.0（允許重製、散布、改作、商用，僅需姓名標示——對平台內嵌最友善）。未能查證：繁體中文書目數量與品質（語言篩選頁回 403，僅搜尋摘要稱「似有支援」）。若繁中量足，這是唯一可直接把內容做進 bandu 遊戲室的繪本來源；若不足，可作為英語伴讀素材（國小中高年級以上）。
  來源：<https://storyweaver.org.in/>、<https://play.google.com/store/apps/details?id=theunreal.storyweaver.app>

- **（b）金融素養：金管會金融智慧網分齡專區＋銀行局校園宣導教材（國小 3-6 起）**（信心：high）
  查證到的事實：金管會「金融智慧網」金融教室設有明確分齡專區——「小學生麻吉充電區」「中學生知識搖滾區」（另有大學生/社會/銀髮區），並有防詐騙專區與金融試算工具；頁尾掛「政府網站資料開放宣告」。金管會自 95 年起結合銀行公會、金融研訓院、中央存保等辦理校園宣導，對象自國小 3-6 年級、國中、高中職起，內容六大主題：正確金錢觀、正確用卡、正確理財、正確理債、詐騙防制與救濟、消費者金融權益。適齡結論：金融素材官方定位即從國小中年級（3 年級）開始，低年級無公版教材。注意：金融教室頁面實際列出的 68 筆教材以大學/社會/銀髮影片為主，國小國中區的具體教材清單需進站逐項確認。
  來源：<https://moneywise.fsc.gov.tw/home.jsp?id=1&parentpath=0>、<https://www.fsc.gov.tw/ch/home.jsp?id=644&parentpath=0,7>、<https://www.fsc.gov.tw/ch/home.jsp?id=1089&parentpath=0,5>

- **（b）金融基礎教育課程模組與 FINLEA 民間教材：可下載但授權未開放**（信心：medium）
  查證到的事實：金管會與國教署合辦金融基礎教育課程模組推廣（103-108 年徵選教學方案、111 年高中模組試辦、對象涵蓋國小國中教師，113/115 年度有推廣合作計畫網站）。民間 FINLEA 財金智慧教育推廣協會提供「個人理財-國小篇」「國小學童財金生活教育」、國中/高中/高職教材、「小豆子的理財大冒險」動畫等免費下載，但網站標示「All Rights Reserved」、無開放授權條款。推測：這類推廣型教材對公益平台的個別授權洽談成功率高，但在取得書面同意前只能連結、不能內嵌。
  來源：<https://sites.google.com/view/twnposfinancialnew>、<https://www.finlea.org.tw/List.aspx?id=61>、<http://www.clvsc.tyc.edu.tw/files/14-1000-37203,r174-1.php>

- **（c）法律素養：民間公民與法治教育基金會教材包（國小中高年級）＋司法院圖說司法（國中）**（信心：high）
  查證到的事實：民間公民與法治教育基金會（LRE）2023 年「法治教育教材包【推廣版】」涵蓋責任、分配正義、隱私、權威四主題，適用國小中、高年級，設計為 1 堂課，免費但需填資料下載；網站標示 © All rights reserved，無明示開放授權。司法院「圖說司法」平台有「法律吧」動畫（童話改編傳遞法律概念，適國小高年級～國中）、少年法治教育影片系列，並建置「法治教育人才資料庫」可派講師；司法院另有官方 YouTube 頻道。法律扶助基金會查證結果：主力為弱勢法律扶助服務與「人權法治列車」入校宣講（深耕校園 14 年），未查到可直接下載的兒少自學教材，其角色較適合作為「志工端的法律知識講師/內容顧問」而非素材庫。
  來源：<https://www.lre.org.tw/news/801>、<https://www.lre.org.tw/node/499>、<https://social.judicial.gov.tw/saylaw/videos/VideoList/animation>、<https://www.youtube.com/user/JudicialYuan/featured>、<https://www.laf.org.tw/>

- **（d）職業探索：新北職探中心 7 職群數位課程（YouTube 公開）＋教育部 CIRN 生涯專區＋國中生涯手冊**（信心：high）
  查證到的事實：新北市 16 所職探中心自製「職探數位課程影片」涵蓋七大職群（工業、商業、農業、家事、海事水產、藝術、醫護），每部約 15 分鐘、生活化實作導向、含雙語版，已上架新北市教育局 YouTube（@ntpcedu）及新北技職教育資源網（tve.ntpc.edu.tw），且教育局公告「鼓勵全校教師踴躍使用」；職探育樂營對象為國小 4-6 年級與國中生，可作為適齡定位參考。教育部 CIRN 平臺設「生涯發展教育」專區（國中小課程與教學資源），各縣市「國中學生生涯發展紀錄手冊」PDF 在多所學校網站公開下載（國中適用）。推測：YouTube 公開影片依 YouTube 服務條款可用內建嵌入功能嵌入第三方網頁，這是職探素材進入 bandu 最乾淨的路徑（但 bandu 遊戲室為純語音無畫面，實際使用會是「孩子自己裝置上看、房內語音討論」）。
  來源：<https://www.dfes.ntpc.edu.tw/p/406-1000-8131,r164.php?Lang=zh-tw>、<https://udn.com/news/story/6885/8510119>、<https://sites.google.com/apps.ntpc.edu.tw/ntcceeec/>、<https://cirn.moe.edu.tw/WebFile/index.aspx?sid=1182&mid=11018>、<https://www.cpjh.ntpc.edu.tw/p/406-1000-6143,r293.php>

- **（e）授權總則：政府網站資料開放宣告相容 CC BY 4.0，但「個站聲明優先、第三方著作除外」**（信心：high）
  查證到的事實：我國政府網站普遍採「政府網站資料開放宣告」，政府自產資料以相當於 CC BY 4.0 之條件開放（可重製、散布、公開傳輸、改作、含商用，僅需標示來源機關）；「政府資料開放授權條款—第 1 版」明文可直接轉換為 CC BY 4.0 且允許再授權。關鍵限制（兒童文化館為實證案例）：機關網站上「經第三方授權刊載」之內容不在開放範圍，且個別網站可訂更嚴格條款並優先適用——兒童文化館即明文排除開放授權。實務結論：政府「機關自產」的教材 PDF/影片（如金管會宣導教材、司法院自製動畫、生涯手冊）多可在標示來源下於 bandu 內使用，但每一個站都要先讀它自己的宣告頁，凡素材內含商業出版社圖文（繪本尤其如此）一律視為第三方著作、只連結不重製。
  來源：<https://data.gov.tw/license>、<https://www.ithome.com.tw/news/97922>、<https://www.ndc.gov.tw/cp.aspx?n=3834119503953383>、<https://children.moc.gov.tw/copyright>

- **分齡總表（依查證素材整理）**（信心：medium）
  國小低年級（1-2）：兒童文化館繪本動畫（僅能連結站外閱讀）、圓夢繪本資料庫低齡作品、國資圖電子書童書（各自借閱）——此齡層無公版金融/法律教材，以共讀為主。國小中年級（3-4）：上述共讀素材＋金管會宣導教材起始年段（國小 3-6）、金融智慧網小學生麻吉充電區、FINLEA 國小篇、LRE 法治教材包（國小中年級起）。國小高年級（5-6）：加上新北職探數位課程（育樂營對象國小 4-6）、司法院「法律吧」動畫、教育大市集高年級學習單。國中：金融智慧網中學生知識搖滾區、FINLEA 國中教材、司法院少年法治教育影片系列、國中生涯發展紀錄手冊、職探中心國中課程、圓夢繪本國中生創作作品（同儕感強）。此表為對前述各查證事實的彙整；各素材官方適齡標示以其原站為準。
  來源：<https://moneywise.fsc.gov.tw/home.jsp?id=1&parentpath=0>、<https://www.lre.org.tw/news/801>、<https://www.dfes.ntpc.edu.tw/p/406-1000-8131,r164.php?Lang=zh-tw>、<https://storybook.nlpi.edu.tw/book-single.aspx?BookNO=1499>、<https://cirn.moe.edu.tw/WebFile/index.aspx?sid=1182&mid=11018>

### 未解問題

- StoryWeaver 繁體中文繪本的實際數量與品質未能驗證（頁面 403）——若量足，它是唯一可直接內嵌 bandu 的 CC BY 繪本庫，值得人工上站清點

- 金融智慧網「小學生麻吉充電區/中學生知識搖滾區」的具體教材清單與各件授權（是否含第三方著作）需進站逐項確認；銀行公會教材頁已 404，最新載點待查

- 司法院圖說司法與 moneywise 各自的「政府網站資料開放宣告」全文未逐字取得（部分頁面擋爬蟲），需逐站人工確認有無『僅限本站使用』類例外

- 國資圖電子書『各自借閱』模式的細節：未成年人辦證是否需法定代理人同意、同一本書同時可借冊數是否足以支撐志工＋孩子同時借閱、熱門書等待期對排課的影響

- LRE／FINLEA／法扶對公益平台內嵌或改作其教材的授權意願（需正式發函）；法扶會是否有未公開於網站的兒少教材或願意提供志工培訓內容

- 除新北外，其他縣市職探中心（桃園幸福國中示範中心等）是否有公開線上教材可納入索引；教育局 YouTube 影片是否全部開放嵌入（個別影片可關閉嵌入功能）

## 遊戲室互動形態

**結論**：開源積木（Excalidraw MIT、pdf.js Apache-2.0、epub.js BSD-2-Clause、Yjs/boardgame.io MIT）足以在零授權費下組出「共享白板＋同步共讀＋簡單回合制遊戲」的遊戲室，且全部能以 npm 元件/library 形式深度嵌入自建平台、留在平台安全殼與存檔機制之內。台灣既有伴讀計畫驗證了這個形態：教育部數位學伴十餘年來就是「JoinNet 白板＋語音＋手寫板」模式且已於 2025-12-31 結束，KIST 學伴用商業 RAZ-Kids 繪本做一對一英文共讀——但 RAZ-Kids 與 Minecraft Education（非學術機構 US$36/人/年、原生 app、自帶文字聊天）都無法嵌入受控遊戲室，後者還會繞過語音監控。低配舊平板上，audio-only WebRTC＋canvas 白板＋伺服器端預轉圖的同步翻頁可行；Minecraft 類原生 app（Chromebook 需 4GB RAM、最低規格仍會掉幀卡頓）則吃緊，不宜作 MVP。

**建議**：MVP 遊戲室採「純開源自建三件組」：(1) 自建 web shell 負責 audio-only WebRTC、錄音與逐字稿管線；(2) 嵌入 @excalidraw/excalidraw（MIT）作共享白板＋紙筆遊戲（井字、你畫我猜、接龍），自架 excalidraw-room 並關閉所有分享/匯出旁路；(3) 自製同步共讀器——伺服器端把機構審核過的 PDF/EPUB（優先公有領域與 CC 讀物）預轉成頁面圖片，client 以 WebSocket/Yjs（MIT）同步頁碼與指讀游標。理由：全部 MIT/Apache-2.0/BSD-2-Clause 授權、零授權費、可完整嵌入受控環境，互動內容全數留在平台稽核範圍內，且對 4GB 級舊平板友善；此形態並非賭注——教育部數位學伴十餘年的 JoinNet「白板＋語音」模式與 KIST 的 RAZ-Kids 一對一共讀已驗證這就是伴讀的核心互動。桌遊擴充用 boardgame.io（MIT）自寫 2-3 個簡單回合制遊戲即可，但認知它已停維護、不作長期地基，勿引入 AGPL 的 FreeBoardGames 程式碼、勿實作有 IP 的商業桌遊。明確不建議 MVP 採用 Minecraft Education：原生 app 無法嵌入、需 M365 帳號管理、非學術機構 2025-09 起 US$36/人/年（50 對≈US$3,600/年）、內建文字聊天繞過語音監控形成脫離平台破口——若日後合作學校端已有學術授權且證實可停用/稽核遊戲內聊天，再作 P2 評估。RAZ-Kids 不可嵌入，若機構堅持要英文繪本，走「機構購教室授權（US$110-132/年/36 童）＋螢幕分享」過渡，長期仍應把 CC 讀物庫做進自建共讀器。另建議趁教育部數位學伴 2025 年底落幕的真空期，接觸其原夥伴學校與帶班老師網絡作為機構端切入口。

### 查證細項

- **Excalidraw：MIT 授權、官方 npm 元件可嵌入，協作需自架 room server**（信心：high）
  查證事實：Excalidraw 編輯器為 MIT 授權開源，官方提供 @excalidraw/excalidraw npm 套件（現行 0.18.1，npm 註冊資料確認 license=MIT），以 React 元件形式直接嵌入自家 app，字型等資產可自行 self-host（設 window.EXCALIDRAW_ASSET_PATH）。要即時協作，官方自架文件明載必須另外部署 excalidraw-room（Node.js WebSocket 服務）做加密狀態同步；房間 ID 與加密金鑰放在 URL fragment、不上伺服器。自架版不限板數與協作人數。推測：對 bandu 而言這是最低風險的第一塊積木——嵌入自家 shell 後可完全關閉分享連結、匯出等旁路功能，白板內容也可由平台端快照存檔；但 E2E 加密的協作通道若沿用官方 room 協議，平台端要留稽核副本需自行在 client 端旁錄，此點需 PoC 驗證。
  來源：<https://plus.excalidraw.com/docs/self-hosting/excalidraw-open-source-selfhosting>、<https://docs.excalidraw.com/docs/@excalidraw/excalidraw/installation>、<https://www.npmjs.com/package/@excalidraw/excalidraw>、<https://github.com/excalidraw/excalidraw>

- **PDF/EPUB 同步共讀：查無現成「兩人同步翻頁」開源方案，需以 pdf.js/epub.js＋Yjs 自組**（信心：medium）
  查證事實（負面結果）：搜尋開源同步共讀方案，找到的 OpenReader（read-along 指單人 TTS 朗讀跟讀，可自架）與 Readest（單一使用者跨裝置進度同步）都不是「志工與孩子兩端即時同步翻頁」的工具；未找到任何被廣泛採用的 turnkey 雙人同步共讀 OSS。查證事實（積木授權，npm registry 確認）：pdfjs-dist 6.1.200 為 Apache-2.0（Mozilla）、epubjs 0.3.93 為 BSD-2-Clause、Yjs 13.6.31 為 MIT——三者皆允許商用嵌入與修改。推測：同步翻頁本質只是廣播「目前頁碼/捲動位置」一個小狀態，用 WebSocket 或 Yjs 在自建平台內實作是數天級工作量，反而比整合第三方產品乾淨；且自建 viewer 才能保證教材內容經機構審核、無外連。
  來源：<https://github.com/richardr1126/openreader>、<https://readest.com/>、<https://www.npmjs.com/package/pdfjs-dist>、<https://www.npmjs.com/package/epubjs>、<https://www.npmjs.com/package/yjs>

- **boardgame.io：MIT 授權但實質停止維護（最後發版 2022-11-10）；現成遊戲平台 FreeBoardGames.org 為 AGPL-3.0**（信心：high）
  查證事實：boardgame.io 為 MIT 授權的回合制遊戲引擎（GitHub 12.4k stars），提供狀態同步、大廳配對、bot、vanilla JS/React 綁定，以 library 形式存在——天生可完整嵌入自建平台、不依賴外部服務。但 npm registry 顯示最新版 0.50.2 發佈於 2022-11-10，之後三年半無新版，GitHub 也無 releases，實質處於停維護狀態。其上的現成遊戲集 FreeBoardGames.org 採 AGPL-3.0，且生態活躍度低（12 個月無 major release）。推測：拿 boardgame.io 寫 2-3 個自製小遊戲（圈叉、記憶配對、賓果）風險可控——引擎功能封閉完整、無安全面暴露；但別把它當長期平台基礎，也別直接引入 AGPL 的 FreeBoardGames 程式碼（會感染自家授權）；另外自製遊戲應避開有商標/美術 IP 的商業桌遊，只用規則屬公有領域的傳統遊戲（此句為一般法律常識推測，未查證台灣個案）。維護中的替代品 Colyseus（MIT，npm 確認）可作備案。
  來源：<https://github.com/boardgameio/boardgame.io>、<https://www.npmjs.com/package/boardgame.io>、<https://github.com/freeboardgames/FreeBoardGames.org>、<https://www.npmjs.com/package/colyseus>

- **Minecraft Education：學術機構 US$5.04/人/年、非學術機構 2025-09 起 US$36/人/年，原生 app 無法嵌入網頁平台**（信心：medium）
  查證事實：官方 FAQ 與多方來源一致指出合格學術機構價為 US$5.04/人/年（許多 K-12 學校經 M365 Education 免費取得），非學術機構（營隊、社團、課後班、nonprofit）走商用授權；官方「Commercial License Changes」FAQ 顯示商用價已於 2025-09-04 由 US$12 調整為 US$36/人/年。每位使用者都需要 Microsoft 365 帳號，試用限教師 25 次/學生 10 次登入。它是原生 app（Windows/Mac/iPad/Chromebook/Android），非網頁元件，無法 iframe 進自建遊戲室。推測：以 bandu 50 對同時上線估算，100 個帳號×US$36≈US$3,600/年（約 NT$11 萬），對個人出資專案是大負擔；更關鍵的是 Minecraft 多人模式內建文字聊天（此句為常識推測），走在 bandu 語音存檔＋逐字稿篩查之外，形成「脫離平台監控」破口——除非能證明可由伺服器端關閉/紀錄聊天，否則與安全架構直接衝突，不宜納入 MVP。
  來源：<https://edusupport.minecraft.net/hc/en-us/articles/360047119092-FAQ-Availability-Pricing-and-Licensing>、<https://edusupport.minecraft.net/hc/en-us/articles/38684236168980-FAQ-Commercial-License-Changes>、<https://www.learninggame.org/how-much-does-minecraft-edu-cost/>、<https://education.minecraft.net/en-us/licensing>

- **誠致 KIST 學伴：RAZ-Kids 數位繪本＋一對一線上，2022 年規模 70 位大學伴/330 位孩子**（信心：high）
  查證事實：誠致教育基金會 KIST 英文悅讀學伴計畫源自「花東英語悅讀計畫」（2021 接手），採 BigByte Education 提供的 RAZ-Kids 數位英語繪本做線上一對一（或一對多）共讀，每位孩子每次 10-15 分鐘、志工每週 2-3 小時、備課 10-20 分鐘；國中英數學伴則用學校教材與會考題，每孩子 30-60 分鐘。2022 成果文：110 學年 24 位→111 學年 70 位大學伴，服務 5 所 KIST 學校＋成功教會共 330 位學生、陪伴逾 2,400 小時。官網招募頁與成果文均未載明視訊工具為何（Google Meet/Zoom 未證實）。推測：KIST 模式證明「一本共讀教材＋語音對話」就是伴讀的核心互動，遊戲室 MVP 做好同步共讀器即可覆蓋主場景；其 RAZ-Kids 授權由合作方 BigByte 打包提供，bandu 難以直接複製此安排。
  來源：<https://www.chengzhiedu.org/partner-career/reading-buddy/>、<https://www.chengzhiedu.org/blog/2022-reading-buddy/>

- **RAZ-Kids 授權：商業產品、教室授權約 US$110-132/年（1 位教師/最多 36 名學生），無公開嵌入 API**（信心：medium）
  查證事實：RAZ-Kids 為 Learning A-Z 的商業產品，授權按年、按教師（classroom）或家庭計；近年教室授權行情約 US$109.95-132/年，涵蓋 1 位教育者與最多 36 名學生，區域經銷商報價可達 US$150-170。查證事實（負面結果）：搜尋過程未見 Learning A-Z 提供可將閱讀器嵌入第三方平台的公開 API 或 embedding 方案。推測：若 bandu 要用 RAZ-Kids，只能「機構購教室授權＋志工在遊戲室內螢幕分享或另開分頁」，後者會離開受控環境；預算面上一個教室授權可涵蓋 36 個孩子、單價低，但帳號註冊會把兒少個資交給境外服務商，需在法遵維度另行評估。替代路線：公有領域/CC 授權讀物（如國資圖電子書、Project Gutenberg 兒童書）放進自建共讀器，零授權金且全程受控。
  來源：<https://help.learninga-z.com/en/articles/7026222-pricing-discounts>、<https://www.learninga-z.com/site/products/raz-kids/overview>

- **教育部數位學伴：JoinNet（白板＋語音＋視訊＋手寫板）為標準配備，計畫已於 2025-12-31 結束**（信心：high）
  查證事實：教育部數位學伴計畫（隸屬「推動數位平權方案 109-112」）以大學伴對偏鄉 3-9 年級小學伴做線上一對一即時課輔，入口網明載使用 JoinNet 線上教學平台，兩端配備電腦、手寫板（drawing tablet）、耳機麥克風、webcam。計畫已於 114 年（2025）12 月 31 日結束、完成階段性任務，教育部轉向 AI 輔助學習方向。推測：這留下兩個對 bandu 有利的訊號——(1) 十餘年的官方實務驗證「共享白板＋語音＋全程連線」就是遠距伴讀的成熟互動形態，bandu 的遊戲室設計（白板＋純語音＋存檔）與之同構且更強化監控；(2) 計畫落幕後出現志工人力與偏鄉需求的真空，既有帶班老師/夥伴學校網絡可能是 bandu 的機構合作切入點。JoinNet 本身為商用軟體，非開源、不可自行嵌入。
  來源：<https://etutor.moe.gov.tw/zh/node/128>、<https://etutor.moe.gov.tw/edu_index/introduction_list.php>、<https://www.edu.tw/News_Content.aspx?n=9E7AC85F1954DDA8&s=903873EA4C66A0DF>

- **台灣微客數位伴讀：學期間視訊連線課後陪伴（屏東內埔、花蓮壽豐等據點），具體工具未公開**（信心：medium）
  查證事實：台灣微客公益行動協會（Waker，2012 年立案）在屏東內埔三地部落、花蓮壽豐等據點於學期間執行「數位伴讀」計畫，以視訊連線方式帶孩子課後學習與手作活動；官網未載明使用的視訊平台、頻率與志工端工具，志工體系以梯隊制培訓為主。推測：微客模式是「據點集中上課＋遠端志工連線」而非孩子在家單獨連線，這降低了兒少端的裝置與監護問題——bandu 若同樣以「機構據點教室＋平台遊戲室」起步，可沿用學校/據點既有裝置與大人在場的物理監督，安全論述更容易成立。工具細節需直接詢問微客（02-3322-5103）。
  來源：<https://www.waker.org.tw/service-team/Taiwan>、<https://www.waker.org.tw/about>

- **低配裝置（學校舊平板）：Minecraft 類原生 app 門檻高，audio-only＋canvas 白板＋預轉圖翻頁可行**（信心：medium）
  查證事實：Minecraft Education 在 Chromebook 上需 ChromeOS 83+、4GB RAM、1GB 儲存空間且裝置須支援 Google Play Services；官方明言最低規格下會有低幀率、渲染距離縮短、偶發卡頓。教育部「生生用平板」政策（111 年起 4 年 200 億）已使偏遠地區學校達到學生 1 人 1 機，配發機型含 iPad、Chromebook、Surface Go 等——即目標孩子端多半有平板可用，但等級不一且已服役 3-4 年。推測（工程判斷，未實測）：bandu 的技術選型天然對低配友善——純語音 WebRTC（Opus）頻寬與 CPU 佔用極低、遠輕於視訊；Excalidraw 為 canvas 渲染，一對一小場景在舊平板可順跑，但應限制場景元素數量；PDF 共讀若在 client 端用 pdf.js 全量渲染複雜頁面會慢，建議伺服器端預先把每頁轉成固定解析度圖片再推送，client 只做圖片翻頁＋座標同步。這三項組合的記憶體足跡遠低於任何 3D 遊戲方案。
  來源：<https://edusupport.minecraft.net/hc/en-us/articles/360047556591-System-Requirements>、<https://support.google.com/chrome/a/answer/10019965?hl=en>、<https://www.ey.gov.tw/Page/448DE008087A1971/0ba98487-02aa-4bf3-afa5-89a628e29c9d>、<https://www.edu.tw/News_Content.aspx?n=9E7AC85F1954DDA8&s=FC59ECFAA68D5509>

### 未解問題

- 教育部數位學伴結束後的承接方案（數位學習精進方案/AI 學習方向）是否留有可借力的志工池、夥伴學校名單或裝置資源？原 JoinNet 帶班老師網絡如何接觸？

- KIST 學伴實際使用的視訊工具與 BigByte 提供 RAZ-Kids 的授權安排細節（官網未載明），能否直接向誠致請益其安全與工具選型經驗？

- Excalidraw 官方協作通道為 E2E 加密（金鑰在 URL fragment），平台端要留白板互動稽核紀錄需自行旁錄——嵌入模式下改用自建同步層（Yjs）取代 excalidraw-room 的工程量與 50 併發資源需求，需 PoC 驗證。

- Minecraft Education 多人模式的文字聊天能否由管理端全域停用或完整記錄（未查證到官方文件）；台灣立案公益協會有無可能被 Microsoft 認定為 eligible academic institution 適用 US$5.04 價？

- RAZ-Kids／任何第三方教材帳號將兒少個資交給境外服務商的法遵影響（個資法、兒少權法）——屬另一研究維度，但直接決定教材路線取捨。

- 公有領域/CC 授權中文兒童讀物的實際供給量與品質（國資圖、教育部因材網、CC 繪本）——決定自建共讀器上線初期的內容庫是否夠用。

# 第二部：審查員指出的缺口與補查

> **缺口：部署模型未定：機構端合作實況與兒少端上線場景（據點集中 vs 在家單獨）**——六個維度存在一個未被任何人正面處理的矛盾：維度四（法規/安營）建議早期「兒少端由合作機構據點集中上線或家長在場」作為安全底座，但維度一的網路估算（偏鄉家用 4G）與維度六的裝置估算（家用 4GB 舊平板）全都假設孩子在家單獨上線。部署模型是地基級決策——它決定安全架構強度、家長同意書內容、網路與裝置需求、甚至 LLM 篩查是第一道還是第二道網；而「合作機構是誰、據點有無裝置/網路/陪同人力、導入意願與成本」六個維度完全沒查。沒有這塊，第一版方案的所有技術選型都懸在未驗證的假設上。

> **缺口：志工端供給完全未查：招募管道、留存先例與 12 小時培訓素材來源**——平台的另一半是志工，但六個維度只查了志工的「法遵管理」（志願服務法備案、基礎 6hr＋特殊 6hr 訓練、查核義務），從未查證：招募管道能否撐起 50 對的規模、遠距伴讀志工的留存率與 no-show 先例、以及那 12 小時法定訓練有沒有現成公版課程素材可直接用（維度四只說「特殊訓練時數用來上自製課」，但自製 6hr 兒少保護課程對一人團隊是重大工作量）。志工招不到或三個月就流失，技術架構再完整也開不了服務——這是供給側的 go/no-go 問題。

> **缺口：整體成本與營運量能模型不存在：儲存費被三個維度互踢、人工覆核人力從未估算、保存期限矛盾未整合**——維度一、二、五都把儲存/長期成本「留待整體成本維度另行估算」，但那個維度不存在——全程存檔是本案合規核心，其穩態成本卻無人算過。更關鍵的是人力：維度三的架構把人定為終審、維度四要求 24 小時責任通報，但「每週 N 場產生多大覆核佇列、一人團隊撐到第幾對會爆」從未推算。另有一個跨維度矛盾待整合：維度四建議自訂 90 天保存期，同時又指出性剝削條例 §8 的 180 日資料保留會覆蓋自訂刪除期限，儲存成本與刪除排程設計必須先解掉這個矛盾。對極低預算的一人公益專案，這張總表就是第一版方案的 go/no-go 判準。

## 補查：部署模型（機構端合作實況與兒少端上線場景：據點集中 vs 在家單獨）

**結論**：查證結果強烈支持「據點集中上線」為首版預設部署模型：教育部數位學伴計畫 20 年、及其 2026 年承接方案 AI Di+ 的官方模式，就是「學校教室集中上線＋帶班教師在場＋用學校既有載具與網路」，且該計畫明文不補助電腦設備——政府自己都把據點既有設備當前提。家庭端實況打臉「在家＋家長在場」假設：偏鄉學童 85.9% 有個人 3C（多為手機），但線上學習期間僅約 33.3% 有專用電腦/平板，弱勢家庭 60.6% 無電腦或平板，且偏鄉 24.6% 家長不限制上網、44.5% 兒少未與家長討論使用規則——「家長在場」在隔代教養與家長外出工作的偏鄉大量不成立。台灣現成的兒少課後據點生態（博幼 17 中心、家扶 76 據點、1919 陪讀班 233 班、DOC 與數位好厝邊約百餘個有設備的數位據點）已具備場地、時段與現場人力，缺口主要在裝置/網路盤點與總會層級的合作洽談。

**建議**：建議 bandu 第一版把「據點集中上線」定為預設部署模型，並據此回改技術選型：網路按「據點共享頻寬、5-25 路併發純語音」估算（非家用 4G）、裝置按「據點既有電腦/平板＋bandu 配發耳機麥克風」估算（仿 AI Di+ 由平台端配發輔助器材、不碰主機的分工）、LLM 篩查明確定位為第二道網（第一道是現場成人）；「在家＋家長在場」降為例外路徑，需個案審核家中專用裝置、固網/穩定網路與實際在場成人。首發合作機構建議三路並進：(1) 課輔型 NGO 總會（首選 1919 陪讀計畫或博幼基金會）——固定課後時段、現場帶班人力、弱勢兒少名單俱在，總會單一窗口可「談一次、開 1-2 個試點據點」，缺口只在裝置（可循「一人一平板」企業捐贈先例補位）；(2) 原數位學伴夥伴國中小——用 data.gov.tw 媒合清單（dataset 31855）比對 115 年 AI Di+ 核定名單，鎖定「有 20 年遠距伴學文化與帶班老師、但未進 AI Di+」的學校，以「興趣陪伴」定位與 AI Di+ 的學科學習區隔；(3) DOC／數位好厝邊作為裝置與場地的疊加合作方，優先找與課輔班共址的據點。MOU 需載明：志工保險分工（志願服務法法定）、家長同意書含語音存檔授權（可參照 AI Di+ 附件 1-1 的個資＋錄影授權格式）、現場人力與通報分工。此建議的核心理由：政府同構場景 20 年的收斂解就是據點集中＋成人在場，而家庭端數據（僅 1/3 有專用學習載具、家長監管大量落空、孩子手邊 7 成有自己的手機）證明「在家模型」會把 bandu 最脆弱的一道 LLM 網變成唯一的網。

### 查證細項

- **政府先例：數位學伴／AI Di+ 的部署模型就是「據點集中＋帶班教師在場」**（信心：high）
  【事實】教育部數位學伴計畫自 95 年起運作、累計服務 18,939 位學童，官方描述明載上課時「帶班老師在現場」掌握學習狀況並排除技術問題；計畫於 114 年（2025）底結束，由「115-116 年 AI Di+ 實驗方案」承接（期程 115/2/1–116/12/31，1 位大學伴對 4 位小學伴、每週 2 次、每堂 45 分鐘、放學後至最遲 20:30）。AI Di+ 偏遠地區國中小實施計畫全文明定：小學伴於「電腦教室空間或教室，集中管理」上線；「於國民中小學教室現場配置帶班教師」（支付帶班費、可為外聘人員）；申請人數不得超過學校既有學習載具台數；「本計畫不提供額外的電腦設備補助，大學教學端也不提供電腦設備支援」；須於計畫書敘明教室網路穩定度；縣市教育網路中心須保障上課時段學校網路正常；家長同意書（附件 1-1）含學生個資與「教學錄影檔」蒐集使用授權。【意義】這是與 bandu 幾乎同構的場景（遠距成人志工×偏鄉兒少×課後視訊），政府 20 年的解是實體集中＋成人在場＋全程錄影授權——bandu 的語音存檔同意書有現成官方格式可援引。
  來源：<https://depart.moe.edu.tw/ED2700/News_Content.aspx?n=727087A8A1328DEE&s=52FEAB3043C37A59>、<https://www.scps.kh.edu.tw/upload/82/101_88027/115-116%20%E5%B9%B4%20AI%20D%20i+%20%E5%AF%A6%E9%A9%97%E6%96%B9%E6%A1%88%E5%AF%A6%E6%96%BD%E8%A8%88%E7%95%AB_1.pdf>、<https://etutor.moe.gov.tw/>、<https://etutor.moe.gov.tw/etutor/upfd/news/47f672a71f706d4d1d664e5a70f8da3f.pdf>

- **數位學伴結束後的承接動態與切入窗口**（信心：high）
  【事實】113 年數位學伴補助 22 所大學、123 所國中小/數位機會中心；105-113 年夥伴學校媒合清單為政府開放資料（data.gov.tw dataset 31855），可逐校取得縣市/鄉鎮/核定學童數。AI Di+ 承接後模式改為 1 對 4 小組、限 3-9 年級、偏遠地區學校優先，並明文排除「資源重複投入者」（已參與學習扶助、夜光天使等）；初審於 114/12/29–115/1/9 由縣市政府辦理。【推測】原 123 所夥伴校中未獲 AI Di+ 核定者=剛被「斷炊」但有 20 年遠距伴學文化與帶班老師人力的現成名單，是 bandu 最精準的首發切入對象；比對 114 年媒合清單與 115 年核定名單即可鎖定。另 bandu 定位為「興趣陪伴/伴讀」而非學科教學，可與 AI Di+ 的「指定學科學習」區隔，降低被視為資源重複的風險——此區隔是否被縣市政府接受屬未驗證推測。
  來源：<https://www.edu.tw/News_Content.aspx?n=0217161130F0B192&s=04BF2F87D21FA277>、<https://data.gov.tw/dataset/31855>、<https://www.scps.kh.edu.tw/upload/82/101_88027/115-116%20%E5%B9%B4%20AI%20D%20i+%20%E5%AF%A6%E9%A9%97%E6%96%B9%E6%A1%88%E5%AF%A6%E6%96%BD%E8%A8%88%E7%95%AB_1.pdf>

- **家庭端實況：有手機、缺學習載具、缺家長監管——「在家＋家長在場」的三重反證**（信心：high）
  【事實】兒福聯盟 2026 城鄉兒童生活福祉調查（2025/12–2026/2，偏鄉 1,162 份＋都會 768 份，國小 4-6 年級）：偏鄉學童 85.9% 擁有個人 3C（都會 73.6%）、71.7% 有個人手機（都會 44.4%）、每週上網 20.8 小時（都會 12.4 小時）；24.6% 偏鄉家長不限制上網時間、44.5% 兒少未與家長討論使用規則、近 3 成曾遭網友搭訕或提出要求；線上學習期間僅約 33.3% 偏鄉孩子有自己專用的電腦或平板，六成靠向學校借用或與家人共用。家扶 2021 調查：60.6% 弱勢家庭家中無電腦或平板；博幼：疫情期間高達 7 成服務個案無遠距設備，且孩子返家後常因隔代教養、親人外出打工而無人可問功課。偏鄉上網方式以「僅手機行動上網」為最大宗（數位機會調查系列）。114 年數位近用調查全國家戶連網率 93.4%、個人上網率 90.3%，但 113 年調查顯示屏東（81.2%）、雲林（80.9%）、澎湖（79.9%）墊底。【意義】維度一（家用 4G）與維度六（家用 4GB 舊平板）的假設對「一般偏鄉學童」部分成立、對「弱勢個案」多數不成立；且「家長在場」在目標族群（隔代教養/家長晚歸）是名存實亡的控制項，等於把 LLM 篩查逼成第一道也是唯一一道網。
  來源：<https://news.pts.org.tw/article/816479>、<https://udn.com/news/story/7266/9610232>、<https://www.ctwant.com/article/119552>、<https://www.businesstoday.com.tw/article/category/183035/post/202503280031/>、<https://hk.finance.yahoo.com/news/%E6%95%B8%E7%99%BC%E9%83%A8114%E5%B9%B4%E6%95%B8%E4%BD%8D%E8%BF%91%E7%94%A8%E8%AA%BF%E6%9F%A5-%E5%AE%B6%E6%88%B6%E9%80%A3%E7%B6%B2%E7%8E%8793-4-%E5%80%8B%E4%BA%BA%E4%B8%8A%E7%B6%B2%E7%8E%8790-3-020047251.html>、<https://moda.gov.tw/digital-affairs/digital-service/dv-survey/14516>

- **候選機構盤點 A：課輔型 NGO（博幼／家扶／1919 陪讀班）——有場地時段人力，缺裝置網路盤點**（信心：high）
  【事實】博幼基金會：17 個課輔中心（新竹、台中、南投、彰化、雲林、嘉義、屏東、宜蘭、台東、澎湖），每天陪伴 3,600+ 位孩子、週一至五每天 2-3 小時固定課輔時段。家扶基金會：全台 76 個據點、有成熟課輔志工招募體系。基督教救助協會 1919 陪讀計畫：113 學年度 233 個陪讀班、服務 2,500 位弱勢兒少，依托全台 1,300+ 個教會 1919 服務中心、深入 296 鄉鎮，提供課輔＋晚餐＋家庭關懷；疫情期間四公益團體（家扶、展望會、救助協會、博幼）曾聯合發起「一人一平板」募得 619 台平板（台灣大哥大捐 100 台）——顯示其據點端裝置本非現成、但有企業捐贈補位的先例。【推測】這三類機構的固定課後時段（約 16:00–20:00）與現場帶班人力和 bandu 需求完全同構，導入的新增成本主要是裝置/網路（需逐據點盤點）與行政（同意書、名單管理），而非人力；總會層級單一窗口（博幼總會管 17 中心、救助協會總會管 233 班）使「談一次、多點展開」可行。
  來源：<https://www.boyo.org.tw/>、<https://www.businesstoday.com.tw/article/category/183035/post/202503280031/>、<https://www.ccf.org.tw/>、<https://www.igiving.org.tw/npo/introduce?n_id=424>、<https://www.ccra.org.tw/SviAriticlePage.aspx?SVSID=1>、<https://www.ccra.org.tw/AriticlePage.aspx?NWSID=1625>、<https://www.ctwant.com/article/119552>

- **候選機構盤點 B：數位據點型（DOC／數位好厝邊）與人網型（TFT／公益平台）**（信心：medium）
  【事實】教育部「偏鄉數位培力推動計畫」數位機會中心（DOC）：偏鄉據點配有桌機、平板、無線網路（如台東關山 DOC 案例），2025 上半年統計 184 個協力據點、偏鄉教學服務觸及 1,101 人次；DOC 本來就是數位學伴計畫的小學伴端場域之一。中華電信基金會「數位好厝邊」：約 90-91 個社區據點，由中華電信建置數位設備與寬頻資源，服務對象含兒少教育類社福團體。TFT 為台灣而教：累計 85 所合作國小、跨 9 縣市、370+ 計畫成員——是教師「人」的網絡而非據點，適合當轉介與在地信任橋樑。公益平台文化基金會：花東為主的策略樞紐（均一學校、師培中心、英語悅讀計畫與教會/國小合作），非量產據點型。【推測】DOC 與數位好厝邊「裝置網路現成、但兒少個案關係較弱」，與課輔型 NGO「個案關係強、裝置弱」恰好互補；最省錢的試點是找兩者重疊處（如設在教會/社區協會裡的 DOC 或好厝邊據點同時辦課輔者）。
  來源：<https://itaiwan.moe.gov.tw/plan_info.php?id=18>、<https://itaiwan.moe.gov.tw/statistics.php>、<https://twfriendlyliving.blogspot.com/2024/03/blog-post_22.html>、<https://www.chtf.org.tw/page/713>、<https://www.cht.com.tw/zh-tw/home/cht/esg/social-inclusion/digital-inclusion/bridging-the-digital-divide>、<https://www.teach4taiwan.org/about/become_school_partner/>、<https://www.thealliance.org.tw/service/1623>

- **學校端裝置底座：生生用平板 111-114 年到期、後續未見核定公告**（信心：high）
  【事實】「推動中小學數位學習精進方案」（班班有網路、生生用平板）期程 110/12–114/12、總經費 200 億元，偏遠地區學校學生 1 人 1 機、非偏遠依班級數 1/6 配發，並升級校園網路。方案已於 114 年 12 月到期，搜尋未找到 115 年後延續方案的行政院核定公告。AI Di+ 實施計畫要求以學校既有載具上課且不另補助設備，佐證校端平板存量在 115-116 年仍被政府當作可用前提。【推測】若走「學校據點」模型，偏遠校的平板與校網大概率可用（AI Di+ 同款前提），但平板屬校產、能否供「校外 NPO 平台」的活動使用涉及各校財產管理規定，未查得通案規範——這是學校路線相對課輔 NGO 路線多出的一道行政關卡。
  來源：<https://www.ey.gov.tw/Page/448DE008087A1971/0ba98487-02aa-4bf3-afa5-89a628e29c9d>、<https://www.edu.tw/News_Content.aspx?n=9E7AC85F1954DDA8&s=9F7133D453CC16F2>、<https://www.scps.kh.edu.tw/upload/82/101_88027/115-116%20%E5%B9%B4%20AI%20D%20i+%20%E5%AF%A6%E9%A9%97%E6%96%B9%E6%A1%88%E5%AF%A6%E6%96%BD%E8%A8%88%E7%95%AB_1.pdf>

- **機構端法定配置：志工保險與教育訓練是導入時的固定成本項**（信心：medium）
  【事實】志願服務法明定運用單位須為志工辦理意外事故保險（法定必要）、志工有接受教育訓練之權利與義務；衛福部並有志工保險的集中採購機制。AI Di+ 的帶班教師「支付帶班費」設計顯示現場陪伴人力在政府計畫中是明碼標價的成本項而非免費資源。【推測】bandu 與機構的 MOU 至少需載明：志工保險歸屬（bandu 端遠距志工 vs 機構端現場人力各自投保）、家長同意書（含語音存檔授權，可參照 AI Di+ 附件 1-1 格式）、現場人力配置與出缺席責任、可疑事件通報分工。志工的性侵害犯罪紀錄查核（兒少工作人員法定查核）由立案機構執行的前提在本次搜尋未逐條查核條文，維持原提案假設、標記為待維度四交叉確認。
  來源：<https://law.moj.gov.tw/LawClass/LawAll.aspx?PCode=D0050131>、<https://vol.mohw.gov.tw/vol2/lawtask/download/268441013>、<https://www.scps.kh.edu.tw/upload/82/101_88027/115-116%20%E5%B9%B4%20AI%20D%20i+%20%E5%AF%A6%E9%A9%97%E6%96%B9%E6%A1%88%E5%AF%A6%E6%96%BD%E8%A8%88%E7%95%AB_1.pdf>

- **兩模型可行性對照表（50 對規模）**（信心：medium）
  【推測，基於上述事實的分析】■ 模型A 據點集中上線（學校電腦教室／課輔班／DOC）：機構端導入成本＝場地裝置網路多為既有（校平板+校網、DOC 電腦室、好厝邊設備；課輔 NGO 需補裝置）＋現場陪伴人力（真實成本，AI Di+ 帶班費先例；課輔 NGO 可用既有帶班人力吸收）＋行政（同意書/名單/出缺席）。安全性＝實體成人在場為第一道防線，LLM 篩查退居第二道網；裝置受機構控管（無 LINE/私訊 App），「脫離平台」誘導在現場極難得逞。50 對擴充性＝每據點 8-25 孩子，2-6 個據點即滿足，行政集中、頻寬按據點共享估算（每據點 5-25 路併發語音，頻寬需求低）。弱點＝時段被鎖在放學後至約 20:30、寒暑假可能中斷、導入前置談判期長。■ 模型B 在家＋家長在場：成本全數轉嫁家庭＝專用學習載具僅約 1/3 家庭有、上網以手機行動網路為主（品質不穩）、「家長在場」在隔代教養/家長晚歸族群大量落空（44.5% 連使用規則都沒討論過）。安全性＝全壓在平台技術管制＋LLM 篩查（被迫成為第一道網），且孩子 71.7% 自有手機在手邊＝脫離平台管道天然存在、平台管不到。擴充性＝理論無據點上限，但 50 對就要做 50 戶的裝置/網路/同意/在場查核個案管理，早期反而更貴。結論：模型 A 在安全、成本、可驗證性全面佔優；模型 B 應降級為「經個案審核的例外路徑」而非預設。
  來源：<https://www.scps.kh.edu.tw/upload/82/101_88027/115-116%20%E5%B9%B4%20AI%20D%20i+%20%E5%AF%A6%E9%A9%97%E6%96%B9%E6%A1%88%E5%AF%A6%E6%96%BD%E8%A8%88%E7%95%AB_1.pdf>、<https://news.pts.org.tw/article/816479>、<https://www.ctwant.com/article/119552>、<https://www.businesstoday.com.tw/article/category/183035/post/202503280031/>

### 未解問題

- 生生用平板 115 年後是否有延續方案（行政院核定公告未查得）；校產平板可否用於校外 NPO 平台活動，各校財產管理規定是否通案允許

- 1919 陪讀班與博幼各據點的實際網路頻寬、可用裝置數與插座/空間條件——無公開清冊，需與總會接洽後實地盤點 1-2 個候選據點

- AI Di+ 的「排除資源重複投入」條款是否會被縣市政府解讀為排斥同校導入 bandu；「興趣陪伴 vs 學科學習」的區隔論述是否被接受

- 原 123 所數位學伴夥伴校中未進入 115-116 AI Di+ 的名單（最佳切入對象）——需比對 114 年媒合清單與 115 年核定名單，後者尚未查得公開版本

- 據點模式下語音存檔的同意鏈設計：家長同意書由機構代收（AI Di+ 模式）是否滿足個資法委託處理要求，需與維度四（法規）交叉確認；志工性侵害犯罪紀錄查核由立案機構執行的具體法源與流程亦待維度四確認

- 兒盟 2026 調查的「偏鄉」定義與 bandu 目標族群（弱勢個案）的重疊度——85.9% 自有 3C 是全體偏鄉學童數字，弱勢子集的設備缺口（家扶 60.6%、博幼 7 成）明顯更大，兩組數字的母體差異需在需求估算時分開處理

- 寒暑假與週末時段：課輔據點多隨學期運作，bandu 若要提供假期陪伴，據點模式的空窗如何補位（教會系統的 1919 服務中心可能是少數假期仍運作的據點型態，未查證）

## 志工端供給補查：招募管道與規模、12 小時法定培訓素材來源、留存與 no-show 先例、50 對配對之志工漏斗推估

**結論**：查證發現台灣並不存在「純無償、固定週次」的遠距課輔志工先例：教育部數位學伴（唯一大規模遠距一對一先例，22 校、每年 1,600 餘位小學伴）每節付大學伴 549 元獎勵金且僅限在學學生，博幼、永齡、TFT 全是支薪模式——bandu 鎖定「下班社會人士無償伴讀」是走沒人走過的路，供給風險真實存在但 50 對的絕對規模很小（約為數位學伴全國量的 3%）。法定 12 小時訓練的負擔遠比審查員假設的輕：基礎 6hr 有臺北e大免費公版線上課（270 分鐘＋測驗即發證明、各縣市採認），特殊 6hr 課綱是衛福部法定 4 課目（非自由發揮的兒少保護課），真正必須自製的只有「運用單位業務簡介＋綜合討論」約 3hr，兒少保護內容可用國教署公版 PPT、iWIN、e等公務園+ 等免費政府素材補強。留存先例（BBBS 社區型 12 個月內 34% 提前結案、學校型 46%；Reading Partners 學生實得節數僅排定的 75% 左右、靠全職站點協調員吸收志工缺勤）支持的漏斗推估：要撐住 50 對穩定配對，首學期約需 130–150 名報名者、82 名完訓志工（含替補池），且每學期需補充招募約 3 成。

**建議**：供給側結論是「有條件 go」，不是 no-go，但前提有三個。第一，不要直接開 50 對：台灣沒有任何「純無償＋固定週次」遠距伴讀的規模化先例（數位學伴付 549 元/節且限學生，博幼/永齡/TFT 全是支薪），bandu 等於在做供給側實驗——先以 10–15 對跑一學期實測「報名→完訓轉換率」與「首學期留存」兩個本次查不到台灣數據的參數，再決定擴到 50 對；漏斗推估（50 對需首學期 130–150 名報名）在 MVP 數據出來前只能當規劃上限。第二，培訓不是瓶頸，別把工放在這：法定 12hr 採「臺北e大基礎 6hr 線上公版（零成本）＋縣市特殊訓練場次或自辦（法定 4 課目中僅約 3hr 必自製，且內容就是介紹自己平台）」，兒少保護用國教署公版 PPT＋iWIN 免費講師場次＋e等公務園課程拼裝，真正要寫的只有 2–3hr 的「脫離平台訊號辨識＋通報 SOP＋行為準則」——這份教材與你的 LLM 篩查規則同源，寫一次兩用。第三，用設計吸收 no-show 而非用紀律對抗：預算允許時保留「每節小額獎勵金或至少交通/設備補貼」的選項（若純無償使轉換率腰斬，招募量需求近 200 人）；機構端配 1–2 名可代打的帶班人力（Reading Partners 模式）；對無法固定承諾者開 UPchieve 式彈性替補池。招募管道優先序：企業志工/ESG 團體報名（社會人士、整批進、自帶雇主背書）＞大學服務學習合作（可折學分，複製數位學伴誘因，但與「社會人士」定位部分偏離）＞衛福部志願服務網張貼（合規必用、但別指望量）。

### 查證細項

- **招募規模基準：教育部數位學伴是唯一可類比的遠距一對一先例**（信心：high）
  【查證事實】數位學伴計畫每年補助 20 餘所（113 年為 22 所）夥伴大學，服務 123 所國中小/數位機會中心、1,600 餘位小學伴，採線上即時一對一、每學期 10 週、每週 2 次每次 90 分鐘；自 95 年（2006）起累計陪伴 18,939 位學童。大學伴以 1:1 配對推算每年全國約需 1,600–2,000 名。招募由各夥伴大學辦理，資格限「專科、大學、碩博士在學學生，不限科系年級」。【推測】bandu 的 50 對約為全國數位學伴量能的 3%，絕對規模不大；但 bandu 目標族群（下班社會人士）與此管道（在學學生）不重疊，數位學伴模式只能證明『遠距一對一課輔在台灣可運作』，不能證明社會人士招得到。
  來源：<https://depart.moe.edu.tw/ED2700/News_Content.aspx?n=727087A8A1328DEE&s=52FEAB3043C37A59>、<https://www.edu.tw/News_Content.aspx?n=9E7AC85F1954DDA8&s=7319A0A5542F79BA>、<https://etutor.moe.gov.tw/zh/about/intro>、<https://c028.wzu.edu.tw/article/516884>

- **關鍵發現：連數位學伴都不是純志工——每節付 549 元或折服務學習時數**（信心：high）
  【查證事實】2024 年數位學伴大學伴招募簡章載明：每次服務（1.5 小時同步教學＋備課＋教學日誌）發給獎勵金 549 元，或可擇一改領服務學習時數/技能實習時數/課程專題認列；服務滿 10 週且全程參與培訓者發給中英文教學服務證書，部分學校可折抵必修服務學習時數。【推測】這是 bandu 供給側最重要的訊號：台灣現行制度下，穩定週次的遠距伴讀『志工』實際上是低額有償＋學分誘因的複合體。bandu 若堅持純無償，等於拿掉此管道兩個主要誘因；若對接大學服務學習學分（與機構簽服務學習合作），可部分複製此誘因結構，但社會人士端沒有等價物。
  來源：<https://c028.wzu.edu.tw/article/516884>、<https://holistic.sa.ntnu.edu.tw/%E6%95%B8%E4%BD%8D%E5%AD%B8%E4%BC%B4%EF%BD%9C%E5%9C%8B%E7%AB%8B%E8%87%BA%E7%81%A3%E5%B8%AB%E7%AF%84%E5%A4%A7%E5%AD%B8x%E6%95%99%E8%82%B2%E9%83%A8%E6%95%B8%E4%BD%8D%E5%AD%B8%E4%BC%B4%E8%A8%88%E7%95%AB1/>

- **博幼／永齡／TFT 全是支薪模式，年報不揭露流失率——純志工課輔在台灣缺乏規模化先例**（信心：high）
  【查證事實】永齡希望小學：與 15 所大學＋2 個 NPO 合作、280 所國小、課輔教師達 1,500 位，但課輔老師是支薪職位（各分校鐘點費約 150–400 元/時，雲科大分校 400 元/時含勞健保、意外險與交通補助）。博幼基金會：近千位老師在 35 鄉鎮服務 3,600+ 位孩子，模式是聘用當地大學生與部落媽媽，時薪制（早期約百餘元/時）或月薪制（課輔老師平均月薪約 33.2k）。TFT 為台灣而教：兩年全職計畫，第 13 屆總名額 91 位，兩年薪津合計約 114.95 萬（月均 44k+），歷史錄取率約 6.5–7%。三者公開年報/網頁均未揭露教師年流失率。【推測】審查員點名的三個組織實際上都不是志工組織——台灣課輔供給側的既有答案是『在地聘僱＋支薪』，這反向印證純無償模式的招募與留存風險，也解釋為何查不到台灣志工課輔流失率數據：因為幾乎沒有人用純志工做這件事。
  來源：<https://multimedia.nfu.edu.tw/news/6/55798>、<https://tec.ntou.edu.tw/p/406-1019-17969,r535.php?Lang=zh-tw>、<http://www.tomorrowschool.org/location.htm>、<https://blog.104.com.tw/boyo-visits-rural-education/>、<https://salary.tw/c/niSw/positions/%E8%AA%B2%E8%BC%94%E8%80%81%E5%B8%AB>、<https://www.boyo.org.tw/boyoV2/what_are_we_do>、<https://www.teach4taiwan.org/recruitment/>、<https://www.cw.com.tw/article/5096675>

- **衛福部志願服務資訊網：可用的免費招募張貼管道，但媒合量能無公開數據**（信心：medium）
  【查證事實】衛福部志願服務資訊網（vol.mohw.gov.tw）提供志工召募訊息刊登、全國運用單位查詢、志工時數紀錄與年度統計表下載（113 年度各縣市志願服務業務成果統計表可下載）；社福類運用單位每年 1/5、7/5 兩次填報志工人數與服務成果。【推測】此網是合規基礎設施（bandu 合作機構作為運用單位本來就要用它管理志工與時數），但作為『招募漏斗頂端』的實際觸及與轉換量能沒有任何公開數據，不能指望它成為主要招募來源；主要來源仍需自建（企業志工/ESG、大學服務學習、社群）。
  來源：<https://vol.mohw.gov.tw/>、<https://vol.mohw.gov.tw/vol2/statistical/index.gsp>、<https://www.kvc.org.tw/unit_message_Content?stype=2&id=223>

- **基礎訓練 6hr：有免費公版線上課，零自製成本**（信心：high）
  【查證事實】衛福部 2018/6/1 起將志工基礎訓練修正為 3 課目 6 小時（志願服務內涵及倫理、志願服務經驗分享、志願服務法規之認識）。臺北e大提供「[志願服務]志工基礎教育訓練」免費線上課程：上線時間滿 270 分鐘＋測驗達 70 分即核發 6 小時學習時數證明，可列印學習證明；嘉義市、雲林縣、臺北市社會局及多所大學均公告引導志工用此管道完訓，屬廣泛採認的公版數位課程。【推測→授權判斷】此為政府平台開放註冊課程，bandu 志工自行上網完訓即可，無授權問題、無自製工作量；唯需 bandu 合作之運用單位（立案機構）確認採認並登錄紀錄冊——各縣市普遍採認，風險低。
  來源：<https://dep.mohw.gov.tw/dosaasw/cp-530-41204-103.html>、<https://social.chiayi.gov.tw/volunteer/News_Content.aspx?n=4009&s=772243>、<https://stud.ntub.edu.tw/p/406-1007-60136,r1.php?Lang=zh-tw>、<https://dosw.gov.taipei/News_Content.aspx?n=1AD7DB73BB4F1A07&sms=EE18AA70CEE11481&s=ABC0317D1CEE9144>

- **特殊訓練 6hr：課綱是法定 4 課目，必自製部分只有約 3 小時——審查員的前提需修正**（信心：high）
  【查證事實】衛福部 2018/6/1 修正社會福利類志工特殊訓練為 4 課目 6 小時：社會福利概述（2hr）、社會資源與志願服務（1hr）、運用單位業務簡介及工作內容說明含實習（2hr）、綜合討論（1hr）；前兩課目可參加縣市政府/志願服務推廣中心每年開辦之場次，臺北e大亦有特殊訓練線上課程（雲林縣推廣中心公告含基礎＋特殊訓練線上教學，特殊訓練至少有高齡版）。【推測】原維度四『特殊訓練 6hr 拿來上自製兒少保護課』的假設不成立——特殊訓練課綱是法定的，不能整包換成自製兒少保護課；真正落在 bandu/合作機構頭上的必自製內容是『運用單位業務簡介（2hr）＋綜合討論（1hr）』約 3 小時，且這本來就該講自己平台的規則與流程，工作量遠小於自製 6hr 課程。兒少保護內容應定位為法定 12hr 之外的加值訓練模組。
  來源：<https://dep.mohw.gov.tw/dosaasw/cp-530-41204-103.html>、<https://social.yunlin.gov.tw/News_Content.aspx?n=737&s=256277>、<https://tylcvsc.yunlin.gov.tw/News_Content.aspx?n=6546&s=289956>、<https://www.pthg.gov.tw/ptvols/News_Content.aspx?n=D71E5F03F39E2F64&sms=CFAB4DC725BD3DDA&s=04958A0627A04FAA>

- **兒少保護加值模組素材清單：政府公開素材充足，真正的自製缺口約 2–3 小時**（信心：medium）
  【查證事實＋逐項授權判斷】(1) 教育部國教署「兒少性剝削防制條例」宣導公版 PPT、懶人包、教案示例（含國小組『防制網路兒少性剝削』教案）——公開下載，政府宣導教材供教育訓練使用，可直接採用，約可支撐 1–2hr。(2) 衛福部保護服務司宣導文宣/影音專區（含兒少性影像修法懶人包）——公開下載可用，0.5–1hr。(3) iWIN 網路內容防護機構——提供網路交友、私密照外流、網路霸凌等宣導教材，且每年至少 25 場免費宣導可申請講師到場（可直接把一堂培訓外包給 iWIN），1–2hr。(4) e等公務園+（elearn.hrd.gov.tw）——一般民眾可註冊修課並取得認證時數（閱讀時間過半＋測驗 60 分＋問卷），已確認有兒少福權法修法、脆弱家庭辨識與通報等 1hr 課程，具體兒少保護課程清單需上平台逐一確認。(5) 教育部中小學數位素養教育資源網/全民資安素養網——網路安全素材公開。(6) 數位學伴大學伴培訓教材——未公開，需向夥伴大學或計畫辦公室洽詢授權（未查證到可取得性）。【推測】無法用公版覆蓋、必須自製的缺口：bandu 平台專屬的『脫離平台訊號辨識（加LINE/私下聊/不要跟爸媽說）＋通報 SOP＋志工行為準則（界線、單獨互動規範）』約 2–3 小時——這是 LLM 篩查規則的人工訓練對應版，本來就只有 bandu 自己能寫。
  來源：<https://www.k12ea.gov.tw/Tw/Common/DownloadDetail?filter=043F1AEB-690C-4F45-924E-D1C90AE7694E&id=b615cf90-a3e0-45a7-b3d6-aeffcf87dad3>、<https://www.mjes.ntpc.edu.tw/p/406-1000-1564,r23.php>、<https://www.tc.edu.tw/cms-file/6436620db8bdf5040c1cb492.pdf>、<https://dep.mohw.gov.tw/dops/lp-1269-105.html>、<https://dep.mohw.gov.tw/dops/cp-1269-79607-105.html>、<https://i.win.org.tw/propaganda.php?Type=5>、<https://elearn.hrd.gov.tw/>、<https://www.pwr.org.tw/page/1499>、<https://eliteracy.edu.tw/Material.aspx?id=2816>、<https://isafe.moe.edu.tw/kids>

- **留存先例：一對一青少年 mentoring 12 個月內 34–46% 提前結案；大學生 mentor 風險更高**（信心：high）
  【查證事實】Big Brothers Big Sisters 社區型計畫研究（DeWit et al. 2016, American Journal of Community Psychology）：34% 的配對在 12 個月承諾期內提前結案；學校型計畫 12 個月內 46%；更早期的全國性報告約 55% 提前終止。風險因子研究並發現：配對到「大學生 mentor」的關係更容易提前結案，而有先前輔導經驗的 mentor 關係較穩固。另 Science.gov 彙整的志工課輔研究提到某志工課輔計畫年流失率 27%『屬正常範圍』（彙整頁、原始研究未逐一核對，證據力較弱）。【推測】對 bandu 反而是雙面訊息：主力若是社會人士而非大學生，留存可能優於 BBBS 的大學生子群；但 BBBS 是有專職個管員支持的面對面關係，純線上＋輕監督的流失只會更高不會更低。以學期（約 4 個月）為單位，首學期配對存活率抓 65–70% 是有依據的保守假設。
  來源：<https://pubmed.ncbi.nlm.nih.gov/27217312/>、<https://onlinelibrary.wiley.com/doi/full/10.1002/ajcp.12023>、<https://www.evidencebasedmentoring.org/researchers-investigate-why-mentoring-matches-end-early/>、<https://www.cebmentoring.org/wp-content/uploads/2017/03/American-J-of-Comm-Psychol-2017-Kupersmidt-Predictors-of-Premature-Match-Closure-in-Youth-Mentoring-Relationships.pdf>、<https://www.science.gov/topicpages/v/volunteer+tutoring+programs>

- **no-show 管理先例：Reading Partners 靠全職站點協調員吸收缺勤；UPchieve 用 on-demand 設計迴避固定承諾**（信心：high）
  【查證事實】Reading Partners（MDRC 隨機對照評估）：每站 40–100 名社區志工＋1 名全職 AmeriCorps 站點協調員管理日常運作並可親自代課；學生排定每週 2 次但實得平均約 1.5 次/週（≈25% 節數落差），缺課靠補課機制吸收，評估認定在『直接服務者是志工』的先天困難下仍維持高執行忠實度。協調員自述春季補課需求隨學年遞增。UPchieve（美國免費線上一對一輔導）：10,000+ 志工、無最低時數承諾、on-demand 接單制（學生發需求、志工自選是否接）——用產品設計直接消滅 no-show 概念；其網站稱 90%+ 幫過一個學生的志工會繼續幫兩個以上。Outschool：教師留存率無公開數據，且是付費市場、教師是收入方，非志工類比，不建議引用。【推測】對 bandu 的兩個可移植機制：(a) 機構端設一名可代打的『帶班人力』（對應 site coordinator），50 對規模約需 1–2 名；(b) 對無法承諾固定週次的志工開 UPchieve 式彈性池（非固定配對的陪讀/代打），把 no-show 從紀律問題變成排班問題。
  來源：<https://www.mdrc.org/work/publications/mobilizing-volunteer-tutors-improve-student-literacy>、<https://www.americorps.gov/evidence-exchange/Mobilizing-Volunteer-Tutors-to-Improve-Student-Literacy:-Implementation-Impacts-and-Costs-of-the-Reading-Partners-Program>、<https://readingpartners.org/blog/a-day-in-the-life-of-a-reading-partners-americorps-volunteer-coordinator/>、<https://upchieve.org/volunteer-faqs>、<https://upchieve.org/annual-report-2024>

- **志工漏斗推估：50 對穩定配對 ≈ 首學期 130–150 名報名、82 名完訓（含替補），每學期補招 3 成**（信心：low）
  【推測（假設均明列）】目標：50 對撐過一個學期（約 16–20 週，對齊數位學伴 10 週制可放寬）。反推：(1) 首學期配對存活率假設 65–70%（依據 BBBS 社區型 34%/12 個月提前結案折算到 4 個月＋線上無面對面連結的懲罰項），→ 學期初需約 72 名已配對志工；(2) 替補池 15%（依據 Reading Partners 節數落差 ≈25% 與補課機制），→ 共需約 82 名完訓志工；(3) 報名→完成法定 12hr 訓練＋機構查核（含性侵害/兒少法定查核等待期）轉換率假設 55–65%——此環節無台灣公開數據，係參考一般志工 onboarding 流失與 12hr＋查核的摩擦成本所設，是整個漏斗最不確定的一環，→ 需約 130–150 名有效報名。(4) 穩態維運：每學期流失 25–35% → 每學期需補招 15–20 名完訓志工（約 25–35 名報名）。敏感度：若純無償導致轉換率掉到 40%，報名需求升至 ~200 名——這就是『是否保留小額獎勵金/交通費選項』的量化理由（數位學伴的市場價：549 元/節）。
  來源：<https://pubmed.ncbi.nlm.nih.gov/27217312/>、<https://www.mdrc.org/work/publications/mobilizing-volunteer-tutors-improve-student-literacy>、<https://c028.wzu.edu.tw/article/516884>、<https://social.chiayi.gov.tw/volunteer/News_Content.aspx?n=4009&s=772243>

### 未解問題

- 台灣情境「報名→完成 12hr 訓練＋查核」轉換率完全無公開數據，漏斗中 55–65% 為推測——MVP 首學期必須實測，這是擴大到 50 對前的 go/no-go 硬數據。

- 教育部數位學伴的大學伴培訓教材（線上教學法、班級經營、兒少互動）未公開，能否向夥伴大學或計畫辦公室索取及其授權條件未查證。

- 臺北e大特殊訓練線上課程目前確認有「高齡版」，是否有適用一般社福運用單位的通用版、以及 bandu 合作機構所在縣市是否採認全線上特殊訓練，需向該縣市志願服務推廣中心確認。

- e等公務園+ 的兒少保護相關課程完整清單與各課時數需登入平台逐一確認（本次僅確認平台對一般民眾開放且有時數認證機制，及零星 1hr 課程）。

- 博幼/永齡的支薪課輔老師年流失率未公開，無法作為台灣本土留存基準；衛福部志願服務網 113 年度統計表（可下載 .ods）或含社福類志工總量，可再細查作為招募池規模佐證。

- 企業 ESG 志工團體對「每週固定晚間 1 對 1、需完成 12hr 法定訓練＋查核」這種高承諾型志工的實際接受度，本次未查證（多數企業志工偏好一次性活動），需訪談 2–3 家企業 CSR 窗口驗證。

## 整體成本與營運量能模型（補查：儲存/運算/人工覆核/法遵一次性成本，10/25/50 對三級距總表）

**結論**：全案雲端月成本在 50 對滿載下也只有約 NT$500–2,500（儲存費近乎為零：R2 免費額度就吃掉 10 對級距的全部錄音），錢完全不是 go/no-go 的約束條件。真正的斷點在人：以「規則層命中 5–10%＋LLM 高風險 1–3%＋單件覆核 10–20 分＋10% 抽查」推算，樂觀參數下單人綽綽有餘，但悲觀參數（每對每週 3 場、合計命中 13%、單件 20 分）下每週覆核時數 ≈ 0.2×對數，一人每週 10 小時的預算在約 29–49 對之間爆掉，且 24 小時通報在單人排班下是結構性單點故障（生病/出差即斷鏈），與規模無關。90 日自訂刪除 vs 性剝削條例 §8 的 180 日保留矛盾有乾淨的工程解：R2 lifecycle rule 對一般錄音 90 日自動刪除、命中案件搬移至無自動刪除的 legal-hold 前綴保留至 180 日或結案，成本影響僅每月數十元台幣。

**建議**：建議 go，起步規模定在 10–25 對，理由：(1) 錢不構成約束——全雲端月成本 10 對約 NT$270–1,240、50 對悲觀也不超過 ~NT$2,500，先前三個維度互踢的「儲存費」實測近乎為零（R2 免費額度直接吃掉 10 對級距，50 對也只要每月十幾元台幣），個人出資完全可承受；最大單筆現金支出是一次性法遵費，走「縣市免費法律諮詢＋兒少領域 NGO 諮詢定方向、自擬文件付費審閱」可壓到 NT$10,000–30,000。(2) 架構定案建議：儲存選 R2（recordings/ 前綴 90 日 lifecycle 自動刪除＋holds/ 前綴 legal-hold 保留 180 日至結案，一次解掉 90/180 矛盾）；VPS 首選遠振台灣 NT$990（HiNet 低延遲＋錄音不出境的說服力），預算敏感可退 Vultr 東京 $5–10；STT 用自有桌機 GPU 夜批 faster-whisper（電費 NT$50–90/月，兒少語音不出第三方）；LLM 篩查用 Gemini 2.5 Flash-Lite 全量＋Claude Haiku 4.5 Batch 複審，月費個位數美元。(3) 真正的 go/no-go 條件是人力而非錢：悲觀參數下單人每週 10 小時在 ~49 對爆、6 小時在 ~29 對爆，且 24 小時通報單人排班是與規模無關的單點故障——上 50 對前必須先落實「機構社工第二責任人/第二覆核者」的 MOU；前三個月以 10 對實測規則層與 LLM 的真實誤報率，用實測值重算斷點後再決定擴到 25 或 50 對。

### 查證細項

- **三級距月度總表（本補查的核心產出）**（信心：high）
  基準假設：每對每週2場×60分、分軌 Opus 32kbps（28.8MB/場）、1USD≈NT$32.3。【10對｜87場/月】儲存 R2 NT$0（免費額度內）；VPS NT$160（Vultr 東京 $5）～NT$990（遠振台灣）；STT NT$10–115（自有GPU電費～RunPod/Groq）；LLM 篩查 NT$7–35；月計約 NT$270–1,240；覆核 0.5–1.4 小時/週（含抽查）→ 最先爆項目：無，固定成本以 VPS 為大宗。【25對｜217場/月】儲存 ~NT$5；STT NT$25–280；LLM NT$16–90；月計約 NT$350–1,600；覆核 1–3.5 小時/週 → 尚無爆點，但悲觀參數下人力 ~5 小時/週已達單人可持續投入的一半。【50對｜433場/月】儲存 ~NT$15–25；STT NT$50–560；LLM NT$33–175；月計約 NT$500–1,700（悲觀不超過 ~NT$2,500）；覆核＋抽查 1.8–6.8 小時/週（基準）、悲觀排程 10.3 小時/週 → 最先爆項目：人工覆核人力（非成本）。另有一次性法遵成本 NT$25,000–110,000（可壓至 NT$10,000–30,000，見法遵項）。
  來源：<https://developers.cloudflare.com/r2/pricing/>、<https://www.vultr.com/pricing/>、<https://host.com.tw/%E5%8F%B0%E7%81%A3VPS%E4%B8%BB%E6%A9%9F>、<https://www.runpod.io/pricing>

- **儲存量估值驗證：Opus 32kbps ≈ 14.4MB/participant-hour，原估 10–20MB 成立**（信心：high）
  32kbps = 4KB/s × 3600s = 14.4MB/小時/軌（純算術，無損於封裝開銷約1–2%），落在題目給的 10–20MB 區間內。Xiph（Opus 官方 wiki）建議值：VoIP 單聲道語音 10–24kbps 即可（24kbps 已達 fullband）、fullband 語音 28–40kbps——32kbps 對「純語音伴讀」是偏寬裕的品質設定；若降到 24kbps（10.8MB/hr）或 16kbps 寬頻（7.2MB/hr）可再砍 25–50% 儲存量，但在本案儲存費近零的前提下無此必要。
  來源：<https://wiki.xiph.org/Opus_Recommended_Settings>、<https://opus-codec.org/>

- **三家物件儲存實際單價：R2 完勝本案場景**（信心：high）
  Cloudflare R2：$0.015/GB-月、egress 全免費、免費額度每月 10GB-月儲存＋1M Class A＋10M Class B 操作（本案每月僅數百次 PUT，操作費為零）。Backblaze B2：$6/TB-月（$0.006/GB）、免費 egress 至月均儲存量 3 倍。Wasabi：$6.99/TB-月，但有 1TB 低消（不足 1TB 也收 $6.99/月）＋90 天最短儲存期計費（提前刪除補收剩餘天數）。本案穩態最大僅 ~41–62GB：R2 月費 $0–0.5、B2 $0.05–0.4、Wasabi 固定 $6.99——R2 最便宜且 10 對級距完全免費。
  來源：<https://developers.cloudflare.com/r2/pricing/>、<https://www.backblaze.com/cloud-storage/pricing>、<https://wasabi.com/pricing>、<https://docs.wasabi.com/docs/how-does-wasabis-minimum-storage-duration-policy-work>

- **台灣存取延遲：B2 無亞太機房是唯一硬傷，但本案延遲敏感度低**（信心：medium）
  查證到的事實：Wasabi 有東京（ap-northeast-1）、大阪、新加坡機房；Backblaze B2 僅美西（Sacramento/Phoenix）、美東（Reston）、歐洲（Amsterdam）、加拿大——無亞太區；R2 支援 APAC location hint（實際儲存落亞太），Cloudflare 另有台北 PoP。推測：台灣→B2 美西 RTT 約 130–180ms、→Wasabi 東京/R2 APAC 約 35–60ms（未找到正式評測，屬網路地理常識推估）。但本案存取型態是「夜間批次上傳＋偶發覆核回聽」，非即時串流，延遲差異對體驗影響極小——選型應以價格與 lifecycle 功能為主，延遲僅作 tie-breaker。
  來源：<https://www.backblaze.com/docs/cloud-storage-data-regions>、<https://wasabi.com/company/storage-regions>、<https://developers.cloudflare.com/r2/pricing/>

- **雙軌保存政策穩態成本與 90/180 日矛盾的整合解**（信心：high）
  穩態儲存 ≈ 月增量 ×（0.9×3 個月＋0.1×6 個月）= ×3.3 → 10/25/50 對 = 8.2/20.6/41.2GB（悲觀每週3場 ×1.5 = 12.4/30.9/61.7GB）。矛盾解法（工程上已驗證可行）：R2 支援 object lifecycle rules（如 Expiration: {Days: 90}，可按 prefix 套用）——一般錄音放 recordings/ 前綴掛 90 日自動刪除；規則層或 LLM 命中的場次由篩查 pipeline 立即複製到 holds/ 前綴（無自動刪除規則），依性剝削條例 §8 保留至少 180 日、且個案未結案前不得刪（可加 R2 bucket lock 防誤刪）。即：90 日是「預設軌」、180 日是「法定覆蓋軌」，兩者以 prefix 隔離而非同一排程互斥，矛盾消失。成本影響：holds/ 佔比 10% 時整體僅 +10%，仍是每月個位數美元。附帶：Wasabi 的 90 天最短計費期恰好等於一般軌保存期，無懲罰，但 1TB 低消仍使它不划算。
  來源：<https://developers.cloudflare.com/r2/buckets/object-lifecycles/>、<https://law.moj.gov.tw/LawClass/LawAll.aspx?pcode=D0050023>、<https://docs.wasabi.com/docs/how-does-wasabis-minimum-storage-duration-policy-work>

- **性剝削條例 §8 的 180 日義務範圍（法定覆蓋軌的依據）**（信心：high）
  查證到的事實：兒童及少年性剝削防制條例第 8 條課予網際網路平臺提供者於知悉第四章犯罪嫌疑時「先行限制瀏覽或移除」，且犯罪網頁資料、行為人個資及網路使用紀錄應「保留一百八十日」以供司法及警察機關調查；違反保留或提供義務有罰則。推論：bandu 自建平台若被認定為網際網路平臺提供者，命中疑似案件的錄音、逐字稿、帳號個資與連線紀錄均落入 180 日保留範圍——儲存排程設計必須以此為不可退讓的下限，此即前項 holds/ 軌的法源。
  來源：<https://law.moj.gov.tw/LawClass/LawAll.aspx?pcode=D0050023>

- **運算—VPS：台灣 NT$990 vs 東京 $5–24，語音負載兩者皆綽綽有餘**（信心：medium）
  遠振台灣 VPS NT$990/月 = 2 核/4GB/50GB SSD/2TB 流量，機房直連 HiNet（本島 RTT 最低）。Vultr 東京 regular performance 自 $2.50–5/月（1GB $5 ≈ NT$161），東京區頻寬超額 $0.05/GB；Linode/Akamai 東京/大阪 shared 4GB $24/月（NT$775）。負載試算：50 對同時 = 50 房 × 2 人 × 32kbps ≈ 6.4Mbps 出流量、SFU 純轉發不轉碼 CPU 極輕；月流量 433 場 × ~58MB ≈ 50GB，遠低於任何方案額度。結論：$5–10 東京 VPS 技術上足夠（台日 RTT ~35–50ms 對語音可接受）；NT$990 台灣機的價值是 HiNet 低延遲與在地資料落地（偏鄉學童多走中華電信），若重視「錄音檔不出境」的說服力可選台灣機——差價每月約 NT$600–800。
  來源：<https://host.com.tw/%E5%8F%B0%E7%81%A3VPS%E4%B8%BB%E6%A9%9F>、<https://www.vultr.com/pricing/>、<https://www.akamai.com/cloud/pricing>、<https://www.nss.com.tw/vps-hosting-taiwan-2025>

- **運算—夜間批次 STT：自有 GPU 電費近零，租用月費數百元台幣**（信心：high）
  faster-whisper large-v3 於 RTX 4090：1 小時錄音約 9 分鐘內轉完（≈7–8x），批次＋flash-attention 可達 70–100x（短檔）——保守以 8–13x 計，50 對的 433 音檔小時/月 ≈ 33–54 GPU 小時。三方案：(a) 自有 GPU（桌機）：~0.4kW × 45h ≈ 18 度電 × 台電住宅 NT$2.26–5.27/度（多數落 3–5 元級距）≈ NT$50–90/月，近乎免費；(b) RunPod 4090：community $0.34/hr → 10/25/50 對 ≈ $3/$7.4/$14.7（NT$97/240/475），secure $0.69 翻倍，但 community 可被搶佔且是把兒少語音上第三方 GPU，隱私上不如自架；(c) Groq whisper-large-v3-turbo API $0.04/音檔小時 → $3.5/$8.7/$17.3（OpenAI Whisper $0.36/hr 貴 9 倍不考慮）。推測性提醒：兒童語音送第三方 API 的適法性（個資目的外利用、當事人告知）尚未在法遵維度裁決，自有 GPU 夜批是隱私最穩的預設。
  來源：<https://www.runpod.io/pricing>、<https://www.synpixcloud.com/blog/rtx-4090-cloud-rental-worth-it>、<https://github.com/SYSTRAN/faster-whisper>、<https://runaihome.com/blog/whisper-large-v3-self-hosted-transcription-server-2026/>、<https://console.groq.com/docs/model/whisper-large-v3-turbo>、<https://tokenmix.ai/blog/whisper-api-pricing>、<https://www.housefeel.com.tw/article/%E5%8F%B0%E9%9B%BB%E9%9B%BB%E8%B2%BB-%E5%A4%8F%E5%AD%A3%E9%9B%BB%E8%B2%BB-%E9%9B%BB%E8%B2%BB%E8%A8%88%E7%AE%97/>

- **運算—LLM 篩查單場成本：三候選模型實算，每場 NT$0.1–0.8**（信心：medium）
  Token 估算：華語對話約 150–220 字/分，60 分雙人 ≈ 9,000–13,000 字，以 ~1.5 token/字計＋篩查 rubric 提示 ≈ 15–22k input tokens、輸出 ~1k。單場成本：(a) Gemini 2.5 Flash-Lite（$0.10/$0.40 每百萬）≈ $0.0024（NT$0.08）；(b) GPT-5 mini（現行第三方報價 $0.125/$1.00，發布價 $0.25/$2.00）≈ $0.003–0.007（NT$0.1–0.23）；(c) Claude Haiku 4.5（$1/$5，官方快取表 2026-06）≈ $0.025，走 Batch API 五折 ≈ $0.0125（NT$0.40）——夜批場景天然適合 batch。月費（50 對 433 場）：Flash-Lite ~$1、GPT-5 mini ~$1.5–3、Haiku 4.5 batch ~$5.4。結論：LLM 篩查成本無論選誰都是零頭，選型應以「抓脫離平台誘導訊號的召回率」評測決定而非價格；建議雙模型交叉（如 Flash-Lite 全量＋Haiku 對規則層命中複審）也才 ~$6/月。
  來源：<https://devtk.ai/en/models/gemini-2-5-flash-lite/>、<https://pricepertoken.com/pricing-page/model/openai-gpt-5-mini>、<https://developers.openai.com/api/docs/pricing>、<https://platform.claude.com/docs/en/pricing.md>

- **人工覆核量能模型：斷點落在悲觀參數的 29–49 對**（信心：high）
  模型（假設全部明列）：每週場次 = 對數 × 每週場次/對；覆核佇列 = 規則層命中 5–10% ＋ LLM 高風險 1–3%（部分重疊，合計取 6–13%）；單件含回聽 10–20 分；另設 5–10% 未命中場次隨機抽查 × 10–15 分（維度三「人為終審」的最低品保）。每週覆核時數：10 對 = 0.5–1.4h；25 對 = 1–3.5h；50 對基準 = 1.8–6.8h、悲觀（每週 3 場/對）= 10.3h。對照單人可投入時數（下班志工型估每週 10h、可持續投入覆核約 6h）：樂觀參數（6% 命中、10 分/件、5% 抽查、每週 2 場）時數 ≈ 0.037×對數，斷點在 270 對外，覆核不構成瓶頸；悲觀參數（13%、20 分、10% 抽查、每週 3 場）時數 ≈ 0.205×對數，6h 預算在 ~29 對爆、10h 在 ~49 對爆。結論：50 對是否可行完全取決於誤報率實測值——這是可用前三個月 10 對試營運直接校準的參數，不必紙上賭。
  來源：<https://law.moj.gov.tw/LawClass/LawAll.aspx?pcode=D0050023>

- **24 小時通報的單人排班：件數可行、韌性不可行**（信心：medium）
  件數面：即使 50 對悲觀情境，每週高風險件僅 1–4.5 件（LLM 1–3% × 100–150 場），單件通報作業本身不構成工時問題。韌性面（推測，屬營運判斷）：24 小時時限意味覆核者必須全年無休可觸達——生病、出國、上班日白天開會、手機沒訊號任一情況即形成通報斷鏈，而通報遲誤在兒少保護體系是有裁罰與究責風險的義務（兒少權法第 53 條體系的 24 小時通報義務）。單人排班在結構上不可持續，與規模無關；最低救濟：與合作機構社工簽定「第二責任人」分工（機構本就有法定通報人員身分與經驗），高風險警示同時推送兩人，任一人 24 小時內處理即可。此人同時天然成為 50 對級距所需的第二覆核者。
  來源：<https://law.moj.gov.tw/LawClass/LawAll.aspx?pcode=D0050023>、<https://www.laf.org.tw/>

- **法遵一次性成本：委任行情 NT$25,000–110,000，公益資源可壓到 NT$10,000–30,000**（信心：medium）
  查證到的行情（多家事務所/法律平台 2026 公開報價）：契約/文件審閱固定價每份約 NT$10,000–50,000（依複雜度），鐘點費 NT$3,000–15,000/小時（常見 5,000–10,000）；撰擬契約 NT$10,000–20,000 起。本案需求推估：家長知情同意書（含錄音/個資告知）＋志工服務同意書＋機構合作 MOU 共 2–3 份審閱 ≈ NT$20,000–80,000；代擬主管機關（衛福部社家署/縣市社會局）函詢：無專門公開報價，比照律師函/簡式法律意見行情推估 NT$5,000–30,000/件。降費途徑（查證到）：各縣市政府免費法律諮詢（台北市由法扶、台北律師公會、基隆律師公會輪派義務律師，網路預約、每次 20 分鐘）可免費取得方向但不代擬文件；法律扶助基金會主要扶助「無資力個人」且經資力審查，籌備階段組織不符主要路徑；婦幼兒少領域 NGO（勵馨、兒少權心會、婦女新知等）有領域內法律諮詢資源，且與本案領域高度相關，是比通用管道更對口的免費入口。務實組合：免費諮詢定方向＋自擬文件後付費請律師審閱（單次 NT$3,000–10,000/份）→ 總價可壓到 NT$10,000–30,000。
  來源：<https://www.linelawyer.com.tw/consult-knowledge/lawyerfee/>、<https://zhelu.tw/post/attorney-fees>、<https://legalaffairs.gov.taipei/cp.aspx?n=400B74E56DB26180>、<https://www.laf.org.tw/service-assistance-des>、<https://vocus.cc/article/6886245efd89780001070ab3>、<https://laws010.com/blog/recommend-attorney/how-to-find-an-attorney/how-to-find-an-attorney-02/>

### 未解問題

- 每對每週場次數（2 vs 3 場）與單場時長尚未定案——此參數直接 1.5 倍放大所有儲存、STT 與人力數字，是斷點計算的最大不確定源

- 規則層命中率 5–10% 與 LLM 高風險率 1–3% 均為紙上假設，無同類平台公開基準可查——需以前三個月試營運實測回校，若實際誤報率更高則 25 對即可能滿載

- 桌機 GPU 型號與 VRAM 是否足以夜批 faster-whisper large-v3（需 ≥8GB VRAM）？若不足，退 medium 模型（華語準確度損失需實測）或改租 RunPod

- 兒童語音/逐字稿送第三方 API（Groq/Gemini/Claude）的適法性未經法遵維度裁決——若結論是不可出境或不可目的外利用，STT 與 LLM 都須自架，成本結構不變（仍近零）但 LLM 本地模型的篩查召回率需另行驗證

- 覆核者「單件 10–20 分」是否包含通報文書作業與機構溝通？若通報單件實際耗 1–2 小時，高風險件雖少仍可能在集中爆發週壓垮排程

- 未命中場次的隨機抽查比例（本試算假設 5–10%）是政策決定而非技術參數——抽查率每加 5 個百分點，50 對級距每週多 0.8–1.2 小時

- 合作機構是否願意承擔第二覆核者/24 小時通報備援角色並寫入 MOU？此為 50 對級距的先決條件，若機構不承擔則需自行招募受訓志工覆核者（另有督導與保密協議成本）

- 律師代擬主管機關函詢的報價區間（NT$5,000–30,000）是以律師函/法律意見行情推估，非專門查得的函詢代擬報價——實際委任前應取 2–3 家報價
