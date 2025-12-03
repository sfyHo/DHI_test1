# 你的開心牧場
html_code = """<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>DHI 飼養管理小遊戲 v2</title>
  <style>
    body { font-family: Arial, "Noto Sans TC", sans-serif; padding: 20px; background:#fffef6; color:#222; }
    button { padding: 8px 12px; margin:6px; cursor:pointer; }
    #game, #studentSetup { display:none; max-width: 760px; }
    #rankBoard { margin-top:20px; background:#f7f7f9; padding:12px; border-radius:8px; }
    #working { text-align:center; margin:18px 0; }
    #working img { width:160px; max-width:40%; }
    table.leader { width:100%; border-collapse:collapse; margin-top:8px; }
    table.leader th, table.leader td { border:1px solid #ddd; padding:8px; text-align:left; }
    .meta { margin-top:8px; font-size:0.95rem; color:#555; }
    #investPanel { margin-top:10px; }
  </style>
</head>
<body>

<!-- 身份選擇 -->
<div id="identitySelect">
  <h2>請選擇身份開始遊戲</h2>
  <button onclick="chooseIdentity('student')">我是學生</button>
  <button onclick="chooseIdentity('tester')">我是系統測試員</button>
</div>

<!-- 學生設定畫面 -->
<div id="studentSetup">
  <h2>學生設定</h2>
  <label>選擇組別：</label>
  <select id="groupSelect">
    <option value="G1">G1</option><option value="G2">G2</option><option value="G3">G3</option>
    <option value="G4">G4</option><option value="G5">G5</option><option value="G6">G6</option>
    <option value="G7">G7</option><option value="G8">G8</option><option value="G9">G9</option>
    <option value="G10">G10</option>
  </select>
  <br><br>
  <label>輸入姓名（最多10字元）：</label>
  <input type="text" id="studentName" maxlength="30" />
  <button onclick="confirmStudent()">確認</button>
  <p id="nameError" style="color:red;"></p>
</div>

<!-- 遊戲主體 -->
<div id="game">
  <h2>🐄 DHI 飼養管理小遊戲 v2</h2>
  <div class="meta">玩家：<span id="playerMeta">-</span></div>
  <p id="scenario" style="font-weight:600;"></p>
  <div id="options"></div>
  <div id="working"></div>
  <p id="result"></p>
  <p>總收益：<span id="score">0</span> 萬 NTD（單位）</p>
  <div id="investPanel">
    <p>飼養頭數：<span id="herdSize">30</span> 頭</p>
    <button onclick="investExpand()">投資：增加10頭（花費 50 萬 NTD）</button>
  </div>
  <div id="rankBoard"></div>
  <div id="wrongAnswersDiv"></div>
</div>

<audio id="bgm" loop>
  <source src="https://www.bensound.com/bensound-music/bensound-sunny.mp3" type="audio/mpeg">
</audio>

<script>
/* ------------------------------
   範例題庫（可後續擴充）
------------------------------ */
const scenarios = [
  {
    description: "乳量下降 10%，體細胞上升至 380k。",
    options: [
      { text: "改善牛床乾燥度與墊料", baseEffect: 8, msg: "體細胞下降，乳量回升！", correct:true, reason:"改善環境可降低乳房炎風險" },
      { text: "濃料比例提高 5%", baseEffect: -3, msg: "乳量未改善，反而有亞臨床乳房炎風險。", correct:false, reason:"濃料過多可能造成健康問題" },
      { text: "增加擠乳頻率到每日 3 次", baseEffect: 4, msg: "乳量小幅上升。", correct:false, reason:"可提升乳量，但未根本改善乳房健康" }
    ]
  },
  {
    description: "泌乳初期（30 DIM）乳脂率僅 2.8%，疑似負能量平衡。",
    options: [
      { text: "提高乾物攝取量、改善日糧適口性", baseEffect: 7, msg: "DMI 上升，乳脂正常化！", correct:true, reason:"增加能量攝取改善乳脂率" },
      { text: "減少飼料量以避免乳脂過高", baseEffect: -4, msg: "問題更嚴重，能量不足！", correct:false, reason:"減少飼料會惡化負能量平衡" }
    ]
  }
];

let current = 0;
let score = 0;           // 單位：萬 NTD
let identity = "";
let studentName = "";
let studentGroup = "";
let wrongAnswers = [];
let herdSize = 30;       // 初始頭數
let savedScoreFlag = false; // 避免重複存檔

/* ------------------------------
   日期檢查（台灣時間）
------------------------------ */
function getTodayTW() {
  const now = new Date();
  const utc = now.getTime() + (now.getTimezoneOffset() * 60000);
  const tw = new Date(utc + 8 * 60 * 60000);
  return tw.toISOString().slice(0,10);
}
function checkDailyReset() {
  const today = getTodayTW();
  const last = localStorage.getItem("score_last_reset");
  if (last !== today) {
    localStorage.setItem("score_last_reset", today);
    localStorage.setItem("student_scores", JSON.stringify([]));
  }
}

/* ------------------------------
   身份與學生設定
------------------------------ */
function chooseIdentity(type) {
  identity = type;
  if(identity === "student") {
    document.getElementById("identitySelect").style.display = "none";
    document.getElementById("studentSetup").style.display = "block";
  } else {
    document.getElementById("identitySelect").style.display = "none";
    startGame();
  }
}

function countGraphemeLength(str) {
  // 使用展開的方式計算實際顯示字元（包含 emoji）
  return [...str].length;
}

function confirmStudent() {
  const nameInput = document.getElementById("studentName");
  const name = nameInput.value.trim();
  if(!name || countGraphemeLength(name) > 10) {
    document.getElementById("nameError").innerText = "字元數超過10個，請縮短並重新輸入名稱";
    return;
  }
  studentName = name;
  studentGroup = document.getElementById("groupSelect").value;
  document.getElementById("studentSetup").style.display = "none";
  startGame();
}

/* ------------------------------
   遊戲開始與題目載入
------------------------------ */
function startGame() {
  checkDailyReset();
  document.getElementById("game").style.display = "block";
  document.getElementById("playerMeta").innerText = (identity==="student") ? (studentGroup + " / " + studentName) : "系統測試員";
  document.getElementById("herdSize").innerText = herdSize;
  savedScoreFlag = false;
  document.getElementById("bgm").play().catch(()=>{});
  loadQuestion();
}

function loadQuestion() {
  const s = scenarios[current];
  document.getElementById("scenario").innerText = `情境：${s.description}`;
  const optionsDiv = document.getElementById("options");
  optionsDiv.innerHTML = "";
  document.getElementById("result").innerText = "";
  document.getElementById("working").innerHTML = "";

  s.options.forEach((opt, idx) => {
    const btn = document.createElement("button");
    btn.textContent = opt.text;
    btn.onclick = () => chooseWithEffect(idx);
    optionsDiv.appendChild(btn);
  });
}

/* ------------------------------
   投資擴張功能（增加 herdSize）
------------------------------ */
function investExpand() {
  const cost = 50; // 50 萬 NTD
  if(score < cost) {
    alert("目前收益不足，無法投資擴充。");
    return;
  }
  score -= cost;
  herdSize += 10;
  document.getElementById("score").innerText = score;
  document.getElementById("herdSize").innerText = herdSize;
}

/* ------------------------------
   點選答案後的特效（努力泌乳中...）
------------------------------ */
function chooseWithEffect(idx) {
  const workingDiv = document.getElementById("working");
  workingDiv.innerHTML = '<p style="font-weight:700;">努力泌乳中...</p><img src="https://i.imgur.com/EF4s7I6.png" alt="卡通乳牛" />';
  document.getElementById("options").querySelectorAll("button").forEach(b=>b.disabled=true);

  setTimeout(() => {
    workingDiv.innerHTML = "";
    choose(idx);
  },3000);
}

/* ------------------------------
   核心計算：套用收益遞減與 herdSize 放大效果
   baseEffect 單位：萬 NTD（可為負）
------------------------------ */
function calculateEffectiveGain(baseEffect) {
  // (A) 收益越高，提升難度：100/(100+|score|)
  const diminishing = 100 / (100 + Math.abs(score) + 0.0001); // 加小量避免除以零
  // (B) herdSize 放大（相對於 30 頭為基準）
  const scale = herdSize / 30;
  // 合併
  const eff = baseEffect * diminishing * scale;
  // 取整數（萬 NTD）
  return Math.round(eff);
}

/* ------------------------------
   選項處理、錯題記錄
------------------------------ */
function choose(idx) {
  const s = scenarios[current];
  const opt = s.options[idx];

  const eff = calculateEffectiveGain(opt.baseEffect);
  score += eff;
  document.getElementById("score").innerText = score;
  document.getElementById("result").innerText =
    `結果：${opt.msg}（收益 ${eff >= 0 ? "+" : ""}${eff} 萬 NTD）`;

  // 錯題紀錄（僅學生） ；建議答案取第一個 correct=true 的選項
  if(identity==="student") {
    const correctOpt = s.options.find(o=>o.correct);
    if(!opt.correct) {
      wrongAnswers.push({
        question: s.description,
        wrongChoice: opt.text,
        correctChoice: correctOpt ? correctOpt.text : '(無)',
        reason: correctOpt ? correctOpt.reason : ''
      });
    }
  }

  // 允許自動進下一題（1 秒後），或若為最後題則結束
  setTimeout(() => {
    current++;
    if(current >= scenarios.length) {
      endGame();
    } else {
      loadQuestion();
    }
  }, 1000);
}

/* ------------------------------
   遊戲結束、排行榜與錯題檢討
------------------------------ */
function endGame() {
  document.getElementById("scenario").innerText = "🎉 遊戲結束！";
  document.getElementById("options").innerHTML = "";
  document.getElementById("working").innerHTML = "";
  document.getElementById("result").innerText = "";

  if(identity==="student" && !savedScoreFlag) {
    saveStudentScore({ group: studentGroup, name: studentName, score: score });
    savedScoreFlag = true;
  }
  if(identity==="student") {
    showRankBoard();
    showWrongAnswers();
  } else {
    // 測試員結束不記錄
    document.getElementById("rankBoard").innerHTML = "<p>系統測試員遊玩，不列入排行榜。</p>";
  }

  // 隱藏 investPanel（結束後不再操作）
  document.getElementById("investPanel").style.display = "none";
}

/* ------------------------------
   儲存排行榜（localStorage 儲存物件陣列）
   格式：[{group,name,score},...]
------------------------------ */
function saveStudentScore(obj) {
  const key = "student_scores";
  let arr = JSON.parse(localStorage.getItem(key) || "[]");
  arr.push(obj);
  // 以 score 排序（大到小）
  arr.sort((a,b)=>b.score - a.score);
  localStorage.setItem(key, JSON.stringify(arr));
}

/* ------------------------------
   顯示排行榜（含組別、名稱、最終收益，單位：萬 NTD）
------------------------------ */
function showRankBoard() {
  const arr = JSON.parse(localStorage.getItem("student_scores") || "[]");
  if(arr.length === 0) {
    document.getElementById("rankBoard").innerHTML = "<p>今日尚無紀錄。</p>";
    return;
  }
  let html = '<h3>📊 今日績優酪農戶排行榜</h3>';
  html += '<table class="leader"><thead><tr><th>名次</th><th>組別</th><th>名稱</th><th>最終收益</th></tr></thead><tbody>';
  arr.forEach((it, idx) => {
    html += `<tr><td>${idx+1}</td><td>${it.group}</td><td>${it.name}</td><td>${it.score} 萬 NTD</td></tr>`;
  });
  html += '</tbody></table>';
  document.getElementById("rankBoard").innerHTML = html;
}

/* ------------------------------
   顯示錯題檢討表
------------------------------ */
function showWrongAnswers() {
  if(wrongAnswers.length === 0) {
    document.getElementById("wrongAnswersDiv").innerHTML = "<p>恭喜！沒有錯題。</p>";
    return;
  }
  let html = "<h3>❌ 錯題檢討表</h3><ol>";
  wrongAnswers.forEach(w=>{
    html += `<li><strong>題目：</strong>${w.question}<br><strong>你選擇：</strong>${w.wrongChoice}<br><strong>建議答案：</strong>${w.correctChoice}<br><strong>原因：</strong>${w.reason}</li><br>`;
  });
  html += "</ol>";
  document.getElementById("wrongAnswersDiv").innerHTML = html;
}

/* ------------------------------
   初始化（避免未開始就按投資）
------------------------------ */
document.getElementById("investPanel").style.display = "none";
// 當遊戲開始時才顯示投資面板
(function observeGameStart(){
  const target = document.getElementById('game');
  const observer = new MutationObserver(()=>{
    if(target.style.display === 'block') {
      document.getElementById("investPanel").style.display = "block";
    }
  });
  observer.observe(target, { attributes: true, attributeFilter: ['style'] });
})();

</script>
</body>
</html>
