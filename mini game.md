<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>보안 미니게임: 악성코드 디버깅</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Nanum+Myeongjo:wght@700;800&family=Noto+Sans+KR:wght@400;700&display=swap" rel="stylesheet">

  <style>
    :root {
      --font-title: 'Nanum Myeongjo', serif;
      --font-body: 'Noto Sans KR', sans-serif;
      --bg-wood-overlay: rgba(36, 23, 16, 0.75);
      --paper-card-bg: #FDF9F0;
      --paper-card-sub: #F3EBDD;
      --text-dark: #2B1A10;
      --border-gold: #A67C52;
      --accent-red: #8B261D;
      --accent-green: #2E7D32;
      --cell-bg: #E6D8C3;
      --cell-revealed: #FFFDF9;
    }

    * { box-sizing: border-box; margin: 0; padding: 0; }

    body {
      background: linear-gradient(var(--bg-wood-overlay), var(--bg-wood-overlay)),
                  url('https://images.unsplash.com/photo-1546484475-7f7bd55792da?auto=format&fit=crop&w=1920&q=80') center/cover fixed no-repeat #2B1A10;
      color: var(--text-dark);
      font-family: var(--font-body);
      line-height: 1.5;
      padding: 20px 10px;
      min-height: 100vh;
    }

    .game-container {
      max-width: 600px;
      margin: 0 auto;
      background-color: var(--paper-card-bg);
      border: 6px double var(--border-gold);
      border-radius: 8px;
      box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
      padding: 25px;
      text-align: center;
    }

    h1 {
      font-family: var(--font-title);
      font-size: 1.8rem;
      color: var(--accent-red);
      margin-bottom: 6px;
    }

    .rule-box {
      background-color: var(--paper-card-sub);
      border: 1px solid var(--border-gold);
      padding: 10px 15px;
      border-radius: 4px;
      font-size: 0.95rem;
      margin-bottom: 15px;
      text-align: left;
    }

    /* 게임 상태 및 대시보드 */
    .dashboard {
      display: flex;
      justify-content: space-between;
      align-items: center;
      background-color: #FFFDF9;
      border: 1px solid var(--border-gold);
      padding: 10px 15px;
      border-radius: 6px;
      margin-bottom: 15px;
      font-weight: bold;
    }

    .status-text { font-size: 1.1rem; color: var(--accent-red); }

    /* 컨트롤 버튼 그룹 */
    .controls {
      display: flex;
      gap: 10px;
      justify-content: center;
      margin-bottom: 15px;
      flex-wrap: wrap;
    }

    button {
      background-color: var(--paper-card-sub);
      color: var(--text-dark);
      border: 1px solid var(--border-gold);
      padding: 8px 16px;
      font-weight: bold;
      border-radius: 4px;
      cursor: pointer;
      font-family: var(--font-body);
    }

    button:hover { background-color: #E8DCC8; }
    button:focus-visible { outline: 3px solid var(--accent-red); }

    /* 접근성 옵션 */
    .accessibility-panel {
      font-size: 0.85rem;
      margin-bottom: 15px;
      display: flex;
      justify-content: center;
      gap: 15px;
    }

    /* 5x5 지뢰찾기 그리드 */
    .grid {
      display: grid;
      grid-template-columns: repeat(5, 1fr);
      gap: 6px;
      max-width: 350px;
      margin: 0 auto 15px auto;
    }

    .cell {
      width: 100%;
      aspect-ratio: 1;
      background-color: var(--cell-bg);
      border: 2px solid var(--border-gold);
      border-radius: 4px;
      font-size: 1.3rem;
      font-weight: bold;
      display: flex;
      justify-content: center;
      align-items: center;
      cursor: pointer;
      user-select: none;
      transition: transform 0.1s ease;
    }

    .cell:hover { background-color: #DCCBB0; }
    .cell.revealed {
      background-color: var(--cell-revealed);
      cursor: default;
      border-color: #C0A88B;
    }
    .cell.virus { background-color: #FFCDD2; }

    /* 최고 기록 섹션 */
    .high-score {
      font-size: 0.9rem;
      color: var(--border-gold);
      border-top: 1px dashed var(--border-gold);
      padding-top: 10px;
    }

    @media (prefers-reduced-motion: reduce) {
      .cell { transition: none !important; }
    }
  </style>
</head>
<body>

  <div class="game-container">
    <h1>👾 악성코드 디버깅</h1>
    
    <!-- T02-C03: 규칙 / T02-C04: 조율 / T02-C05: 현재 상태 -->
    <div class="rule-box">
      <p><strong>📜 게임 규칙:</strong> 30초 안에 바이러스(👾)가 없는 <strong>정상 영역 20곳</strong>을 모두 정화하세요!</p>
      <p><strong>🎮 조작법:</strong> 타일을 좌클릭하여 검사하거나, 바이러스로 의심되는 곳은 우클릭(또는 긴 누르기)으로 바이러스 깃발(👾)을 표시할 수 있습니다.</p>
    </div>

    <div class="accessibility-panel">
      <label><input type="checkbox" id="toggle-motion"> 움직임 줄이기</label>
      <label><input type="checkbox" id="toggle-sound"> 효과음 끄기</label>
    </div>

    <div class="dashboard">
      <div>⏱️ 남은 시간: <span id="timer">30</span>초</div>
      <div id="game-status" class="status-text">준비 중</div>
      <div>🛡️ 정화: <span id="cleared-count">0</span> / 20</div>
    </div>

    <div class="controls">
      <button id="btn-start">게임 시작 / 재시작</button>
      <button id="btn-pause">일시정지</button>
      <button id="btn-reset-data">기록 초기화</button>
    </div>

    <!-- 5x5 지뢰찾기 격자 -->
    <div class="grid" id="grid" role="grid" aria-label="악성코드 정화 격자"></div>

    <div class="high-score">
      🏆 최고 성공 기록: <span id="best-score">기록 없음</span>
    </div>
  </div>

  <script>
    // 게임 설정 및 상태 관리
    const BOARD_SIZE = 5;
    const VIRUS_COUNT = 5;
    const SAFE_COUNT = (BOARD_SIZE * BOARD_SIZE) - VIRUS_COUNT; // 20
    const TIME_LIMIT = 30;

    let board = [];
    let revealedCount = 0;
    let timeLeft = TIME_LIMIT;
    let timerId = null;
    let gameState = 'READY'; // READY, PLAYING, PAUSED, WIN, GAMEOVER

    // DOM 요소
    const gridEl = document.getElementById('grid');
    const timerEl = document.getElementById('timer');
    const statusEl = document.getElementById('game-status');
    const clearedEl = document.getElementById('cleared-count');
    const bestScoreEl = document.getElementById('best-score');
    
    const btnStart = document.getElementById('btn-start');
    const btnPause = document.getElementById('btn-pause');
    const btnResetData = document.getElementById('btn-reset-data');

    // T02-C24 & T02-C25: localStorage 데이터 보존 및 손상 복구 로직
    function loadHighScore() {
      try {
        const saved = localStorage.getItem('security_game_best');
        if (!saved) return '기록 없음';
        const parsed = JSON.parse(saved);
        if (typeof parsed.time === 'number') {
          return `${parsed.time}초 남기고 성공`;
        }
        throw new Error('손상된 데이터');
      } catch (e) {
        // 손상되거나 빈 값인 경우 안전한 기본값으로 복구
        localStorage.removeItem('security_game_best');
        return '기록 없음';
      }
    }

    function saveHighScore(time) {
      try {
        const currentBest = localStorage.getItem('security_game_best');
        let currentBestTime = -1;
        if (currentBest) {
          currentBestTime = JSON.parse(currentBest).time || -1;
        }
        if (time > currentBestTime) {
          localStorage.setItem('security_game_best', JSON.stringify({ time: time }));
          bestScoreEl.textContent = `${time}초 남기고 성공`;
        }
      } catch (e) {
        console.warn('저장 실패:', e);
      }
    }

    // 보드 초기화 및 30초 핵심 루프 설정
    function initBoard() {
      gridEl.innerHTML = '';
      board = [];
      revealedCount = 0;
      clearedEl.textContent = revealedCount;

      // 5x5 빈 격자 생성
      for (let r = 0; r < BOARD_SIZE; r++) {
        board[r] = [];
        for (let c = 0; c < BOARD_SIZE; c++) {
          board[r][c] = { r, c, isVirus: false, revealed: false, flagged: false, count: 0 };
        }
      }

      // 무작위 5개 바이러스(지뢰) 배치
      let placed = 0;
      while (placed < VIRUS_COUNT) {
        let r = Math.floor(Math.random() * BOARD_SIZE);
        let c = Math.floor(Math.random() * BOARD_SIZE);
        if (!board[r][c].isVirus) {
          board[r][c].isVirus = true;
          placed++;
        }
      }

      // 주변 바이러스 수 계산
      for (let r = 0; r < BOARD_SIZE; r++) {
        for (let c = 0; c < BOARD_SIZE; c++) {
          if (!board[r][c].isVirus) {
            let count = 0;
            for (let dr = -1; dr <= 1; dr++) {
              for (let dc = -1; dc <= 1; dc++) {
                let nr = r + dr, nc = c + dc;
                if (nr >= 0 && nr < BOARD_SIZE && nc >= 0 && nc < BOARD_SIZE && board[nr][nc].isVirus) {
                  count++;
                }
              }
            }
            board[r][c].count = count;
          }
        }
      }

      // DOM 요소 그리기
      for (let r = 0; r < BOARD_SIZE; r++) {
        for (let c = 0; c < BOARD_SIZE; c++) {
          const cellEl = document.createElement('div');
          cellEl.className = 'cell';
          cellEl.dataset.r = r;
          cellEl.dataset.c = c;
          
          cellEl.addEventListener('click', () => handleCellClick(r, c));
          cellEl.addEventListener('contextmenu', (e) => {
            e.preventDefault();
            handleCellRightClick(r, c);
          });
          gridEl.appendChild(cellEl);
        }
      }
    }

    function startGame() {
      clearInterval(timerId);
      initBoard();
      timeLeft = TIME_LIMIT;
      timerEl.textContent = timeLeft;
      gameState = 'PLAYING';
      statusEl.textContent = '진행 중';
      statusEl.style.color = '#8B261D';

      timerId = setInterval(() => {
        if (gameState === 'PLAYING') {
          timeLeft--;
          timerEl.textContent = timeLeft;
          if (timeLeft <= 0) {
            endGame(false, '시간 초과!');
          }
        }
      }, 1000);
    }

    function handleCellClick(r, c) {
      if (gameState !== 'PLAYING') return;
      const cell = board[r][c];
      if (cell.revealed || cell.flagged) return;

      cell.revealed = true;
      const cellEl = gridEl.children[r * BOARD_SIZE + c];
      cellEl.classList.add('revealed');

      if (cell.isVirus) {
        cellEl.classList.add('virus');
        cellEl.textContent = '👾'; // 바이러스 아이콘 표시
        endGame(false, '악성코드 감염!');
      } else {
        revealedCount++;
        clearedEl.textContent = revealedCount;
        cellEl.textContent = cell.count > 0 ? cell.count : '';
        
        // 20개 안전 구역 모두 정화 시 승리
        if (revealedCount === SAFE_COUNT) {
          endGame(true, '시스템 정화 완료!');
        }
      }
    }

    function handleCellRightClick(r, c) {
      if (gameState !== 'PLAYING') return;
      const cell = board[r][c];
      if (cell.revealed) return;

      cell.flagged = !cell.flagged;
      const cellEl = gridEl.children[r * BOARD_SIZE + c];
      cellEl.textContent = cell.flagged ? '👾' : '';
    }

    function endGame(isWin, message) {
      gameState = isWin ? 'WIN' : 'GAMEOVER';
      clearInterval(timerId);
      statusEl.textContent = message;
      statusEl.style.color = isWin ? '#2E7D32' : '#8B261D';

      if (isWin) {
        saveHighScore(timeLeft);
      } else {
        // 바이러스 위치 모두 공개
        for (let r = 0; r < BOARD_SIZE; r++) {
          for (let c = 0; c < BOARD_SIZE; c++) {
            if (board[r][c].isVirus) {
              const cellEl = gridEl.children[r * BOARD_SIZE + c];
              cellEl.classList.add('virus');
              cellEl.textContent = '👾';
            }
          }
        }
      }
    }

    // 일시정지 로직
    btnPause.addEventListener('click', () => {
      if (gameState === 'PLAYING') {
        gameState = 'PAUSED';
        statusEl.textContent = '일시정지됨';
      } else if (gameState === 'PAUSED') {
        gameState = 'PLAYING';
        statusEl.textContent = '진행 중';
      }
    });

    // T02-C14: 포커스 이탈 시 일시정지 처리
    window.addEventListener('blur', () => {
      if (gameState === 'PLAYING') {
        gameState = 'PAUSED';
        statusEl.textContent = '일시정지됨 (화면 이탈)';
      }
    });

    btnStart.addEventListener('click', startGame);
    btnResetData.addEventListener('click', () => {
      localStorage.removeItem('security_game_best');
      bestScoreEl.textContent = loadHighScore();
    });

    // 초기 실행
    bestScoreEl.textContent = loadHighScore();
    initBoard();
  </script>
</body>
</html>