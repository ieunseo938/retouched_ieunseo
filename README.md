<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>RETOUCHED: Final Fixed Layout</title>
    
    <!-- 폰트: 고운바탕 (파일 없이 인터넷으로 바로 적용) -->
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Gowun+Batang:wght@400;700&display=swap" rel="stylesheet">
    
    <!-- MediaPipe Hands -->
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>

    <style>
        /* 기본 설정 */
        body, html {
            margin: 0; padding: 0; width: 100%; height: 100%;
            background-color: #ffffff; /* 흰색 배경 */
            overflow: hidden;
            font-family: 'Gowun Batang', serif; /* 고운바탕체 강제 적용 */
            cursor: pointer;
            touch-action: none;
        }

        /* 3D 무대 (화면 전체) */
        .scene {
            position: absolute;
            top: 0; left: 0;
            width: 100vw; height: 100vh;
            perspective: 2000px;
            overflow: hidden;
            z-index: 1; /* 글자보다 뒤에 위치 */
        }

        /* 카드가 모이는 세상 (화면 정중앙 고정) */
        .world {
            position: absolute;
            top: 50%; left: 50%; /* 무조건 화면 중앙 */
            width: 0; height: 0;
            transform-style: preserve-3d;
            z-index: 10;
        }

        /* 텍스트 오버레이 (왼쪽 상단 고정, 절대 안 겹치게 z-index 높임) */
        .overlay-text {
            position: absolute; 
            top: 40px; left: 40px; 
            z-index: 9999; /* 제일 위에 표시 */
            width: 350px; /* 너비 고정해서 레이아웃 유지 */
            color: #ff0000; 
            pointer-events: none; /* 클릭 통과 */
            background: rgba(255, 255, 255, 0.8); /* 글자 잘 보이게 살짝 배경 깖 */
            padding: 20px;
            border-radius: 10px;
        }

        .overlay-text h1 { 
            font-size: 1.8rem; margin: 0 0 20px 0; letter-spacing: 1px; font-weight: bold; color: #ff0000;
        }
        .overlay-text p { 
            font-size: 1rem; line-height: 1.6; color: #ff0000; margin-bottom: 20px;
        }
        .highlight { font-weight: normal; text-decoration: none; }

        /* 사용법 안내 */
        .usage-guide { margin-top: 30px; border-top: 1px solid #ffcccc; padding-top: 20px; }
        .usage-title { font-weight: bold; font-size: 1.1rem; margin-bottom: 10px; display: block; color: #ff0000; }
        .usage-list { padding: 0; margin: 0; list-style: none; }
        .usage-list li { margin-bottom: 8px; word-break: keep-all; color: #ff0000; font-size: 0.95rem; }

        /* 아이콘 범례 */
        .icon-legend {
            display: flex; justify-content: flex-start; gap: 20px;
            margin-top: 20px; pointer-events: auto;
        }
        .legend-item {
            display: flex; flex-direction: column; align-items: center;
            font-size: 0.9rem; color: #ff0000; opacity: 0.7; cursor: pointer;
            transition: transform 0.2s, opacity 0.2s;
        }
        .legend-item:hover, .legend-item.active { opacity: 1; transform: scale(1.1); font-weight: bold; }
        .legend-icon { width: 30px; height: 30px; margin-bottom: 5px; }

        /* 카메라 피드 (배경으로 숨김) */
        .input_video {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; object-fit: cover;
            opacity: 0.05; filter: grayscale(100%) contrast(1.2); z-index: 0; pointer-events: none;
            transform: scaleX(-1);
        }

        .loading-msg {
            position: absolute; bottom: 20px; right: 20px;
            color: #ff0000; font-size: 0.8rem; opacity: 0.7;
            animation: blink 1s infinite; font-family: sans-serif;
            z-index: 50;
        }
        @keyframes blink { 50% { opacity: 0.3; } }

        /* 개별 카드 스타일 */
        .card {
            position: absolute;
            /* 중심점 기준 정렬 */
            left: -85px; top: -102px; 
            width: 170px; height: 204px; /* 크기 고정 */
            border-radius: 3px;
            transform-style: preserve-3d;
            box-shadow: 0 5px 20px rgba(0,0,0,0.1);
            transition: transform 1.5s cubic-bezier(0.2, 0.8, 0.2, 1);
        }
        .card.hidden { opacity: 0 !important; pointer-events: none; transform: scale(0) !important; }
        .card-inner { position: relative; width: 100%; height: 100%; text-align: center; transition: transform 0.6s; transform-style: preserve-3d; }
        .card.flipped .card-inner { transform: rotateY(180deg); }
        .card.revealed .card-front { background-size: contain; filter: grayscale(0%) contrast(1); }
        .card-front, .card-back { position: absolute; width: 100%; height: 100%; backface-visibility: hidden; border-radius: 3px; }
        .card-front { background-color: #1a1a1a; background-size: contain; background-repeat: no-repeat; background-position: center; filter: grayscale(100%) contrast(1.1); transition: filter 0.5s; }
        
        .card-back {
            background-color: #f5f5f5; transform: rotateY(180deg); border: 1px solid #e0e0e0;
            display: flex; justify-content: center; align-items: center; 
            box-sizing: border-box; color: #ff0000; overflow: hidden; position: relative;
            background-size: 40px 40px; background-repeat: no-repeat; background-position: center center;
        }

        /* 설명 텍스트 (7pt 고정) */
        .description-text {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            display: flex; justify-content: center; align-items: center;
            padding: 10px; box-sizing: border-box;
            text-align: center; font-size: 7pt; line-height: 1.5; word-break: keep-all; 
            color: #ff0000; z-index: 2; font-family: 'Gowun Batang', serif;
        }

        .scratch-canvas {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 3; cursor: crosshair; touch-action: none;
        }
    </style>
</head>
<body>

    <video class="input_video"></video>
    <div class="loading-msg" id="cameraStatus">Initializing Camera... (Allow permissions)</div>

    <div class="overlay-text">
        <h1>RETOUCHED</h1>
        <p>
            '포토샵에 의한 죽음'. 디지털 픽셀이 발명되기 훨씬 전부터, 권력은 이미 '포토샵'을 하고 있었다. 
            스탈린의 에어브러시, 마오쩌둥의 덧칠, 그리고 누군가의 얼굴을 긁어낸 날카로운 칼날. 
            기록에서 지워진다는 것은 곧 존재한 적 없는 사람이 된다는 것. 
            이것은 단순한 이미지의 수정이 아니다. 기억에 대한 사형 선고다.
        </p>

        <div class="icon-legend">
            <div class="legend-item" onclick="window.filterCards('group4', this)">
                <img src="https://raw.githubusercontent.com/ieunseo938/image/34a3561312b38d8b3d3bd83b6f6798e2c6296cfc/Asset%2015.svg" alt="삭제" class="legend-icon">
                <span>삭제</span>
            </div>
            <div class="legend-item" onclick="window.filterCards('group3', this)">
                <img src="https://raw.githubusercontent.com/ieunseo938/image/34a3561312b38d8b3d3bd83b6f6798e2c6296cfc/Asset%2013.svg" alt="잔여" class="legend-icon">
                <span>잔여</span>
            </div>
            <div class="legend-item" onclick="window.filterCards('group2', this)">
                <img src="https://raw.githubusercontent.com/ieunseo938/image/34a3561312b38d8b3d3bd83b6f6798e2c6296cfc/Asset%2014.svg" alt="합성" class="legend-icon">
                <span>합성</span>
            </div>
            <div class="legend-item" onclick="window.filterCards('group1', this)">
                <img src="https://raw.githubusercontent.com/ieunseo938/image/34a3561312b38d8b3d3bd83b6f6798e2c6296cfc/Asset%2016.svg" alt="왜곡" class="legend-icon">
                <span>왜곡</span>
            </div>
        </div>
        
        <div class="usage-guide">
            <span class="usage-title">How to use</span>
            <ul class="usage-list">
                <li>1. 주먹을 쥐었다 펼치면 카드들이 흩어진다.</li>
                <li>2. 흩어진 카드를 클릭하여 뒤집고, 마우스 드래그로 뒷면을 지우면 설명이 나타난다.</li>
                <li>3. 다시 한 번 클릭하면 지워지거나 대체된 이미지들이 나타난다.</li>
            </ul>
        </div>
    </div>

    <div class="scene">
        <div class="world" id="world"></div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', () => {
            const individualDescriptions = [
                "1994년, 타임 매거진은 O.J 심슨의 머그샷을 표지에 실으면서 원본보다 피부색을 훨씬 검고 어둡게 만들었다. 이는 흑인에 대한 편견을 자극하고 그를 더욱 위협적인 범죄자로 보이게 하려는 의도적인 왜곡이었다.",
                "처칠의 건강한 이미지를 만들기 위해 입에 물고 있던 담배를 지워버렸다. 입 모양이 어색해지는 경우가 많다.",
                "히틀러는 안경 쓴 모습을 나약하다고 생각해서 사진에서 안경을 지우거나, 안경 쓴 사진 배포를 금지했다.",
                "1869년 링컨의 부인 메리 토드 링컨의 초상화이다. 그녀의 뒤에 암살당한 남편 에이브러햄 링컨의 유령이 두 손을 얹고 서있다. 사진가 윌리엄 멈러가 '이중 노출' 기법을 이용해 링컨의 옛날 사진을 흐릿하게 합성해 넣은 사기극이다.",
                "소련군이 베를린 의사당에 깃발을 꽂는 사진이다. 더 극적인 연출을 위해 배경에 검은 연기를 그려 넣고 구도를 수정했다.",
                "링컨의 얼굴과 노예제 옹호자 존 칼훈의 몸과 다른 배경을 합친 사진이다. 링컨의 가장 유명한 전신 사진이 사실은 가짜였다.",
                "김정일이 군부대와 함께 찍은 단체 사진이다. 행렬에서 벗어나 있는 사람들의 흔적을 지웠다.",
                "무솔리니가 영웅처럼 보이기 위해 말 고삐를 잡아주던 마부를 지웠다. 그 결과 말 고삐가 공중에 둥둥 떠있다.",
                "고트발트 옆에 있던 클레멘티스는 지워졌지만, 그가 고트발트 머리에 씌워준 털모자는 그대로 남았다.",
                "유리 가가린과 함께 찍힌 소련 우주비행사 팀 '소치 6인' 사진에서, 그리고리 낼류보프라는 비행사가 제명된 후 단체 사진에서 삭제되었다.",
                "레닌의 연설 단상 옆에 서 있었으나, 스탈린 집권 후 나무 계단으로 덮여 삭제되었다.",
                "스탈린과 걷던 예조프가 처형 후 물결로 대체되었다."
            ];

            const manipulatedImageUrls = [
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%201.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%202.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%203.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%204.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%205.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%206.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%207.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%208.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%209.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%2010.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%2011.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/20d2f43916559a7ac2369f0724fba32a2b5420e0/Asset%2012.svg" 
            ];
            
            const originalImageUrls = [
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2020.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2021.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2022.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2028.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2027.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2026.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2018.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2019.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2017.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2023.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2024.svg",
                "https://raw.githubusercontent.com/ieunseo938/image/5fc79c16eba87e4aebf01642b6c4a2c6c86852a9/Asset%2025.svg"
            ];

            const backIcons = {
                group1: "https://raw.githubusercontent.com/ieunseo938/image/34a3561312b38d8b3d3bd83b6f6798e2c6296cfc/Asset%2016.svg",
                group2: "https://raw.githubusercontent.com/ieunseo938/image/34a3561312b38d8b3d3bd83b6f6798e2c6296cfc/Asset%2014.svg",
                group3: "https://raw.githubusercontent.com/ieunseo938/image/34a3561312b38d8b3d3bd83b6f6798e2c6296cfc/Asset%2013.svg", 
                group4: "https://raw.githubusercontent.com/ieunseo938/image/34a3561312b38d8b3d3bd83b6f6798e2c6296cfc/Asset%2015.svg"
            };

            const world = document.getElementById('world');
            const cards = [];
            let isScattered = false;
            let activeGroup = null;

            function setupScratchCanvas(canvas, iconUrl, width, height) {
                const ctx = canvas.getContext('2d');
                canvas.width = width; canvas.height = height;
                ctx.fillStyle = '#f5f5f5';
                ctx.fillRect(0, 0, width, height);
                
                const iconImg = new Image();
                iconImg.crossOrigin = "Anonymous";
                iconImg.src = iconUrl;
                iconImg.onload = () => {
                    const maxIconSize = 40;
                    let drawWidth = iconImg.width;
                    let drawHeight = iconImg.height;
                    const scale = Math.min(maxIconSize / drawWidth, maxIconSize / drawHeight);
                    drawWidth *= scale; drawHeight *= scale;
                    const x = (width - drawWidth) / 2;
                    const y = (height - drawHeight) / 2;
                    ctx.drawImage(iconImg, x, y, drawWidth, drawHeight);
                };
                
                let isDrawing = false;
                const startDrawing = () => isDrawing = true;
                const stopDrawing = () => isDrawing = false;
                const scratch = (e) => {
                    if (!isDrawing) return;
                    const rect = canvas.getBoundingClientRect();
                    let clientX, clientY;
                    if (e.touches) { clientX = e.touches[0].clientX; clientY = e.touches[0].clientY; } 
                    else { clientX = e.clientX; clientY = e.clientY; }
                    const x = clientX - rect.left;
                    const y = clientY - rect.top;
                    ctx.globalCompositeOperation = 'destination-out';
                    ctx.beginPath();
                    ctx.arc(x, y, 20, 0, Math.PI * 2);
                    ctx.fill();
                };
                canvas.addEventListener('mousedown', (e) => { e.stopPropagation(); startDrawing(); });
                canvas.addEventListener('mousemove', (e) => { e.stopPropagation(); scratch(e); });
                canvas.addEventListener('mouseup', (e) => { e.stopPropagation(); stopDrawing(); });
                canvas.addEventListener('mouseleave', stopDrawing);
                canvas.addEventListener('touchstart', (e) => { e.stopPropagation(); startDrawing(); });
                canvas.addEventListener('touchmove', (e) => { e.stopPropagation(); scratch(e); });
                canvas.addEventListener('touchend', (e) => { e.stopPropagation(); stopDrawing(); });
            }

            manipulatedImageUrls.forEach((url, index) => {
                const img = new Image();
                img.src = url;
                const card = document.createElement('div');
                card.classList.add('card');
                
                let group = "";
                let backIconUrl = "";
                if (index >= 0 && index <= 2) { group = "group1"; backIconUrl = backIcons.group1; }
                else if (index >= 3 && index <= 5) { group = "group2"; backIconUrl = backIcons.group2; }
                else if (index >= 6 && index <= 8) { group = "group3"; backIconUrl = backIcons.group3; }
                else if (index >= 9 && index <= 11) { group = "group4"; backIconUrl = backIcons.group4; }
                
                card.dataset.group = group;
                card.dataset.index = index;

                const cardInner = document.createElement('div');
                cardInner.classList.add('card-inner');
                const front = document.createElement('div');
                front.classList.add('card-front');
                front.style.backgroundImage = `url('${url}')`; 
                const back = document.createElement('div');
                back.classList.add('card-back');
                
                const descContainer = document.createElement('div');
                descContainer.classList.add('description-text');
                descContainer.innerText = individualDescriptions[index];
                back.appendChild(descContainer);

                const canvas = document.createElement('canvas');
                canvas.classList.add('scratch-canvas');
                back.appendChild(canvas);

                cardInner.appendChild(front);
                cardInner.appendChild(back);
                card.appendChild(cardInner);

                img.onload = () => {
                    const aspectRatio = img.naturalWidth / img.naturalHeight;
                    let width, height;
                    if (aspectRatio > 1) { width = 170; height = width / aspectRatio; } 
                    else { height = 204; width = height * aspectRatio; }
                    card.style.width = `${width}px`;
                    card.style.height = `${height}px`;
                    setupScratchCanvas(canvas, backIconUrl, width, height);
                };

                // 초기 랜덤 겹침 (정중앙 기준)
                card.style.transform = `translate3d(0,0,0) rotate(${Math.random() * 4 - 2}deg)`;
                card.style.zIndex = manipulatedImageUrls.length - index;

                card.addEventListener('click', (e) => {
                    if (!isScattered) return;
                    if (e.target.classList.contains('scratch-canvas')) return;
                    
                    const wasFlipped = card.classList.contains('flipped');
                    const wasRevealed = card.classList.contains('revealed');
                    
                    if (!wasFlipped && !wasRevealed) {
                        card.classList.add('flipped');
                    } else if (wasFlipped && !wasRevealed) {
                        const idx = parseInt(card.dataset.index);
                        const frontElement = card.querySelector('.card-front');
                        frontElement.style.backgroundImage = `url('${originalImageUrls[idx]}')`;
                        card.classList.add('revealed');
                        card.classList.remove('flipped'); 
                    } else if (wasRevealed) {
                        const idx = parseInt(card.dataset.index);
                        const frontElement = card.querySelector('.card-front');
                        frontElement.style.backgroundImage = `url('${manipulatedImageUrls[idx]}')`;
                        card.classList.remove('revealed');
                        card.classList.add('flipped');
                    }
                });

                world.appendChild(card);
                cards.push(card);
            });

            window.filterCards = function(groupName, element) {
                if (event) event.stopPropagation();
                if (!isScattered) scatterCards();
                if (activeGroup === groupName) { resetFilter(); return; }
                activeGroup = groupName;
                document.querySelectorAll('.legend-item').forEach(item => item.classList.remove('active'));
                if(element) element.classList.add('active');
                cards.forEach(card => {
                    if (card.dataset.group === groupName) { card.classList.remove('hidden'); } 
                    else { card.classList.add('hidden'); card.classList.remove('flipped'); card.classList.remove('revealed'); }
                });
            };

            function resetFilter() {
                activeGroup = null;
                document.querySelectorAll('.legend-item').forEach(item => item.classList.remove('active'));
                cards.forEach(card => card.classList.remove('hidden'));
            }

            function scatterCards() {
                if(isScattered) return;
                isScattered = true;
                cards.forEach(card => {
                    // 화면 중앙 기준 산개 로직 (정확한 센터링)
                    const rangeX = window.innerWidth * 0.35; // 좌우 폭 70%
                    const rangeY = window.innerHeight * 0.35; // 상하 폭 70%
                    const x = (Math.random() - 0.5) * 2 * rangeX;
                    const y = (Math.random() - 0.5) * 2 * rangeY;
                    const z = (Math.random() - 0.5) * 800;
                    const rx = (Math.random() - 0.5) * 60;
                    const ry = (Math.random() - 0.5) * 60;
                    const rz = (Math.random() - 0.5) * 30;
                    card.style.transform = `translate3d(${x}px, ${y}px, ${z}px) rotateX(${rx}deg) rotateY(${ry}deg) rotateZ(${rz}deg)`;
                });
                document.querySelector('.overlay-text').style.opacity = 0.5;
            }

            function resetCards() {
                isScattered = false;
                resetFilter();
                cards.forEach(card => {
                    card.style.transform = `translate3d(0,0,0) rotate(${Math.random() * 4 - 2}deg)`;
                    card.classList.remove('flipped');
                    card.classList.remove('revealed');
                    const frontElement = card.querySelector('.card-front');
                    const idx = parseInt(card.dataset.index);
                    frontElement.style.backgroundImage = `url('${manipulatedImageUrls[idx]}')`;
                });
                document.querySelector('.overlay-text').style.opacity = 1;
            }

            document.addEventListener('click', () => {
                if (isScattered) return;
                scatterCards();
            });

            document.addEventListener('dblclick', () => {
                if (!isScattered) return;
                resetCards();
            });

            // MediaPipe 초기화 (안전 실행)
            if (typeof Hands !== 'undefined' && typeof Camera !== 'undefined') {
                const videoElement = document.querySelector('.input_video');
                const cameraStatus = document.getElementById('cameraStatus');

                function onResults(results) {
                    cameraStatus.innerText = "Camera Active";
                    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                        const landmarks = results.multiHandLandmarks[0];
                        const wrist = landmarks[0];
                        const middleTip = landmarks[12];
                        const distance = Math.sqrt(Math.pow(wrist.x - middleTip.x, 2) + Math.pow(wrist.y - middleTip.y, 2));
                        const threshold = 0.25;
                        if (distance > threshold) {
                            if (!isScattered) scatterCards();
                        } else {
                            if (isScattered) resetCards();
                        }
                    }
                }

                const hands = new Hands({locateFile: (file) => {
                    return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
                }});

                hands.setOptions({
                    maxNumHands: 1,
                    modelComplexity: 1,
                    minDetectionConfidence: 0.5,
                    minTrackingConfidence: 0.5
                });

                hands.onResults(onResults);

                const camera = new Camera(videoElement, {
                    onFrame: async () => {
                        await hands.send({image: videoElement});
                    },
                    width: 640,
                    height: 480
                });
                
                camera.start().catch(err => {
                    console.error("Camera Error:", err);
                    cameraStatus.innerText = "Camera Error";
                });
            }
        });
    </script>
</body>
</html>
