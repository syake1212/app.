<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>総合防災モニタ + AI</title>
    
    <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2000/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>🛡️</text></svg>">
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" integrity="sha256-p4NxAoJBhIIN+hmNHrzRCf9tD/miZyoHS5obTRR9BMY=" crossorigin=""/>
    
    <style>
        body { 
            margin: 0; padding: 0; 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
            background-color: #a0dceb;
            overflow: hidden;
        }
        
        #map { height: 100vh; width: 100%; z-index: 1; }

        /* --- 左上：地震情報パネル --- */
        #info-panel {
            position: absolute; top: 20px; left: 20px; z-index: 1000;
            background: rgba(255, 255, 255, 0.9);
            backdrop-filter: blur(10px); -webkit-backdrop-filter: blur(10px);
            padding: 20px; border-radius: 16px;
            box-shadow: 0 8px 32px rgba(0,0,0,0.1);
            max-width: 320px; width: 90%;
            border: 1px solid rgba(255, 255, 255, 0.4);
            transition: transform 0.3s cubic-bezier(0.25, 0.8, 0.25, 1), opacity 0.3s;
            transform: translateX(0);
            opacity: 1;
        }
        /* パネル非表示時のクラス */
        #info-panel.panel-hidden {
            transform: translateX(-120%);
            opacity: 0;
            pointer-events: none;
        }

        /* パネルヘッダー（タイトルと閉じるボタン） */
        .panel-header {
            display: flex; justify-content: space-between; align-items: center;
            margin-bottom: 15px;
        }
        h1 { margin: 0; font-size: 20px; font-weight: 800; color: #2c3e50; display: flex; align-items: center; gap: 8px; }
        
        /* 閉じるボタン（最小化） */
        .btn-minimize {
            background: none; border: none; color: #999; font-size: 24px;
            cursor: pointer; line-height: 1; padding: 0 5px;
        }
        .btn-minimize:hover { color: #555; }

        /* 復帰ボタン（情報表示） */
        #btn-restore-panel {
            position: absolute; top: 20px; left: 20px; z-index: 999;
            background: white; border: none; border-radius: 50%;
            width: 44px; height: 44px;
            box-shadow: 0 4px 10px rgba(0,0,0,0.2);
            cursor: pointer; display: flex; align-items: center; justify-content: center;
            font-size: 20px;
            transition: transform 0.3s cubic-bezier(0.175, 0.885, 0.32, 1.275);
            transform: scale(0); /* 初期状態は非表示 */
        }
        #btn-restore-panel.visible {
            transform: scale(1);
        }

        .status-box { margin-bottom: 15px; font-size: 14px; line-height: 1.6; color: #444; }
        .status-box strong { font-size: 16px; color: #e74c3c; }

        /* --- 天気セクション --- */
        .weather-section {
            margin-top: 15px; padding-top: 10px; border-top: 1px solid rgba(0,0,0,0.1);
        }
        .weather-header {
            display: flex; justify-content: space-between; align-items: center; margin-bottom: 5px;
        }
        .weather-title { font-weight: bold; font-size: 14px; color: #2c3e50; display: flex; align-items: center; gap: 5px; }
        .btn-get-location {
            background: #3498db; color: white; border: none; border-radius: 12px;
            padding: 4px 10px; font-size: 11px; cursor: pointer; font-weight: bold;
        }
        .btn-get-location:hover { background: #2980b9; }
        
        .weather-content {
            display: flex; align-items: center; justify-content: center;
            background: rgba(255,255,255,0.5); border-radius: 8px;
            padding: 10px; min-height: 50px;
        }
        .weather-icon { font-size: 32px; margin-right: 10px; }
        .weather-details { font-size: 13px; color: #444; line-height: 1.4; text-align: left; }
        .weather-temp { font-size: 18px; font-weight: bold; color: #333; }

        /* --- 地震凡例 --- */
        .legend {
            margin-top: 10px; padding-top: 10px; border-top: 1px solid rgba(0,0,0,0.1);
            font-size: 12px; display: flex; flex-wrap: wrap; gap: 8px; color: #555;
        }
        .legend-item { display: flex; align-items: center; }
        .color-box { width: 14px; height: 14px; margin-right: 6px; border-radius: 4px; border: 1px solid rgba(0,0,0,0.1); }

        /* --- 左下：レイヤー＆雨雲コントロール --- */
        #layer-controls {
            position: absolute; bottom: 30px; left: 20px; z-index: 900;
            display: flex; flex-direction: column; gap: 10px; width: 300px; max-width: 80vw;
        }
        
        .control-group {
            background: rgba(255, 255, 255, 0.95);
            backdrop-filter: blur(5px);
            padding: 10px; border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.1);
        }

        .layer-btn {
            background: white; border: 1px solid #ddd; padding: 8px 12px;
            border-radius: 20px; font-weight: bold; color: #555;
            cursor: pointer; display: flex; align-items: center; gap: 6px;
            font-size: 13px; width: 100%; justify-content: flex-start;
            margin-bottom: 5px;
        }
        .layer-btn:hover { background: #f8f9fa; }
        .layer-btn.active { background: #3498db; color: white; border-color: #3498db; }
        .layer-btn.active-rain { background: #2980b9; color: white; border-color: #2980b9; }
        .layer-btn.active-cloud { background: #7f8c8d; color: white; border-color: #7f8c8d; }

        /* 雨雲プレイヤーのUI */
        #rain-player { display: none; margin-top: 5px; }
        .player-row { display: flex; align-items: center; gap: 10px; margin-top: 5px; }
        .player-btn {
            background: #2980b9; color: white; border: none; border-radius: 50%;
            width: 30px; height: 30px; cursor: pointer; display: flex; align-items: center; justify-content: center;
            font-size: 14px;
        }
        /* 「現在」ボタンのスタイル */
        .btn-now {
            background: #e67e22; font-size: 11px; width: auto; padding: 0 10px; border-radius: 15px;
        }
        
        .player-slider { flex: 1; cursor: pointer; }
        .time-label { font-size: 12px; font-weight: bold; color: #2980b9; min-width: 50px; text-align: right; }
        
        .forecast-badge { 
            font-size: 10px; padding: 2px 6px; border-radius: 4px; display: inline-block; margin-right: 5px;
        }
        .badge-live { background: #27ae60; color: white; } /* 実況（緑） */
        .badge-forecast { background: #e67e22; color: white; } /* 予報（オレンジ） */

        /* ガイドメッセージ（雨雲表示中など） */
        #guide-message {
            position: absolute; top: 10px; left: 50%; transform: translateX(-50%);
            z-index: 1001; background: rgba(0,0,0,0.7); color: white;
            padding: 8px 16px; border-radius: 20px; font-size: 12px;
            display: none; pointer-events: none; white-space: nowrap;
        }

        /* --- 右下：AIチャットボタン --- */
        #ai-trigger {
            position: absolute; bottom: 30px; right: 20px; z-index: 1000;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white; width: 60px; height: 60px; border-radius: 30px;
            display: flex; align-items: center; justify-content: center;
            font-size: 30px; border: none; cursor: pointer;
            box-shadow: 0 4px 15px rgba(0,0,0,0.3);
            transition: transform 0.2s;
        }
        #ai-trigger:hover { transform: scale(1.05); }

        /* --- AIチャットウィンドウ --- */
        #ai-window {
            position: absolute; bottom: 100px; right: 20px; z-index: 1001;
            width: 350px; height: 500px; max-width: 90vw; max-height: 60vh;
            background: white; border-radius: 16px;
            box-shadow: 0 5px 25px rgba(0,0,0,0.2);
            display: none; flex-direction: column;
            overflow: hidden;
            border: 1px solid #ddd;
        }
        .ai-header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white; padding: 15px; font-weight: bold;
            display: flex; justify-content: space-between; align-items: center;
        }
        .close-btn { background: none; border: none; color: white; font-size: 20px; cursor: pointer; }
        .chat-area {
            flex: 1; padding: 15px; overflow-y: auto; background: #f9f9f9;
            display: flex; flex-direction: column; gap: 10px;
        }
        .message { padding: 10px 14px; border-radius: 12px; max-width: 80%; font-size: 14px; line-height: 1.5; }
        .msg-ai { background: white; border: 1px solid #eee; align-self: flex-start; color: #333; }
        .msg-user { background: #667eea; color: white; align-self: flex-end; }
        
        .input-area {
            padding: 10px; border-top: 1px solid #eee; background: white;
            display: flex; gap: 8px;
        }
        #user-input {
            flex: 1; padding: 10px; border: 1px solid #ddd; border-radius: 20px;
            outline: none; font-size: 14px;
        }
        #send-btn {
            background: #764ba2; color: white; border: none; padding: 0 15px;
            border-radius: 20px; cursor: pointer; font-weight: bold;
        }
        #send-btn:disabled { background: #ccc; cursor: not-allowed; }

        /* --- AI分析レポートモーダル --- */
        #report-modal {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            z-index: 2001; width: 90%; max-width: 500px;
            background: white; border-radius: 16px;
            box-shadow: 0 10px 40px rgba(0,0,0,0.3);
            display: none; flex-direction: column;
            border: 1px solid #ddd;
            animation: fadeIn 0.3s ease;
        }
        .report-header {
            padding: 15px 20px; background: #f8f9fa; border-bottom: 1px solid #eee;
            border-radius: 16px 16px 0 0; display: flex; justify-content: space-between; align-items: center;
        }
        .report-title { font-size: 18px; font-weight: bold; color: #2c3e50; display: flex; align-items: center; gap: 8px; }
        .report-content { padding: 20px; line-height: 1.6; color: #444; max-height: 60vh; overflow-y: auto; }
        .report-content h3 { color: #e74c3c; margin-top: 0; }
        .report-content ul { padding-left: 20px; margin: 10px 0; }
        .report-content li { margin-bottom: 8px; }
        
        @keyframes fadeIn { from { opacity: 0; transform: translate(-50%, -45%); } to { opacity: 1; transform: translate(-50%, -50%); } }

        /* --- その他 --- */
        #loading {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            z-index: 2000; color: #333; background: rgba(255,255,255,0.95);
            padding: 20px 40px; border-radius: 50px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.15); font-weight: bold;
            display: none; pointer-events: none;
        }

        .epicenter-icon { background: transparent; border: none; }
        .epicenter-svg {
            width: 100%; height: 100%; overflow: visible;
            animation: pulse 2s infinite;
            filter: drop-shadow(0 3px 5px rgba(0,0,0,0.3));
        }
        @keyframes pulse {
            0% { transform: scale(0.9); opacity: 1; }
            50% { transform: scale(1.1); opacity: 0.9; }
            100% { transform: scale(0.9); opacity: 1; }
        }
        
        /* AIボタンのスタイル */
        .btn-ai-analyze {
            width: 100%; margin-top: 10px; padding: 10px;
            background: linear-gradient(90deg, #667eea, #764ba2);
            color: white; border: none; border-radius: 8px;
            font-weight: bold; cursor: pointer; display: flex; align-items: center; justify-content: center; gap: 6px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.2); transition: transform 0.1s;
        }
        .btn-ai-analyze:hover { transform: translateY(-1px); }
        .btn-ai-analyze:active { transform: translateY(0); }
    </style>
</head>
<body>

    <!-- ガイドメッセージ -->
    <div id="guide-message"></div>

    <!-- パネル復帰ボタン (初期は非表示) -->
    <button id="btn-restore-panel" onclick="toggleInfoPanel()">ℹ️</button>

    <!-- 左上：地震情報 -->
    <div id="info-panel">
        <div class="panel-header">
            <h1>🛡️ 総合防災モニタ</h1>
            <button class="btn-minimize" onclick="toggleInfoPanel()" title="パネルを隠す">×</button>
        </div>

        <div id="quake-data" class="status-box">最新データを取得中...</div>
        
        <!-- 現在地の天気セクション -->
        <div class="weather-section">
            <div class="weather-header">
                <div class="weather-title">📍 現在地の天気</div>
                <button class="btn-get-location" onclick="getUserLocation()">取得</button>
            </div>
            <div id="weather-display" class="weather-content">
                <span style="font-size:12px; color:#777;">「取得」ボタンを押して表示</span>
            </div>
        </div>

        <div class="legend">
            <div class="legend-item" style="width:100%; margin-bottom:5px;"><strong>震度階級</strong></div>
            <div class="legend-item"><div class="color-box" style="background:transparent; border: 1px dashed #999;"></div>なし</div>
            <div class="legend-item"><div class="color-box" style="background:#00aaff;"></div>1-2</div>
            <div class="legend-item"><div class="color-box" style="background:#00c400;"></div>3</div>
            <div class="legend-item"><div class="color-box" style="background:#f6f600;"></div>4</div>
            <div class="legend-item"><div class="color-box" style="background:#ffae00;"></div>5</div>
            <div class="legend-item"><div class="color-box" style="background:#ff0000;"></div>6+</div>
        </div>
        
        <div style="margin-top:15px; display:flex; gap:10px;">
            <button onclick="fetchQuakeData()" style="flex:1; padding:8px; border-radius:6px; border:none; background:#eee; cursor:pointer;">🔄 更新</button>
            <button onclick="loadTestData()" style="flex:1; padding:8px; border-radius:6px; border:none; background:#eee; cursor:pointer;">🧪 テスト</button>
        </div>
        
        <!-- 新機能：AI分析レポートボタン -->
        <button class="btn-ai-analyze" onclick="generateAiReport()">
            <span>✨</span> AI分析レポートを作成
        </button>
    </div>

    <!-- 左下：レイヤーコントロール -->
    <div id="layer-controls">
        <div class="control-group">
            <button class="layer-btn active" id="btn-quake" onclick="toggleLayer('quake')">
                <span>🌋</span> 地震情報
            </button>
        </div>

        <div class="control-group">
            <button class="layer-btn" id="btn-rain" onclick="toggleLayer('rain')">
                <span>☔</span> 雨雲レーダー & 予報
            </button>
            
            <div id="rain-player">
                <div class="player-row">
                    <button class="player-btn" id="play-pause-btn" onclick="toggleRainAnimation()">▶</button>
                    <!-- 「現在」に戻るボタン -->
                    <button class="player-btn btn-now" onclick="resetRainToNow()">現在</button>
                    <input type="range" id="rain-slider" class="player-slider" min="0" max="100" value="0" step="1">
                </div>
                <div class="player-row" style="justify-content: flex-end;">
                    <span id="rain-forecast-badge" class="forecast-badge">実況</span>
                    <span id="rain-time-label" class="time-label">--:--</span>
                </div>
            </div>
        </div>

        <!-- 気象衛星ボタン -->
        <div class="control-group">
            <button class="layer-btn" id="btn-cloud" onclick="toggleLayer('cloud')">
                <span>☁️</span> 気象衛星 (ひまわり)
            </button>
        </div>
    </div>

    <!-- 右下：AIチャット -->
    <button id="ai-trigger" onclick="toggleAiWindow()">🤖</button>
    
    <!-- AIチャットウィンドウ -->
    <div id="ai-window">
        <div class="ai-header">
            <span>AI防災アドバイザー</span>
            <button class="close-btn" onclick="toggleAiWindow()">×</button>
        </div>
        <div class="chat-area" id="chat-area">
            <div class="message msg-ai">
                こんにちは！AI防災アドバイザーです。<br>
                現在の地震情報や、雨雲の状況について何でも聞いてください。避難の知識などもお答えします。
            </div>
        </div>
        <div class="input-area">
            <input type="text" id="user-input" placeholder="例: 今の地震はどこ？" onkeypress="handleKeyPress(event)">
            <button id="send-btn" onclick="sendMessage()">送信</button>
        </div>
    </div>
    
    <!-- AI分析レポートモーダル -->
    <div id="report-modal">
        <div class="report-header">
            <div class="report-title">📊 AI防災分析</div>
            <button class="close-btn" onclick="closeReportModal()" style="color:#555; font-size:24px;">×</button>
        </div>
        <div class="report-content" id="report-content">
            分析中...
        </div>
    </div>
    
    <!-- モーダル背景（オーバーレイ） -->
    <div id="modal-overlay" onclick="closeReportModal()" style="display:none; position:fixed; top:0; left:0; width:100%; height:100%; background:rgba(0,0,0,0.5); z-index:2000;"></div>

    <div id="loading">システム起動中...</div>
    <div id="map"></div>

    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js" integrity="sha256-20nQCchB9co0qIjJZRGuk2/Z9VM+kNiyxNV1lvTlZBo=" crossorigin=""></script>

    <script>
        // =========================================================
        // 1. 基本設定・変数
        // =========================================================
        const map = L.map('map', { zoomControl: false }).setView([38, 137], 5);
        L.control.zoom({ position: 'bottomright' }).addTo(map);

        L.tileLayer('https://cyberjapandata.gsi.go.jp/xyz/pale/{z}/{x}/{y}.png', {
            attribution: '国土地理院 | 気象庁 | Open-Meteo', maxZoom: 18, opacity: 0.9
        }).addTo(map);

        // API Key
        const apiKey = ""; 

        // レイヤー・データ管理
        let geojsonLayer = null; 
        let hypocenterLayer = null; 
        let rainLayer = null;
        let testRainLayer = null; // テスト用雨雲レイヤー
        let cloudLayer = null; // ひまわりレイヤー
        let currentLocationMarker = null; // 現在地マーカー
        let prefShakeData = {}; 
        let currentQuakeInfo = null; // AI用に最新地震データを保持

        // 雨雲アニメーション管理
        let rainTimeData = []; // APIから取得した時刻リスト
        let rainIndex = 0;
        let rainInterval = null;
        let isRainPlaying = false;
        let rainAutoUpdateTimer = null; // 自動更新タイマー
        
        let showQuake = true;
        let showRain = false;
        let showCloud = false;

        // パネル表示制御
        let isPanelVisible = true;

        function toggleInfoPanel() {
            const panel = document.getElementById('info-panel');
            const restoreBtn = document.getElementById('btn-restore-panel');
            
            if (isPanelVisible) {
                // 隠す
                panel.classList.add('panel-hidden');
                restoreBtn.classList.add('visible');
            } else {
                // 表示する
                panel.classList.remove('panel-hidden');
                restoreBtn.classList.remove('visible');
            }
            isPanelVisible = !isPanelVisible;
        }

        // =========================================================
        // 2. 現在地の天気取得 (Open-Meteo API)
        // =========================================================
        function getUserLocation() {
            const display = document.getElementById('weather-display');
            display.innerHTML = '<span style="font-size:12px; color:#555;">位置情報を取得中...</span>';

            if (!navigator.geolocation) {
                display.innerHTML = '<span style="font-size:12px; color:red;">お使いのブラウザは位置情報をサポートしていません</span>';
                return;
            }

            navigator.geolocation.getCurrentPosition(
                async (position) => {
                    const lat = position.coords.latitude;
                    const lon = position.coords.longitude;
                    
                    // 地図を現在地に移動し、マーカーを表示
                    map.setView([lat, lon], 12);
                    
                    if (currentLocationMarker) map.removeLayer(currentLocationMarker);
                    currentLocationMarker = L.marker([lat, lon]).addTo(map)
                        .bindPopup("現在地").openPopup();

                    // 天気API呼び出し
                    await fetchLocalWeather(lat, lon);
                },
                (error) => {
                    console.error(error);
                    let msg = "位置情報の取得に失敗しました";
                    if (error.code === 1) msg = "位置情報の利用が許可されていません";
                    display.innerHTML = `<span style="font-size:12px; color:red;">${msg}</span>`;
                }
            );
        }

        async function fetchLocalWeather(lat, lon) {
            const display = document.getElementById('weather-display');
            try {
                // Open-Meteo API (無料・認証不要)
                const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,relative_humidity_2m,precipitation,weather_code&timezone=auto`;
                const res = await fetch(url);
                const data = await res.json();
                
                if (!data.current) throw new Error("No data");

                const current = data.current;
                const weather = getWeatherInfo(current.weather_code);
                
                display.innerHTML = `
                    <div class="weather-icon">${weather.icon}</div>
                    <div class="weather-details">
                        <div class="weather-temp">${current.temperature_2m}°C</div>
                        <div>${weather.text} | 湿度 ${current.relative_humidity_2m}%</div>
                        ${current.precipitation > 0 ? `<div style="color:#2980b9;">☔ ${current.precipitation}mm</div>` : ''}
                    </div>
                `;

            } catch (e) {
                console.error(e);
                display.innerHTML = '<span style="font-size:12px; color:red;">天気情報の取得に失敗しました</span>';
            }
        }

        // WMO天気コード変換
        function getWeatherInfo(code) {
            // 0: 快晴, 1-3: 晴れ〜曇り, 45-48: 霧, 51-55: 霧雨, 61-65: 雨, 71-77: 雪, 80-82: にわか雨, 95-99: 雷雨
            if (code === 0) return { icon: "☀️", text: "快晴" };
            if (code <= 3) return { icon: "⛅", text: "晴れ/曇り" };
            if (code <= 48) return { icon: "🌫️", text: "霧" };
            if (code <= 57) return { icon: "🌦️", text: "霧雨" };
            if (code <= 67) return { icon: "☔", text: "雨" };
            if (code <= 77) return { icon: "☃️", text: "雪" };
            if (code <= 82) return { icon: "☔", text: "にわか雨" };
            if (code <= 99) return { icon: "⚡", text: "雷雨" };
            return { icon: "❓", text: "不明" };
        }

        // =========================================================
        // 3. 雨雲レーダー（アニメーション・予報対応・自動更新）
        // =========================================================
        
        // UTC時刻文字列(YYYYMMDDHHmmss)をDateオブジェクト(JST)に変換する
        function parseJmaTime(str) {
            const y = parseInt(str.substring(0, 4), 10);
            const m = parseInt(str.substring(4, 6), 10) - 1;
            const d = parseInt(str.substring(6, 8), 10);
            const h = parseInt(str.substring(8, 10), 10);
            const min = parseInt(str.substring(10, 12), 10);
            const sec = parseInt(str.substring(12, 14), 10);
            // UTCとしてDateを作成し、それを利用
            return new Date(Date.UTC(y, m, d, h, min, sec));
        }

        async function fetchRainTimes(forceLatest = false) {
            try {
                const ts = new Date().getTime(); 
                const res = await fetch(`https://www.jma.go.jp/bosai/jmatile/data/nowc/targetTimes_N1.json?_=${ts}`);
                const data = await res.json();
                
                // hrpnsを含む要素のみ抽出
                rainTimeData = data.filter(d => d.elements.includes('hrpns')).sort((a, b) => {
                    return Number(a.validtime) - Number(b.validtime);
                });
                
                if (rainTimeData.length > 0) {
                    const slider = document.getElementById('rain-slider');
                    slider.max = rainTimeData.length - 1;
                    
                    // 現在時刻(JST)とAPIのvalidtime(UTC)を比較して、
                    // 「現在時刻を過ぎていない最新の観測データ」を探す。
                    const now = new Date();
                    let bestIndex = 0;
                    
                    for (let i = 0; i < rainTimeData.length; i++) {
                        const d = rainTimeData[i];
                        const validDate = parseJmaTime(d.validtime);
                        // 未来(予報)になったらその手前が最新の実況
                        if (validDate > now) {
                            // ひとつ前が最新の実況だが、なければ0
                            bestIndex = (i > 0) ? i - 1 : 0;
                            break;
                        }
                        // 最後まで行ったら最後が最新
                        bestIndex = i;
                    }
                    
                    // 強制リセット時、またはスライダー操作中でない時にインデックスを合わせる
                    if (forceLatest || (!isRainPlaying && document.activeElement !== slider)) {
                        rainIndex = bestIndex;
                        slider.value = rainIndex;
                        updateRainDisplay();
                    }
                }
            } catch (e) {
                console.error("雨雲データ取得エラー:", e);
                document.getElementById('rain-time-label').innerText = "error";
            }
        }

        function updateRainDisplay() {
            // テスト表示中はリアル雨雲を更新しない
            if (testRainLayer) return;

            if (!rainTimeData[rainIndex]) return;
            const data = rainTimeData[rainIndex];
            const validTime = data.validtime; // UTC文字列
            
            // 地図タイルURL (APIはUTCをそのまま使う)
            const url = `https://www.jma.go.jp/bosai/jmatile/data/nowc/${validTime}/none/{z}/{x}/{y}.png`;
            if (rainLayer) map.removeLayer(rainLayer);
            rainLayer = L.tileLayer(url, {
                opacity: 0.6, maxZoom: 18, attribution: '気象庁(雨雲)'
            }).addTo(map);

            // UI更新: UTC時刻をJSTに変換して表示する
            const dateObj = parseJmaTime(validTime);
            // 日本時間での時分を取得
            const hh = dateObj.getHours().toString().padStart(2, '0');
            const mm = dateObj.getMinutes().toString().padStart(2, '0');
            document.getElementById('rain-time-label').innerText = `${hh}:${mm}`;
            document.getElementById('rain-slider').value = rainIndex;

            // バッジの切り替え
            const badge = document.getElementById('rain-forecast-badge');
            // validtime > basetime なら予報
            if (Number(data.validtime) > Number(data.basetime)) {
                badge.className = 'forecast-badge badge-forecast';
                badge.innerText = '予報';
                badge.style.display = 'inline-block';
            } else {
                badge.className = 'forecast-badge badge-live';
                badge.innerText = '実況';
                badge.style.display = 'inline-block';
            }
        }

        function resetRainToNow() {
            // テストモード解除
            if (testRainLayer) {
                map.removeLayer(testRainLayer);
                testRainLayer = null;
            }
            if (isRainPlaying) toggleRainAnimation();
            // データ再取得して最新に合わせる
            fetchRainTimes(true);
        }

        function toggleRainAnimation() {
            if (testRainLayer) {
                map.removeLayer(testRainLayer);
                testRainLayer = null;
            }

            const btn = document.getElementById('play-pause-btn');
            if (isRainPlaying) {
                clearInterval(rainInterval);
                isRainPlaying = false;
                btn.innerText = "▶";
            } else {
                isRainPlaying = true;
                btn.innerText = "⏸";
                rainInterval = setInterval(() => {
                    rainIndex++;
                    if (rainIndex >= rainTimeData.length) {
                        rainIndex = 0; // ループ
                    }
                    updateRainDisplay();
                }, 500);
            }
        }
        
        function startRainAutoUpdate() {
            if (rainAutoUpdateTimer) clearInterval(rainAutoUpdateTimer);
            rainAutoUpdateTimer = setInterval(() => {
                if (showRain && !testRainLayer) fetchRainTimes(false);
            }, 60000); 
        }
        
        function stopRainAutoUpdate() {
            if (rainAutoUpdateTimer) {
                clearInterval(rainAutoUpdateTimer);
                rainAutoUpdateTimer = null;
            }
        }

        document.getElementById('rain-slider').addEventListener('input', function(e) {
            if (testRainLayer) {
                map.removeLayer(testRainLayer);
                testRainLayer = null;
            }
            if (isRainPlaying) toggleRainAnimation();
            rainIndex = parseInt(e.target.value);
            updateRainDisplay();
        });

        // =========================================================
        // 4. 気象衛星（ひまわり）
        // =========================================================
        async function updateCloudLayer() {
            try {
                // ひまわりの最新時刻リストを取得
                const ts = new Date().getTime();
                const res = await fetch(`https://www.jma.go.jp/bosai/himawari/data/satimg/targetTimes_fd.json?_=${ts}`);
                const data = await res.json();
                
                // 最新のデータを使う
                const latest = data[data.length - 1];
                if (!latest) return;
                
                const validTime = latest.validtime; // YYYYMMDDHHmmss
                const url = `https://www.jma.go.jp/bosai/himawari/data/satimg/${validTime}/fd/{z}/{x}/{y}.jpg`;
                
                if (cloudLayer) map.removeLayer(cloudLayer);
                cloudLayer = L.tileLayer(url, {
                    opacity: 0.5, 
                    maxZoom: 18, 
                    attribution: '気象庁(ひまわり)'
                }).addTo(map);

            } catch(e) {
                console.error("ひまわりデータ取得エラー", e);
            }
        }

        // =========================================================
        // 5. レイヤー切り替え・UI制御
        // =========================================================
        function toggleLayer(type) {
            const guide = document.getElementById('guide-message');

            if (type === 'quake') {
                showQuake = !showQuake;
                const btn = document.getElementById('btn-quake');
                if (showQuake) {
                    btn.classList.add('active');
                    if(geojsonLayer) geojsonLayer.addTo(map);
                    if(hypocenterLayer) hypocenterLayer.addTo(map);
                } else {
                    btn.classList.remove('active');
                    if(geojsonLayer) map.removeLayer(geojsonLayer);
                    if(hypocenterLayer) map.removeLayer(hypocenterLayer);
                }
            } else if (type === 'rain') {
                showRain = !showRain;
                const btn = document.getElementById('btn-rain');
                const player = document.getElementById('rain-player');
                
                if (showRain) {
                    btn.classList.add('active-rain');
                    player.style.display = 'block';
                    // 初回ロード時は強制的に最新に合わせる
                    fetchRainTimes(true);
                    startRainAutoUpdate();
                    
                    guide.innerText = "☔ 現在の雨雲を表示中（色がなければ雨なし）";
                    guide.style.display = 'block';
                    setTimeout(() => { guide.style.display = 'none'; }, 5000);
                } else {
                    if (rainLayer) map.removeLayer(rainLayer);
                    if (testRainLayer) {
                        map.removeLayer(testRainLayer);
                        testRainLayer = null;
                    }
                    if (isRainPlaying) toggleRainAnimation();
                    stopRainAutoUpdate();
                    btn.classList.remove('active-rain');
                    player.style.display = 'none';
                }
            } else if (type === 'cloud') {
                showCloud = !showCloud;
                const btn = document.getElementById('btn-cloud');
                
                if (showCloud) {
                    btn.classList.add('active-cloud');
                    updateCloudLayer();
                    
                    guide.innerText = "☁️ 気象衛星(ひまわり)を表示中";
                    guide.style.display = 'block';
                    setTimeout(() => { guide.style.display = 'none'; }, 5000);
                } else {
                    btn.classList.remove('active-cloud');
                    if (cloudLayer) map.removeLayer(cloudLayer);
                }
            }
        }

        // =========================================================
        // 6. AIチャット機能 (Google Gemini)
        // =========================================================
        function toggleAiWindow() {
            const win = document.getElementById('ai-window');
            win.style.display = (win.style.display === 'flex') ? 'none' : 'flex';
        }

        function handleKeyPress(e) {
            if (e.key === 'Enter') sendMessage();
        }

        async function sendMessage() {
            const input = document.getElementById('user-input');
            const text = input.value.trim();
            if (!text) return;

            addMessage(text, 'user');
            input.value = '';
            document.getElementById('send-btn').disabled = true;

            const loadingMsgId = addMessage("考えています...", 'ai', true);

            let context = "あなたは親切なAI防災アドバイザーです。ユーザーの安全を第一に考え、簡潔に回答してください。\n";
            if (currentQuakeInfo) {
                context += `【現在の地震状況】\n発生時刻: ${currentQuakeInfo.time}\n震源地: ${currentQuakeInfo.hypo}\n最大震度: ${currentQuakeInfo.scaleText}\n`;
            } else {
                context += "現在、大きな地震データは表示されていません。\n";
            }
            if (showRain && rainTimeData.length > 0) context += "現在、ユーザーは雨雲レーダーを見ています。\n";
            if (showCloud) context += "現在、ユーザーは気象衛星画像を見ています。\n";
            
            context += "ユーザーの質問: " + text;

            try {
                const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: context }] }]
                    })
                });

                const data = await response.json();
                removeMessage(loadingMsgId);

                if (data.candidates && data.candidates[0].content) {
                    const reply = data.candidates[0].content.parts[0].text;
                    addMessage(reply, 'ai');
                } else {
                    addMessage("すみません、うまく答えられませんでした。", 'ai');
                }

            } catch (error) {
                console.error(error);
                removeMessage(loadingMsgId);
                addMessage("通信エラーが発生しました。", 'ai');
            } finally {
                document.getElementById('send-btn').disabled = false;
            }
        }

        function addMessage(text, sender, isLoading = false) {
            const area = document.getElementById('chat-area');
            const div = document.createElement('div');
            div.className = `message msg-${sender}`;
            div.innerHTML = text.replace(/\n/g, '<br>');
            if (isLoading) div.id = 'msg-loading-' + Date.now();
            area.appendChild(div);
            area.scrollTop = area.scrollHeight;
            return div.id;
        }

        function removeMessage(id) {
            const el = document.getElementById(id);
            if (el) el.remove();
        }

        // =========================================================
        // 7. AI分析レポート作成機能
        // =========================================================
        function closeReportModal() {
            document.getElementById('report-modal').style.display = 'none';
            document.getElementById('modal-overlay').style.display = 'none';
        }

        async function generateAiReport() {
            const modal = document.getElementById('report-modal');
            const overlay = document.getElementById('modal-overlay');
            const content = document.getElementById('report-content');
            
            modal.style.display = 'flex';
            overlay.style.display = 'block';
            content.innerHTML = '<div style="text-align:center; padding:20px;">AIが状況を分析しています...<br>お待ちください。</div>';

            let context = "あなたは防災のプロフェッショナルです。以下の情報を基に、一般市民が今すぐとるべき具体的な行動や注意点を分析し、レポートを作成してください。\n";
            context += "【重要】出力はHTML形式（<ul>, <li>, <strong>, <span>などを使用）で、見やすく簡潔な箇条書きにしてください。見出しは不要です。\n";
            
            if (currentQuakeInfo) {
                context += `[地震情報]\n発生時刻: ${currentQuakeInfo.time}\n震源地: ${currentQuakeInfo.hypo}\n最大震度: ${currentQuakeInfo.scaleText}\n`;
            } else {
                context += "[地震情報] 現在、大きな地震データは表示されていません。\n";
            }
            if (showRain && rainTimeData.length > 0) context += "[気象情報] 雨雲レーダーが表示されています。\n";
            if (showCloud) context += "[気象情報] 気象衛星画像が表示されています。\n";

            context += "ユーザーへの安心させるメッセージも一言添えてください。";

            try {
                const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({
                        contents: [{ parts: [{ text: context }] }]
                    })
                });

                const data = await response.json();
                
                if (data.candidates && data.candidates[0].content) {
                    const reportText = data.candidates[0].content.parts[0].text;
                    const cleanHtml = reportText.replace(/```html/g, '').replace(/```/g, '');
                    content.innerHTML = cleanHtml;
                } else {
                    content.innerHTML = "<p>レポートの生成に失敗しました。</p>";
                }

            } catch (error) {
                console.error(error);
                content.innerHTML = "<p>通信エラーが発生しました。</p>";
            }
        }

        // =========================================================
        // 8. 地震地図ロジック
        // =========================================================
        function getColor(scale) {
            if (!scale || scale < 10) return 'transparent'; 
            if (scale < 30) return '#00aaff';
            if (scale < 40) return '#00c400';
            if (scale < 45) return '#f6f600';
            if (scale < 55) return '#ffae00';
            return '#ff0000';
        }

        function getPrefName(feature) {
            if (!feature || !feature.properties) return "";
            const p = feature.properties;
            // 複数のデータ形式に対応できるようチェックを強化
            return p.name || p.name_ja || p.nam_ja || p.nam || p.prefecture || "";
        }

        function getScaleForPref(prefName) {
            let scale = 0;
            if (prefName) {
                Object.keys(prefShakeData).forEach(key => {
                    if (prefName.includes(key) || key.includes(prefName)) {
                        if (prefShakeData[key] > scale) scale = prefShakeData[key];
                    }
                });
            }
            return scale;
        }

        function styleFeature(feature) {
            const prefName = getPrefName(feature);
            const scale = getScaleForPref(prefName);
            const isShaking = scale >= 10;
            const currentZoom = map.getZoom();
            const isZoomedIn = currentZoom >= 7;

            return {
                fillColor: getColor(scale),
                weight: isZoomedIn ? 1 : (isShaking ? 2 : 1),
                opacity: 1,
                color: isShaking ? '#555' : '#ccc',
                dashArray: isShaking ? '' : '3',
                fillOpacity: isShaking ? (isZoomedIn ? 0.3 : 0.7) : 0 
            };
        }

        async function initMapData() {
            const loadingEl = document.getElementById('loading');
            loadingEl.style.display = 'block';
            try {
                // GeoJSONのURLを変更 (より安定したソースに変更: Code for Future Moms)
                const geoJsonUrl = 'https://raw.githubusercontent.com/code-for-future-moms/fetch-open-data/master/data/prefectures.geojson';
                
                const response = await fetch(geoJsonUrl);
                if (!response.ok) throw new Error(`Network Error: ${response.status}`);
                const data = await response.json();
                
                geojsonLayer = L.geoJson(data, {
                    style: styleFeature,
                    onEachFeature: function (feature, layer) {
                        const name = getPrefName(feature);
                        const scale = getScaleForPref(name);
                        let content = `<b>${name}</b>`;
                        if (scale > 0) content += `<br>震度: ${convertScaleToString(scale)}`;
                        layer.bindPopup(content);
                    }
                });
                
                if(showQuake) geojsonLayer.addTo(map);
                map.on('zoomend', function() {
                    if (geojsonLayer) geojsonLayer.setStyle(styleFeature);
                });

            } catch (error) {
                console.error("GeoJSON Error:", error);
                document.getElementById('quake-data').innerHTML = `
                    <div style="color:red;font-weight:bold;">地図データの読み込みに失敗しました</div>
                `;
            } finally {
                loadingEl.style.display = 'none';
            }
        }

        async function fetchQuakeData() {
            const infoDiv = document.getElementById('quake-data');
            infoDiv.style.opacity = '0.5';
            try {
                const ts = new Date().getTime();
                const res = await fetch(`https://api.p2pquake.net/v2/jma/quake?limit=1&_=${ts}`);
                const json = await res.json();
                if (json.length > 0) processQuakeData(json[0]);
                else infoDiv.innerHTML = "データなし";
            } catch (e) {
                console.error(e);
                infoDiv.innerText = "データ取得エラー";
            } finally {
                infoDiv.style.opacity = '1';
            }
        }

        function processQuakeData(data) {
            const infoDiv = document.getElementById('quake-data');
            if (data.code !== 551) {
                infoDiv.innerHTML = "最新情報は地震情報ではありません";
                return;
            }

            const time = data.earthquake.time;
            const hypo = data.earthquake.hypocenter.name;
            const maxScale = data.earthquake.maxScale;
            let lat = parseFloat(data.earthquake.hypocenter.latitude);
            let lng = parseFloat(data.earthquake.hypocenter.longitude);
            const scaleText = convertScaleToString(maxScale);

            currentQuakeInfo = { time, hypo, maxScale, scaleText };

            infoDiv.innerHTML = `
                📅 <strong>${time}</strong><br>
                📍 震源: <strong>${hypo}</strong><br>
                ⚡ 最大震度: <strong>${scaleText}</strong>
            `;

            prefShakeData = {};
            if (data.points) {
                data.points.forEach(point => {
                    const pref = point.pref;
                    const scale = point.scale;
                    if (!prefShakeData[pref] || prefShakeData[pref] < scale) {
                        prefShakeData[pref] = scale;
                    }
                });
            }

            if (geojsonLayer) geojsonLayer.setStyle(styleFeature);
            updateHypocenterMarker(lat, lng, hypo);
        }

        function updateHypocenterMarker(lat, lng, label) {
            if (hypocenterLayer) {
                map.removeLayer(hypocenterLayer);
                hypocenterLayer = null;
            }
            if (!isNaN(lat) && !isNaN(lng) && lat !== -200 && lng !== -200) {
                const svgIcon = `
                    <svg viewBox="0 0 100 100" class="epicenter-svg" xmlns="http://www.w3.org/2000/svg">
                        <path d="M20,20 L80,80 M80,20 L20,80" stroke="white" stroke-width="20" stroke-linecap="round" />
                        <path d="M20,20 L80,80 M80,20 L20,80" stroke="#d00" stroke-width="12" stroke-linecap="round" />
                    </svg>
                `;
                const icon = L.divIcon({
                    className: 'epicenter-icon',
                    html: svgIcon, iconSize: [40, 40], iconAnchor: [20, 20]
                });
                hypocenterLayer = L.marker([lat, lng], {icon: icon})
                    .bindPopup(`<b>震源地</b><br>${label}`);
                
                if(showQuake) hypocenterLayer.addTo(map);
            }
        }

        function convertScaleToString(scale) {
            switch(scale) {
                case 10: return "1"; case 20: return "2"; case 30: return "3";
                case 40: return "4"; case 45: return "5弱"; case 50: return "5強";
                case 55: return "6弱"; case 60: return "6強"; case 70: return "7";
                default: return "?";
            }
        }

        function loadTestData() {
            // 地震のテストデータ
            const dummyData = {
                code: 551,
                earthquake: {
                    time: "テスト表示中",
                    hypocenter: { name: "テスト震源（東京湾）", latitude: "35.5", longitude: "139.9" },
                    maxScale: 50
                },
                points: [
                    { pref: "東京都", scale: 50 }, { pref: "神奈川県", scale: 45 },
                    { pref: "千葉県", scale: 40 }, { pref: "埼玉県", scale: 40 },
                    { pref: "茨城県", scale: 30 }, { pref: "栃木県", scale: 20 },
                    { pref: "群馬県", scale: 20 }, { pref: "静岡県", scale: 30 },
                    { pref: "山梨県", scale: 20 }
                ]
            };
            processQuakeData(dummyData);

            // 雨雲のテストデータ
            if (showRain) {
                if (testRainLayer) map.removeLayer(testRainLayer);
                const group = L.layerGroup();
                const rainSpots = [
                    [35.6, 139.7], [35.0, 135.7], [43.0, 141.3], [33.5, 130.4]
                ];
                rainSpots.forEach(coords => {
                    L.circle(coords, {
                        color: 'transparent', fillColor: '#2980b9', fillOpacity: 0.5, radius: 50000
                    }).addTo(group);
                });
                testRainLayer = group.addTo(map);
                
                const guide = document.getElementById('guide-message');
                guide.innerText = "🧪 テストモード: 疑似的な雨雲を表示しています";
                guide.style.display = 'block';
                setTimeout(() => { guide.style.display = 'none'; }, 3000);
            }
        }

        initMapData().then(() => fetchQuakeData());
    </script>
</body>
</html>
