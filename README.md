# ChromecastTV.github.io
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Ultimate Cast Controller</title>
    <script src="https://www.gstatic.com/cv/js/sender/v1/cast_sender.js?loadCastFramework=1"></script>
    <style>
        :root { --p: #6200ee; --s: #03dac6; --bg: #0a0a0a; --card: #1a1a1a; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); color: #fff; margin: 0; padding: 15px; display: flex; flex-direction: column; align-items: center; }
        
        /* Header y Vinculación */
        .header { width: 100%; background: var(--card); padding: 20px; border-radius: 20px; text-align: center; margin-bottom: 15px; box-shadow: 0 4px 20px rgba(0,0,0,0.5); }
        google-cast-launcher { width: 50px; height: 50px; --connected-color: var(--s); cursor: pointer; }

        /* Vista Previa Real */
        .preview-container { width: 100%; max-width: 400px; aspect-ratio: 16/9; background: #000; border-radius: 15px; overflow: hidden; margin-bottom: 15px; border: 2px solid #333; position: relative; }
        #localPlayer { width: 100%; height: 100%; object-fit: contain; }
        .preview-label { position: absolute; top: 10px; left: 10px; background: rgba(0,0,0,0.6); padding: 2px 8px; border-radius: 5px; font-size: 10px; color: var(--s); border: 1px solid var(--s); }

        /* Control Remoto */
        .remote-control { width: 100%; max-width: 400px; background: var(--card); border-radius: 25px; padding: 20px; box-sizing: border-box; }
        .track-info { text-align: center; margin-bottom: 15px; font-weight: bold; color: var(--s); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
        
        .progress-area { margin-bottom: 20px; }
        input[type="range"] { width: 100%; accent-color: var(--p); }
        .time-labels { display: flex; justify-content: space-between; font-size: 12px; color: #888; margin-top: 5px; }

        .main-controls { display: flex; justify-content: space-around; align-items: center; margin-bottom: 20px; }
        .btn-circle { width: 60px; height: 60px; border-radius: 50%; border: none; background: #333; color: white; font-size: 24px; display: flex; align-items: center; justify-content: center; transition: 0.2s; }
        .btn-circle:active { transform: scale(0.9); background: var(--p); }
        .play-btn { width: 80px; height: 80px; background: var(--p); font-size: 30px; }

        .volume-area { display: flex; align-items: center; gap: 10px; background: #222; padding: 10px 15px; border-radius: 50px; }
        
        /* Botones de Selección */
        .file-selector { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; width: 100%; max-width: 400px; margin-top: 15px; }
        .sel-btn { background: #252525; border: 1px solid #444; color: white; padding: 15px; border-radius: 15px; font-weight: bold; display: flex; flex-direction: column; align-items: center; gap: 5px; }
        .sel-btn:active { border-color: var(--s); }
    </style>
</head>
<body>

    <div class="header">
        <google-cast-launcher></google-cast-launcher>
        <div id="cast-status" style="font-size: 12px; margin-top: 5px; color: #888;">Toca para vincular TV</div>
    </div>

    <div class="preview-container">
        <div class="preview-label">VISTA PREVIA MÓVIL</div>
        <video id="localPlayer" playsinline muted></video>
    </div>

    <div class="remote-control">
        <div id="trackName" class="track-info">Ningún archivo seleccionado</div>
        
        <div class="progress-area">
            <input type="range" id="progressBar" value="0" step="1">
            <div class="time-labels">
                <span id="currTime">00:00</span>
                <span id="totalTime">00:00</span>
            </div>
        </div>

        <div class="main-controls">
            <button class="btn-circle" onclick="seekRelative(-10)">⏪</button>
            <button class="btn-circle play-btn" id="playPauseBtn" onclick="togglePlay()">▶️</button>
            <button class="btn-circle" onclick="seekRelative(10)">⏩</button>
        </div>

        <div class="volume-area">
            <span>🔈</span>
            <input type="range" id="volumeBar" min="0" max="100" value="50" oninput="setVolume(this.value)">
            <span>🔊</span>
        </div>
    </div>

    <div class="file-selector">
        <button class="sel-btn" onclick="document.getElementById('f-audio').click()">
            <span>🎵</span> AUDIO
            <input type="file" id="f-audio" hidden accept="audio/*" onchange="handleMedia(this)">
        </button>
        <button class="sel-btn" onclick="document.getElementById('f-video').click()">
            <span>🎬</span> VIDEO
            <input type="file" id="f-video" hidden accept="video/*" onchange="handleMedia(this)">
        </button>
    </div>

    <script>
        let castSession, player, controller, localPlayer;

        window.__onGCastApiAvailable = function(isAvailable) {
            if (isAvailable) {
                const context = cast.framework.CastContext.getInstance();
                context.setOptions({
                    receiverApplicationId: chrome.cast.media.DEFAULT_MEDIA_RECEIVER_APP_ID,
                    autoJoinPolicy: chrome.cast.AutoJoinPolicy.ORIGIN_SCOPED
                });

                player = new cast.framework.RemotePlayer();
                controller = new cast.framework.RemotePlayerController(player);
                localPlayer = document.getElementById('localPlayer');

                // Sincronización automática de controles
                controller.addEventListener(cast.framework.RemotePlayerEventType.ANY_CHANGE, (e) => {
                    document.getElementById('progressBar').max = player.duration;
                    document.getElementById('progressBar').value = player.currentTime;
                    document.getElementById('currTime').innerText = formatTime(player.currentTime);
                    document.getElementById('totalTime').innerText = formatTime(player.duration);
                    document.getElementById('playPauseBtn').innerText = player.isPaused ? "▶️" : "⏸️";
                });
            }
        };

        async function handleMedia(input) {
            const file = input.files[0];
            if (!file) return;

            const session = cast.framework.CastContext.getInstance().getCurrentSession();
            if (!session) {
                alert("Primero vincula la TV con el botón de arriba");
                return;
            }

            const url = URL.createObjectURL(file);
            document.getElementById('trackName').innerText = file.name;

            // Iniciar vista previa local
            localPlayer.src = url;
            localPlayer.play();

            // Enviar a Chromecast
            const mediaInfo = new chrome.cast.media.MediaInfo(url, file.type);
            mediaInfo.metadata = new chrome.cast.media.GenericMediaMetadata();
            mediaInfo.metadata.title = file.name;
            
            const request = new chrome.cast.media.LoadRequest(mediaInfo);
            
            try {
                await session.loadMedia(request);
                console.log("Carga exitosa en TV");
            } catch (e) {
                console.error("Error: Probablemente bloqueo de red local.", e);
                alert("Error de conexión. Asegúrate de usar HTTPS y estar en la misma red.");
            }
        }

        function togglePlay() { controller.playOrPause(); }
        function setVolume(v) { player.volumeLevel = v / 100; controller.setVolumeLevel(); }
        function seekRelative(sec) {
            let newTime = player.currentTime + sec;
            player.currentTime = newTime;
            controller.seek();
        }

        function formatTime(s) {
            if (isNaN(s)) return "00:00";
            const m = Math.floor(s / 60);
            const sec = Math.floor(s % 60);
            return `${m.toString().padStart(2, '0')}:${sec.toString().padStart(2, '0')}`;
        }

        // Sincronizar barra de progreso al arrastrar
        document.getElementById('progressBar').oninput = function() {
            player.currentTime = this.value;
            controller.seek();
        };
    </script>
</body>
</html>