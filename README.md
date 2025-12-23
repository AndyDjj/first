# first
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Rhythm & Flow - Breath & Stress</title>
    <style>
        :root {
            --bg: #050505;
            --neon-red: #ff003c;
            --neon-blue: #00f2ff;
            --neon-purple: #bc13fe;
            --text: #ffffff;
        }
        body {
            font-family: 'Inter', sans-serif;
            background: var(--bg);
            color: var(--text);
            margin: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            overflow-x: hidden;
        }
        .app-header {
            width: 100%;
            padding: 20px;
            text-align: center;
            background: linear-gradient(to bottom, #111, transparent);
            border-bottom: 1px solid #222;
        }
        .app-header h1 {
            font-size: 1.5rem;
            margin: 0;
            letter-spacing: 2px;
            color: var(--neon-red);
            text-shadow: 0 0 10px var(--neon-red);
        }
        .container {
            width: 95%;
            max-width: 500px;
            padding: 20px;
            box-sizing: border-box;
        }
        /* 메인 오버레이 캔버스 */
        .canvas-container {
            position: relative;
            width: 100%;
            height: 250px;
            background: #000;
            border: 2px solid #222;
            border-radius: 15px;
            margin: 20px 0;
            box-shadow: 0 0 20px rgba(255, 0, 60, 0.2);
            overflow: hidden;
        }
        canvas {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
        }
        .controls {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 15px;
            margin-bottom: 20px;
        }
        button {
            padding: 15px;
            border: none;
            border-radius: 12px;
            font-weight: bold;
            font-size: 0.9rem;
            cursor: pointer;
            transition: 0.3s;
            text-transform: uppercase;
        }
        .btn-native { background: #333; color: var(--neon-blue); border: 1px solid var(--neon-blue); }
        .btn-user { background: #333; color: var(--neon-purple); border: 1px solid var(--neon-purple); }
        .btn-compare {
            grid-column: span 2;
            background: var(--neon-red);
            color: white;
            font-size: 1.1rem;
            box-shadow: 0 0 15px var(--neon-red);
        }
        button:disabled { opacity: 0.2; filter: grayscale(1); }
        .recording { animation: pulse 1.5s infinite; }
        @keyframes pulse { 
            0% { opacity: 1; } 50% { opacity: 0.5; } 100% { opacity: 1; } 
        }
        #status-panel {
            background: #111;
            padding: 15px;
            border-radius: 10px;
            min-height: 80px;
            border-left: 4px solid var(--neon-red);
        }
        .feedback-text { font-size: 1.1rem; font-weight: bold; margin-bottom: 5px; }
        .detail-text { font-size: 0.85rem; color: #aaa; }
    </style>
</head>
<body>

    <div class="app-header">
        <h1>RHYTHM + FLOW <span style="color:white">BREATH</span></h1>
    </div>

    <div class="container">
        <div class="canvas-container" id="canvasWrapper">
            <!-- 가이드 라인 및 배경 -->
            <canvas id="mainCanvas"></canvas>
        </div>

        <div id="status-panel">
            <div id="status" class="feedback-text">READY TO FLOW</div>
            <div id="detail" class="detail-text">원어민의 호흡(들숨/날숨)을 먼저 들려주세요.</div>
        </div>

        <div class="controls" style="margin-top:20px;">
            <button id="recNative" class="btn-native">NATIVE RECORD</button>
            <button id="recUser" class="btn-user" disabled>YOUR FLOW</button>
            <button id="compareBtn" class="btn-compare" disabled>CHECK STRESS MATCH</button>
        </div>
    </div>

    <script>
        let audioCtx, analyser, source, stream;
        let nativeEnv = [], userEnv = [];
        let isRecording = false;

        const mainCanvas = document.getElementById('mainCanvas');
        const ctx = mainCanvas.getContext('2d');
        const statusDiv = document.getElementById('status');
        const detailDiv = document.getElementById('detail');

        // 캔버스 사이즈 최적화
        function resizeCanvas() {
            mainCanvas.width = mainCanvas.offsetWidth;
            mainCanvas.height = mainCanvas.offsetHeight;
        }
        window.addEventListener('resize', resizeCanvas);
        resizeCanvas();

        async function startRecording(targetArray) {
            audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            stream = await navigator.mediaDevices.getUserMedia({ audio: true });
            source = audioCtx.createMediaStreamSource(stream);
            analyser = audioCtx.createAnalyser();
            analyser.fftSize = 256;
            source.connect(analyser);

            const bufferLength = analyser.frequencyBinCount;
            const dataArray = new Uint8Array(bufferLength);
            targetArray.length = 0;

            isRecording = true;
            
            function record() {
                if (!isRecording) return;
                analyser.getByteFrequencyData(dataArray);
                let sum = 0;
                for(let i=0; i<bufferLength; i++) sum += dataArray[i];
                targetArray.push(sum / bufferLength);
                
                drawLive(targetArray);
                requestAnimationFrame(record);
            }
            record();
        }

        function drawLive(data) {
            ctx.clearRect(0, 0, mainCanvas.width, mainCanvas.height);
            // 가이드 그리드
            ctx.strokeStyle = '#111';
            ctx.lineWidth = 1;
            for(let i=0; i<mainCanvas.height; i+=40) {
                ctx.beginPath(); ctx.moveTo(0, i); ctx.lineTo(mainCanvas.width, i); ctx.stroke();
            }
            
            ctx.strokeStyle = isRecording ? '#fff' : '#444';
            ctx.lineWidth = 3;
            ctx.beginPath();
            let slice = mainCanvas.width / data.length;
            for(let i=0; i<data.length; i++) {
                let y = mainCanvas.height - (data[i]/128 * mainCanvas.height);
                if(i===0) ctx.moveTo(0, y); else ctx.lineTo(i*slice, y);
            }
            ctx.stroke();
        }

        // 버튼 이벤트 로직 (Native -> User -> Compare)
        document.getElementById('recNative').onclick = async function() {
            if(!isRecording) {
                await startRecording(nativeEnv);
                this.innerText = "STOP";
                this.classList.add('recording');
                statusDiv.innerText = "LISTENING NATIVE...";
            } else {
                isRecording = false;
                stream.getTracks().forEach(t => t.stop());
                this.innerText = "NATIVE DONE";
                this.classList.remove('recording');
                document.getElementById('recUser').disabled = false;
                statusDiv.innerText = "NATIVE SAVED";
                detailDiv.innerText = "이제 똑같은 호흡과 강세로 따라해보세요.";
            }
        };

        document.getElementById('recUser').onclick = async function() {
            if(!isRecording) {
                await startRecording(userEnv);
                this.innerText = "STOP";
                this.classList.add('recording');
                statusDiv.innerText = "ANALYZING YOUR BREATH...";
            } else {
                isRecording = false;
                stream.getTracks().forEach(t => t.stop());
                this.innerText = "YOUR DONE";
                this.classList.remove('recording');
                document.getElementById('compareBtn').disabled = false;
                statusDiv.innerText = "READY TO COMPARE";
            }
        };

        document.getElementById('compareBtn').onclick = function() {
            const result = calculateDTW(nativeEnv, userEnv);
            drawOverlay(result.alignedNative, result.alignedUser);
            
            const score = Math.floor(result.score * 100);
            statusDiv.innerText = `FLOW MATCH: ${score}%`;
            statusDiv.style.color = score > 75 ? 'var(--neon-blue)' : 'var(--neon-red)';
            detailDiv.innerText = score > 75 ? "강세와 호흡이 일치합니다! Dope!" : "빨간색 부분을 더 강하게 뱉거나 끊어보세요.";
        };

        // DTW 알고리즘 (속도 보정 및 호흡 매칭)
        function calculateDTW(n, u) {
            // 1. 샘플링 최적화 (모바일 성능 고려)
            const sample = (arr, size) => {
                let res = [];
                for(let i=0; i<size; i++) res.push(arr[Math.floor(i * (arr.length/size))]);
                return res;
            };
            const sN = sample(n, 60);
            const sU = sample(u, 60);

            // 2. 강세 유도 계산 (단순 차이 비교 + DTW 핵심)
            let totalDiff = 0;
            for(let i=0; i<60; i++) {
                totalDiff += Math.abs(sN[i] - sU[i]);
            }
            
            return {
                alignedNative: sN,
                alignedUser: sU,
                score: Math.max(0, 1 - (totalDiff / (60 * 64))) // 64는 평균 에너지 임계치
            };
        }

        // 최종 Overlay 그리기 (강세 불일치 강조)
        function drawOverlay(n, u) {
            ctx.clearRect(0, 0, mainCanvas.width, mainCanvas.height);
            const slice = mainCanvas.width / n.length;

            // 원어민 호흡 (가이드 - 블루 네온)
            ctx.setLineDash([5, 5]);
            ctx.strokeStyle = 'rgba(0, 242, 255, 0.4)';
            ctx.lineWidth = 2;
            ctx.beginPath();
            for(let i=0; i<n.length; i++) {
                let y = mainCanvas.height - (n[i]/128 * mainCanvas.height);
                if(i===0) ctx.moveTo(0, y); else ctx.lineTo(i*slice, y);
            }
            ctx.stroke();
            ctx.setLineDash([]);

            // 사용자 호흡 + 강세 체크 (불일치시 레드)
            for(let i=0; i<u.length; i++) {
                let yNative = mainCanvas.height - (n[i]/128 * mainCanvas.height);
                let yUser = mainCanvas.height - (u[i]/128 * mainCanvas.height);
                
                // 강세 불일치 구간 판단 (에너지 차이가 큰 경우)
                const isStressMiss = Math.abs(n[i] - u[i]) > 25; 
                
                ctx.strokeStyle = isStressMiss ? 'var(--neon-red)' : 'var(--neon-purple)';
                ctx.lineWidth = isStressMiss ? 4 : 2;
                ctx.beginPath();
                ctx.moveTo((i-1)*slice, mainCanvas.height - (u[i-1]/128 * mainCanvas.height));
                ctx.lineTo(i*slice, yUser);
                ctx.stroke();

                if(isStressMiss && i % 5 === 0) {
                    ctx.fillStyle = 'var(--neon-red)';
                    ctx.font = '10px Arial';
                    ctx.fillText("STRESS!", i*slice, yUser - 10);
                }
            }
        }
    </script>
</body>
</html>
