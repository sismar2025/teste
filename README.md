<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Monitoramento de Movimento</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            margin-top: 50px;
            transition: background-color 0.5s;
        }
        button {
            padding: 15px;
            font-size: 18px;
            cursor: pointer;
            margin-top: 20px;
            border: none;
            border-radius: 8px;
        }
        #calibrateButton {
            background-color: #d3d3d3;
        }
        #startButton, #resetButton {
            background-color: #4CAF50;
            color: white;
            display: none;
        }
        #message, #countdownMessage, #vibrationData {
            font-size: 20px;
            margin-top: 20px;
        }
        #alarmMessage {
            font-size: 40px;
            color: red;
            font-weight: bold;
            display: none;
            animation: blink 1s infinite;
        }
        @keyframes blink {
            50% { opacity: 0; }
        }
        body.alarm-active {
            animation: bgBlink 1s infinite;
        }
        @keyframes bgBlink {
            0%, 100% { background-color: white; }
            50% { background-color: yellow; }
        }
    </style>
</head>
<body>

    <h1>Monitoramento de Movimento</h1>
    <button id="calibrateButton">Calibrar</button>
    <button id="startButton">Iniciar Monitoramento</button>
    <button id="resetButton">Reiniciar</button>
    <div id="message"></div>
    <div id="countdownMessage"></div>
    <div id="vibrationData"></div>
    <div id="alarmMessage">ALARME ATIVADO!</div>

    <script>
        let maxVibration = 0;
        let calibrationInProgress = false;
        let monitoringInProgress = false;
        let countdownTimer;

        const recipients = ['+5512997951434', '+5512997370493'];
        const alertMessage = "CUIDADO DESLIZAMENTO DE TERRA DETECTADO";

        const googleScriptURL = 'COLE_AQUI_O_SEU_URL_DO_GOOGLE_SCRIPT'; // Substitua pelo URL do Web App

        async function enviarParaPlanilha(vibration, alarm = false) {
            const data = {
                vibration: vibration.toFixed(3),
                alarm: alarm
            };

            try {
                const response = await fetch(googleScriptURL, {
                    method: 'POST',
                    body: JSON.stringify(data),
                    headers: {
                        'Content-Type': 'application/json'
                    }
                });
                const resultado = await response.text();
                console.log('Resposta do Google Sheets:', resultado);
            } catch (error) {
                console.error('Erro ao enviar dados para planilha:', error);
            }
        }

        async function sendWhatsAppMessage(to, message) {
            const url = 'https://api.z-api.io/instances/3E1A3374AEAEC04531A4FECBD5821189/token/0FCEA8567EF42F67DB49B241/send-text';
            const data = { phone: to, message: message };

            try {
                const response = await fetch(url, {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify(data)
                });
                const result = await response.json();
                console.log('Mensagem enviada via WhatsApp:', result);
            } catch (error) {
                console.error('Erro ao enviar mensagem:', error);
            }
        }

        function generateOscillatingTone(duration = 10) {
            const audioContext = new (window.AudioContext || window.webkitAudioContext)();
            const oscillator = audioContext.createOscillator();
            const gainNode = audioContext.createGain();

            oscillator.connect(gainNode);
            gainNode.connect(audioContext.destination);
            oscillator.type = 'sine';
            oscillator.frequency.setValueAtTime(800, audioContext.currentTime);

            const startTime = audioContext.currentTime;
            oscillator.frequency.linearRampToValueAtTime(1200, startTime + duration / 2);
            oscillator.frequency.linearRampToValueAtTime(800, startTime + duration);

            oscillator.start(startTime);
            oscillator.stop(startTime + duration);
        }

        function handleMotion(event) {
            if (!monitoringInProgress) return;

            const acc = event.acceleration || event.accelerationIncludingGravity;
            if (!acc) return;

            const vibration = Math.abs(acc.x) + Math.abs(acc.y) + Math.abs(acc.z);
            document.getElementById('vibrationData').innerText = Vibração atual: ${vibration.toFixed(3)};

            if (vibration > maxVibration * 1.2) {
                triggerAlarm(vibration);
            }
        }

        function triggerAlarm(vibration) {
            monitoringInProgress = false;

            generateOscillatingTone(10);
            const sirene = new Audio("/mnt/data/sirene-boa-207574.mp3");
            sirene.play();

            document.getElementById('alarmMessage').style.display = 'block';
            document.body.classList.add('alarm-active');

            recipients.forEach(to => sendWhatsAppMessage(to, alertMessage));
            enviarParaPlanilha(vibration, true); // Envia para a planilha + email

            setTimeout(() => {
                document.getElementById('alarmMessage').style.display = 'none';
                document.body.classList.remove('alarm-active');
                document.getElementById('resetButton').style.display = 'inline-block';
            }, 10000);

            window.removeEventListener('devicemotion', handleMotion, false);
        }

        function calibrateMotion(event) {
            const acc = event.acceleration || event.accelerationIncludingGravity;
            if (!acc) return;

            const vibration = Math.abs(acc.x) + Math.abs(acc.y) + Math.abs(acc.z);
            document.getElementById('vibrationData').innerText = Vibração atual: ${vibration.toFixed(3)};

            if (vibration > maxVibration) {
                maxVibration = vibration;
            }
        }

        function startCountdown() {
            let countdown = 5;
            document.getElementById('countdownMessage').innerText = Iniciando em ${countdown}...;
            countdownTimer = setInterval(() => {
                countdown--;
                document.getElementById('countdownMessage').innerText = Iniciando em ${countdown}...;
                if (countdown === 0) {
                    clearInterval(countdownTimer);
                    document.getElementById('countdownMessage').innerText = 'Monitoramento ativado!';
                    document.getElementById('startButton').style.display = 'inline-block';
                }
            }, 1000);
        }

        document.getElementById('calibrateButton').onclick = function () {
            if (calibrationInProgress) return;
            calibrationInProgress = true;
            maxVibration = 0;

            document.getElementById('message').innerText = 'Calibrando...';
            window.addEventListener('devicemotion', calibrateMotion, false);

            setTimeout(() => {
                window.removeEventListener('devicemotion', calibrateMotion, false);
                calibrationInProgress = false;
                document.getElementById('message').innerText = 'Calibração finalizada.';
                document.getElementById('vibrationData').innerText = '';
                startCountdown();
            }, 5000);
        };

        document.getElementById('startButton').onclick = function () {
            monitoringInProgress = true;
            document.getElementById('message').innerText = 'Monitorando...';
            this.style.display = 'none';
            window.addEventListener('devicemotion', handleMotion, false);
        };

        document.getElementById('resetButton').onclick = function () {
            location.reload();
        };
    </script>

</body>
</html>
