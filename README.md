# ChromecastTV.github.io
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Universal Cast Controller</title>
    <script src="https://www.gstatic.com/cv/js/sender/v1/cast_sender.js?loadCastFramework=1"></script>
    <style>
        :root { --accent: #00e676; --bg: #090909; --card: #151515; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); color: #eee; margin: 0; padding: 15px; }
        
        /* Panel de Conexión */
        .header { background: var(--card); padding: 20px; border-radius: 25px; text-align: center; margin-bottom: 15px; border: 1px solid #222; }
        google-cast-launcher { width: 45px; height: 45px; background: #222; border-radius: 50%; padding: 10px; cursor: pointer; }
        
        /* Pantalla de Preview */
        .display-screen { width: 100%; aspect-ratio: 16/9; background: #000; border-radius: 15px; margin-bottom: 15px; position: relative; overflow: hidden; border: 2px solid #333; }
        #localPreview { width: 100%; height: 100%; object-fit: contain; }
        .badge { position: absolute; top: 10px; left: 10px; background: rgba(0,230,118,0.2); color: var(--accent); padding: 4px 10px; border-radius: 5px; font-size: 10px; border: 1px solid var(--accent); }

        /* Control Remoto */
        .remote { background: var(--card); border-radius: 30px; padding: 25px; box-shadow: 0 10px 30px rgba(0,0,0,0.8); }
        .info { text-align: center; margin-bottom: 20px; }
        #fileName { font-weight: bold; font-size: 14px; color: var(--accent); }

        .timeline { margin-bottom: 20px; }
        input[type="range"] { width: 100%; accent-color: var(--accent); height: 8px; border-radius: 5px; }
        .times { display: flex; justify-content: space-between; font-size: 11px; color: #777; margin-top: 8px; }

        .playback-keys { display: flex; justify-content: center; align-items: center; gap: 25px; margin-bottom: 25px; }
        .btn-main { width: 70px; height: 70px; border-radius: 50%; border: none; background: var(--accent); color: #000; font-size: 28px; font-weight: bold; }
        .btn-sec { width: 50px; height: 50px; border-radius: 50%; border: none; background: #222; color: #fff; font-size: 20px; }

        .vol-control { display: flex; align-items: center; gap: 15px; background: #1a1a1a; padding: 12px 20px; border-radius: 40px; }

        /* Selectores de Archivo */
        .footer-tools { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-top: 20px; }
        .tool-btn { background: #1a1a1a; border: 1px solid #333; color: white; padding: 15px; border-radius: 15px; font-size: 12px; font-weight: bold; }
    </style>
</head>
<body>

    <div class="header">
        <google-cast-launcher></google-cast-launcher>
        <div id="status" style="font-size: 11px; color: #666; margin-top: 8px;">DISPOSITIVO NO VINCULADO</div>
    </div>

    <div class="display-screen">
        <div class="badge">LIVE PREVIEW</div>
        <video id="localPreview" muted playsinline></video>
    </div>

    <div class="remote">
        <div class="info">
            <div id="fileName">Esperando archivo...</div>
        </div>

        <div class="timeline">
            <input type="range" id="seekSlider" value="0" min="0" step="1">
            <div class="times">
                <span id="currentTime">00:00</span>
                <span id="totalDuration">00:00</span>
            </div>
        </div>

        <div class="playback-keys">
            <button class="btn-sec" onclick="mediaControl('rewind')">↺</button>
            <button class="btn-main" id="playBtn" onclick="mediaControl('play')">▶</button>
            <button class="btn-sec" onclick="mediaControl('forward')">↻</button>
        </div>

        <div class="vol-control">
            <span>🔈</span>
            <input type="range" id="volSlider" min="0" max="100" value="50" oninput="changeVolume(this.value)">
            <span>🔊</span>
        </div>
    </div>

    <div class="footer-tools">
        <button class="tool-btn" onclick="document.getElementById('pick-video').click()">🎬 TRANSMITIR VIDEO
            <input type="file" id="pick-video" hidden accept="video/*" onchange="loadMedia(this)">
        </button>
        <button class="tool-btn" onclick="document.getElementById('pick-audio').click()">🎵 TRANSMITIR AUDIO
            <input type="file" id="pick-audio" hidden accept="audio/*" onchange="loadMedia(this)">
        </button>
    </div>

    <script>
        let castSession, player, controller, currentMediaURL;

        // Inicialización del Motor de Google Cast
        window.__onGCastApiAvailable = function(isAvailable) {
            if (isAvailable) {
                const context = cast.framework.CastContext.getInstance();
                context.setOptions({
                    receiverApplicationId: chrome.cast.media.DEFAULT_MEDIA_RECEIVER_APP_ID,
                    autoJoinPolicy: chrome.cast.AutoJoinPolicy.ORIGIN_SCOPED
                });

                player = new cast.framework.RemotePlayer();
                controller = new cast.framework.RemotePlayerController(player);

                // Escuchar cambios en la TV y reflejarlos en el móvil
                controller.addEventListener(cast.framework.RemotePlayerEventType.ANY_CHANGE, () => {
                    document.getElementById('seekSlider').max = player.duration;
                    document.getElementById('seekSlider').value = player.currentTime;
                    document.getElementById('currentTime').innerText = formatTime(player.currentTime);
                    document.getElementById('totalDuration').innerText = formatTime(player.duration);
                    document.getElementById('playBtn').innerText = player.isPaused ? "▶" : "⏸";
                });
            }
        };

        async function loadMedia(input) {
            const file = input.files[0];
            if (!file) return;

            castSession = cast.framework.CastContext.getInstance().getCurrentSession();
            if (!castSession) {
                alert("Brother, primero vincula la TV con el botón de arriba.");
                return;
            }

            document.getElementById('fileName').innerText = "Procesando: " + file.name;
            
            // Creamos la URL del archivo
            currentMediaURL = URL.createObjectURL(file);
            
            // Preview Local
            const localPlayer = document.getElementById('localPreview');
            localPlayer.src = currentMediaURL;
            localPlayer.play();

            // Configurar Metadata para la TV
            const mediaInfo = new chrome.cast.media.MediaInfo(currentMediaURL, file.type);
            mediaInfo.metadata = new chrome.cast.media.GenericMediaMetadata();
            mediaInfo.metadata.title = file.name;
            mediaInfo.metadata.images = [{url: 'https://cdn-icons-png.flaticon.com/512/3159/3159461.png'}];

            const request = new chrome.cast.media.LoadRequest(mediaInfo);
            
            try {
                await castSession.loadMedia(request);
                document.getElementById('status').innerText = "TRANSMITIENDO AHORA";
            } catch (e) {
                console.error(e);
                alert("Error: El Chromecast no puede acceder al archivo local. Mira la nota abajo.");
            }
        }

        function mediaControl(action) {
            if (action === 'play') controller.playOrPause();
            if (action === 'rewind') { player.currentTime -= 10; controller.seek(); }
            if (action === 'forward') { player.currentTime += 10; controller.seek(); }
        }

        function changeVolume(v) {
            player.volumeLevel = v / 100;
            controller.setVolumeLevel();
        }

        document.getElementById('seekSlider').oninput = function() {
            player.currentTime = this.value;
            controller.seek();
        };

        function formatTime(s) {
            if (!s) return "00:00";
            let min = Math.floor(s / 60);
            let sec = Math.floor(s % 60);
            return `${min.toString().padStart(2, '0')}:${sec.toString().padStart(2, '0')}`;
        }
    </script>
</body>
</html>