# 你的開心牧場

<html>
<head>
  <meta charset="UTF-8" />
  <title>DHI 飼養管理小遊戲</title>
  <style>
    body {
      font-family: Arial;
      padding: 20px;
      background: #fffaf0;
    }
    button {
      padding: 10px 15px;
      margin: 5px;
      border: none;
      background: #4CAF50;
      color: white;
      border-radius: 6px;
      cursor: pointer;
    }
    button:hover {
      background: #45a049;
    }
    #game { max-width: 700px; display:none; }
    #identitySelect { margin-bottom:20px; }
    .hidden { display:none; }
    #loadingBox {
      display: none;
      text-align: center;
      margin: 20px 0;
      font-size: 20px;
    }
    #loadingBox img {
      width: 180px;
      display: block;
      margin: auto;
    }
    #rankBoard { 
      margin-top:20px; 
      background:#f0f0f0; 
      padding:15px; 
      border-radius:8px; 
    }
    #summaryBox{
      background:#eef7ff;
      padding:15px;
      border-radius:10px;
      margin-top:20px;
    }
  </style>
</head>
<body>

<!-- 背景音樂 -->
<audio id="bgm" loop autoplay>
  <source src="https://www.bensound.com/bensound-music/bensound-sunny.mp3" type="audio/mpeg">
</audio>

<!-- 身份選擇 -->
<div id="identitySelect">
  <h2>請選擇身份開始遊戲</h2>
  <button onclick="chooseIdentity('student')">我是學生</button>
  <button onclick="chooseIdentity('tester')">我是系統測試員</button>
</div>

<!-- 學生輸入組別 + 名字 -->
<div id="studentInfo" class="hidden">
  <h3>學生資訊</h3>
  <label>請選擇組別：</label>
  <select id="groupSelect">
    <option value="">請選擇</option>
    <option value="G1">G1</option>
    <option value="G2">G2</option>
    <option value="G3">G3</option>
    <option value="G4">G4</option>
    <option value="G5">G5</option>
    <option value="G6">G6</option>
    <option value="G7">G7</option>
    <option value="G8">G8</option>
    <option value="G9">G9</option>
    <option value="G10">G10</option>
  </select>

  <br><br>

  <label>請輸入名稱（10 字元以內）：</label>
  <input type="text" id="studentName" maxlength="20">

  <br><br>

  <button onclick="confirmStudent()">開始遊戲</button>
  <p id="nameError" style="color:red;"></p>
</div>

<!-- 遊戲主體 -->
<div id="game">
  <h2>🐄 DHI 飼養管理小遊戲</h2>
  <p id="scenario"></p>

  <div id="options"></div>
  <p id="result"> </p> 
  <p>目前收益：<span id="score">0</span> 萬 NTD</p>

  <p>目前飼養頭數：<span id="herdSize">0</span> 頭</p>
  
  <div id="loadingBox">
    <p>努力泌乳中...</p>
  </div>

  <button id="investBtn" onclick="invest()" class="hidden">投資（花費 40 萬，增加 10 頭牛）</button>

  <div id="rankBoard"></div>
  <div id="summaryBox" class="hidden"></div>
</div>

<script>
 // ----- 遊戲狀態變數 -----
let current = 0;
let score = 0;           // 單位：萬 NTD
let identity = "";
let studentName = "";
let studentGroup = "";
let wrongAnswers = [];
let herdSize = 30;       // 初始頭數
let savedScoreFlag = false;
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
   身份與學生資訊處理
------------------------------ */
function chooseIdentity(type) {
  identity = type;
  document.getElementById("identitySelect").style.display = "none";
  if (type === 'student') {
    document.getElementById("studentInfo").classList.remove("hidden");
  } else {
    startGame();
  }
}

function countGraphemeLength(str) {
  // 計算包含 emoji 的真實字元數
  return [...str].length;
}

function confirmStudent() {
  const name = document.getElementById("studentName").value.trim();
  if (!name || countGraphemeLength(name) > 10) {
    document.getElementById("nameError").innerText = "字元數超過10個，請縮短並重新輸入名稱";
    return;
  }
  studentName = name;
  studentGroup = document.getElementById("groupSelect").value || "未分組";
  document.getElementById("studentInfo").classList.add("hidden");
  startGame();
}

/* ------------------------------
   載入題目與顯示
------------------------------ */
function startGame() {
  checkDailyReset();
  document.getElementById("game").style.display = "block";
  document.getElementById("score").innerText = score;
  document.getElementById("herdSize") && (document.getElementById("herdSize").innerText = herdSize);
  document.getElementById("playerMeta") && (document.getElementById("playerMeta").innerText = (identity==="student") ? (studentGroup+" / "+studentName) : "系統測試員");
  savedScoreFlag = false;
  try { document.getElementById("bgm").play(); } catch(e){}
  loadQuestion();
}

function loadQuestion() {
  const s = scenarios[current];
  document.getElementById("scenario").innerText = "情境：" + s.description;
  const optionsDiv = document.getElementById("options");
  optionsDiv.innerHTML = "";
  document.getElementById("result").innerText = "";
  // 顯示按鈕
  s.options.forEach((opt, idx) => {
    const btn = document.createElement("button");
    btn.textContent = opt.text;
    btn.onclick = () => chooseWithEffect(idx);
    optionsDiv.appendChild(btn);
  });
}

/* ------------------------------
   點選答案後的特效（努力泌乳中...）並呼叫 choose()
------------------------------ */
function chooseWithEffect(idx) {
  // 隱藏選項，顯示努力泌乳中
  document.getElementById("options").style.display = "none";
  document.getElementById("loadingBox").style.display = "block";
  // 禁用按鈕（若還存在）
  document.querySelectorAll("#options button").forEach(b=>b.disabled=true);

  setTimeout(() => {
    document.getElementById("loadingBox").style.display = "none";
    document.getElementById("options").style.display = "block";
    choose(idx);
  }, 2000);
}
</script>

<script>
/* ------------------------------
   計分模型：收益遞減 + 規模放大
   baseEffect 單位：萬 NTD
------------------------------ */
function calculateEffectiveGain(baseEffect) {
  // (A) 收益越高 → 遞減收益係數（避免後期暴衝）
  const diminishing = 100 / (100 + Math.abs(score) + 0.0001)
  // (B) 飼養頭數對應收益
  const scale = Math.pow(herdSize / 60, 0.75); 
  // 先套用 3 倍倍率與 herd scaling
  let eff = baseEffect * 3 * scale;
  // 正負分使用不同 diminishing
  if (eff > 0) {
    // 答對 → 用原本 diminishing（增益後期下降）
    eff *= diminishing;
  } else {
    // 答錯 → 用弱化版 diminishing（扣分變大但不失控）
    const softDim = Math.sqrt(diminishing); 
    eff *= softDim;
  }
  return Math.round(eff);
}

/* ------------------------------
   投資擴張（與 UI 綁定）
------------------------------ */
function invest() {
  const cost = 40; // 40 萬 NTD
  if (score < cost) {
    alert("目前收益不足，無法投資擴充。");
    return;
  }
  score -= cost;
  herdSize += 10;
  updateUIAfterChange();
}

/* ------------------------------
   更新畫面顯示（收益、頭數）
------------------------------ */
function updateUIAfterChange() {
  document.getElementById("score").innerText = score;
  const herdEl = document.getElementById("herdSize");
  if (herdEl) herdEl.innerText = herdSize;
}

/* ------------------------------
   選項選擇處理：計算、記錄錯題、顯示結果
------------------------------ */
function choose(idx) {
  const s = scenarios[current];
  const opt = s.options[idx];

  // 計算實際影響（萬 NTD）
  const eff = calculateEffectiveGain(opt.baseEffect || 0);

  // 更新 score
  score += eff;
  updateUIAfterChange();

  // 顯示結果訊息（顯示為 萬 NTD）
  document.getElementById("result").innerText =
    `結果：${opt.msg}（收益 ${eff >= 0 ? "+" : ""}${eff} 萬 NTD）`;

  // 如果是學生且選錯，記錄錯題
  if (identity === "student") {
    const correctOpt = s.options.find(o => o.correct);
    if (!opt.correct) {
      wrongAnswers.push({
        question: s.description,
        wrongChoice: opt.text,
        correctChoice: correctOpt ? correctOpt.text : "(無)",
        reason: correctOpt ? correctOpt.reason : ""
      });
    }
  }

  // 小延遲後自動下一題或結束（3.5 秒）
  setTimeout(() => {
    current++;
    if (current >= scenarios.length) {
      endGame();
    } else {
      loadQuestion();
    }
  }, 3500);
}

/* ------------------------------
   遊戲結束：儲存成績、顯示排行榜與錯題檢討
------------------------------ */
function endGame() {
  // 隱藏題目與操作，顯示結算
  document.getElementById("scenario").innerText = "🎉 遊戲結束！";
  document.getElementById("options").innerHTML = "";
  document.getElementById("loadingBox").style.display = "none";
  document.getElementById("result").innerText = "";

  // 隱藏按鈕（避免再操作）
  document.getElementById("nextBtn") && (document.getElementById("nextBtn").style.display = "none");
  document.getElementById("investBtn") && (document.getElementById("investBtn").style.display = "none");

  // 學生儲存成績（避免重複存檔）
  if (identity === "student" && !savedScoreFlag) {
    saveStudentScore({ group: studentGroup || "未分組", name: studentName || "匿名", score: score });
    savedScoreFlag = true;
  }

  // 顯示排行榜與錯題檢討（學生）
  if (identity === "student") {
    showRankBoard();
    showWrongAnswers();
    // 顯示 summaryBox（結算摘要）
    const sb = document.getElementById("summaryBox");
    sb.classList.remove("hidden");
    sb.innerHTML = `<h3>結算摘要</h3>
      <p>玩家：${studentGroup || '未分組'} / ${studentName || '匿名'}</p>
      <p>最終收益：${score} 萬 NTD</p>
      <p>飼養頭數：${herdSize} 頭</p>
      <p>錯題數：${wrongAnswers.length}</p>`;
  } else {
    document.getElementById("rankBoard").innerHTML = "<p>系統測試員遊玩，不列入排行榜。</p>";
  }
}

/* ------------------------------
   排行榜儲存（localStorage）：物件陣列格式
   [{group,name,score},...]
   每日會由 checkDailyReset() 清空
------------------------------ */
function saveStudentScore(obj) {
  const key = "student_scores";
  const arr = JSON.parse(localStorage.getItem(key) || "[]");
  arr.push(obj);
  // 依 score 排序（大到小）
  arr.sort((a, b) => b.score - a.score);
  localStorage.setItem(key, JSON.stringify(arr));
}

/* ------------------------------
   顯示排行榜（含組別、名稱、最終收益）
------------------------------ */
function showRankBoard() {
  const arr = JSON.parse(localStorage.getItem("student_scores") || "[]");
  if (!arr || arr.length === 0) {
    document.getElementById("rankBoard").innerHTML = "<p>今日尚無紀錄。</p>";
    return;
  }

  let html = `<h3>📊 今日績優酪農戶排行榜</h3>
    <table style="width:100%;border-collapse:collapse;">
      <thead>
        <tr style="background:#f2f2f2;"><th style="padding:6px;border:1px solid #ddd;">名次</th>
        <th style="padding:6px;border:1px solid #ddd;">組別</th>
        <th style="padding:6px;border:1px solid #ddd;">名稱</th>
        <th style="padding:6px;border:1px solid #ddd;">最終收益</th></tr>
      </thead><tbody>`;

  arr.forEach((it, idx) => {
    html += `<tr><td style="padding:6px;border:1px solid #ddd;">${idx+1}</td>
             <td style="padding:6px;border:1px solid #ddd;">${it.group}</td>
             <td style="padding:6px;border:1px solid #ddd;">${it.name}</td>
             <td style="padding:6px;border:1px solid #ddd;">${it.score} 萬 NTD</td></tr>`;
  });

  html += "</tbody></table>";
  document.getElementById("rankBoard").innerHTML = html;
}

/* ------------------------------
   顯示錯題檢討表
------------------------------ */
function showWrongAnswers() {
  const div = document.getElementById("wrongAnswersDiv");
  if (!div) return;
  if (wrongAnswers.length === 0) {
    div.innerHTML = "<p>恭喜沒有任何失誤！你是集幸運與睿智於一身之人。</p>";
    return;
  }
  let html = "<h3>❌ 錯題檢討表</h3><ol>";
  wrongAnswers.forEach(w => {
    html += `<li><strong>題目：</strong>${w.question}<br>
             <strong>你選擇：</strong>${w.wrongChoice}<br>
             <strong>建議答案：</strong>${w.correctChoice}<br>
             <strong>原因：</strong>${w.reason}</li><br>`;
  });
  html += "</ol>";
  div.innerHTML = html;
}

/* ------------------------------
   初始化顯示控制：當遊戲出現時顯示 invest 按鈕
------------------------------ */
(function initializeUIControls() {
  // 確保 invest 按鈕與 herdSize 顯示在遊戲開始後
  const gameEl = document.getElementById("game");
  const observer = new MutationObserver(() => {
    if (gameEl.style.display === "block") {
      document.getElementById("investBtn") && (document.getElementById("investBtn").classList.remove("hidden"));
      // 顯示 herdSize（若已存在）
      const herd = document.getElementById("herdSize");
      if (herd) herd.innerText = herdSize;
    }
  });
  observer.observe(gameEl, { attributes: true, attributeFilter: ["style"] });
})();
</script>

<script>
/* ------------------------------
   題庫（含你提供的所有情境）
   每題 options 含 baseEffect（萬 NTD 單位）、msg、correct、reason
------------------------------ */
const scenarios = [
  { description: "乳量下降 10%，體細胞上升至 38萬/mL。",
    options:[
{ text:"提高精料比例以刺激乳量", baseEffect:-5, msg:"短期可能增加但增加疾病風險。", correct:false, reason:"精料過高會增加瘤胃酸中毒與健康風險" },
{ text:"減少擠乳次數以讓乳房休息", baseEffect:-6, msg:"可能導致脹奶、細菌滋生。", correct:false, reason:"減少擠乳會增加乳房感染與產量下降風險" },
{ text:"改善牛床乾燥度與墊料", baseEffect:10, msg:"體細胞下降，乳量回升！", correct:true, reason:"改善環境可降低乳房炎風險" }
    ]
  },
  { description: "泌乳初期（30 DIM）乳脂率僅 2.8%，疑似負能量平衡。",
    options:[
      { text:"提高乾物攝取量、改善日糧適口性", baseEffect:9, msg:"DMI 上升，乳脂正常化！", correct:true, reason:"增加能量攝取改善乳脂率" },
      { text:"減少總飼料以促進瘤胃健康", baseEffect:-4, msg:"會惡化負能量平衡。", correct:false, reason:"降低飼料會使能量不足更嚴重" },
      { text:"立即停用所有營養補充劑", baseEffect:-3, msg:"移除可能有益的補充劑，效果不佳。", correct:false, reason:"補充劑可能有助恢復能量平衡" }
    ]
  },

  /* 以下為你新增的大量題目（我已把你提供的情境逐一加入） */
  { description: "全場平均乳量 20 kg，牛群食慾下降，P/F升高。",
    options:[
{ text:"立即提高精料比例大量補能", baseEffect:-5, msg:"短期可能增加但風險較高。", correct:false, reason:"盲目提高精料可能造成瘤胃問題" },
{ text:"減少牛群活動量以節省能量", baseEffect:-2, msg:"非根本解決方案。", correct:false, reason:"減少活動不會恢復食慾" },
{ text:"檢查環境溫度、通風與飲水系統是否有問題", baseEffect:8, msg:"發現熱緊迫/飲水問題並改善，DMI 回升。", correct:true, reason:"熱緊迫與飲水問題常導致食慾下降" }
    ]
  },
  { description: "全場平均乳量 26 kg，部分牛隻體況偏瘦。",
    options:[
{ text:"統一減少飼料以控制成本", baseEffect:-6, msg:"可能惡化體況。", correct:false, reason:"減少飼料會使瘦牛更差" },
{ text:"評估個體體況，增加高能量補料與個別照護", baseEffect:7, msg:"體況回升，產量維持或改善。", correct:true, reason:"針對瘦牛補充能量與照護有助恢復" },
{ text:"增加搾乳前等待到搾乳完回舍進食的時間，分散回舍牛群以減少群內競爭", baseEffect:2, msg:"可能稍有幫助但非主要處方。", correct:false, reason:"管理調整對於部分弱勢牛可能有幫助但需配合營養，策略不完整" }
    ]
  },
  { description: "檸檬酸 80 mg/dL，乳脂率 3.0%，乳量 25 kg。",
    options:[
      { text:"檢查是否有乳房炎或熱緊迫，並做乳房檢查與環境改善", baseEffect:8, msg:"若為乳房炎或熱緊迫，對症處理可改善產量/品質。", correct:true, reason:"低檸檬酸可能與乳房炎或熱緊迫相關" },
      { text:"增加精料以追高乳脂，提高泌乳牛儲備能量", baseEffect:-4, msg:"可能無效且有風險。", correct:false, reason:"盲目增加精料非最佳策略" },
      { text:"立即替換高乳脂乳牛品種", baseEffect:-10, msg:"極端且無效。", correct:false, reason:"非短期可行方案" }
    ]
  },
  { description: "體細胞數 9 萬/mL，乳蛋白率 3.6%。",
    options:[
      { text:"增加抗生素使用以追求更低SCC，精益求精", baseEffect:-3, msg:"不必要且有抗藥性風險。", correct:false, reason:"過度用藥不可取" },
	{ text:"維持現狀，加強日常監測", baseEffect:5, msg:"數值屬佳，維持管理並監測即可。", correct:true, reason:"SCC 低且蛋白良好，過度干預反而有風險" },
      { text:"減少飼料以降低乳蛋白，避免高蛋白造成牛隻負擔", baseEffect:-4, msg:"會造成產量與健康問題。", correct:false, reason:"不宜減少飼料" }
    ]
  },
  { description: "體細胞數 28 萬/mL，日產乳量 27 kg。",
    options:[
      { text:"針對高SCC群體做 CMT 篩檢並實施局部處置", baseEffect:8, msg:"針對性治療可降低感染並提升產量。", correct:true, reason:"CMT 篩檢有助於找出感染牛" },
      { text:"立即淘汰SCC高於10萬的牛", baseEffect:-8, msg:"處置過度且不經濟。", correct:false, reason:"不應大規模淘汰" },
      { text:"停用飼料添加劑，全場施用抗生素", baseEffect:-3, msg:"無直接關聯，在未區分健康與不健康牛隻的情況下進行不合理極端處置。", correct:false, reason:"非優先措施，在未區分健康與不健康牛隻的情況下進行不合理極端處置" }
    ]
  },
  { description: "SCC 55 萬/mL，乳房腫脹。",
    options:[
      { text:"提高日糧能量，以彌補損失", baseEffect:-4, msg:"無法處理感染來源。", correct:false, reason:"營養調整非首選" },
      { text:"延後擠乳以讓腫脹消退", baseEffect:-6, msg:"會增加乳房傷害與感染擴散。", correct:false, reason:"延後擠乳有害" },
{ text:"隔離治療，檢查擠乳設備", baseEffect:10, msg:"針對性治療降低疼痛與感染，改善品質。", correct:true, reason:"腫脹+高SCC需即刻處理" }
    ]
  },
  { description: "場均 SCC 75 萬/mL，乳量 29 kg，乳蛋白率 4.1%。",
    options:[
      { text:" 請獸醫進場檢測、隔離感染牛並檢討擠乳流程與設備", baseEffect:9, msg:"改善感染源，長期可提升品質。", correct:true, reason:"高場均SCC需全面介入，甚至可能需要全面淘汰止損" },
      { text:"適度調整精料、芻料比以提高乳量", baseEffect:-5, msg:"可能掩蓋問題造成場內乳房炎問題惡化。", correct:false, reason:"非根本解決" },
      { text:"降低蛋白餵飼減少牛隻代謝負擔，乳量部分講求維持即可", baseEffect:-10, msg:" 未處理急切且嚴重的乳房炎現象，可能造成場內乳房炎問題惡化、因乳品質而受處罰。", correct:false, reason:" 非根本解決" }
    ]
  },
  { description: "乳蛋白率 2.9%，乳量 30 kg，乳脂正常。",
    options:[
     { text:"適度減少糧食供給以提高乳蛋白濃度", baseEffect:-4, msg:"會降低產量。", correct:false, reason:"減少總營養供給非良策" },
{ text:"評估蛋白來源與MUN，補充蛋白質或調整胺基酸平衡", baseEffect:7, msg:"改善蛋白率且非犧牲產量。", correct:true, reason:"MUN與蛋白率相關，低乳蛋白可能需調整營養配方" },
      { text:"全場休息擠乳一日以測試數據，等泌乳牛累積足夠乳蛋白再行搾乳", baseEffect:-2, msg:"無幫助且忽略脹乳的緊迫。", correct:false, reason:"不建議" }
    ]
  }
  ];

//追加題庫：
scenarios.push(
  { description: "乳蛋白率 4.1%，乳量 30 kg，牛群步態怪異。",
    options:[
      { text:"檢查蹄部健康、牛床與通道防滑性", baseEffect:8, msg:"改善步態並降低受傷風險。", correct:true, reason:"高蛋白+步態異常可能與代謝或蹄病相關" },
      { text:"增加精料追求更高乳蛋白", baseEffect:-5, msg:"可能加劇代謝負擔，導致炎症加劇。", correct:false, reason:"蛋白已高，再追高會加劇身體負擔" },
      { text:"維持營養配方，但盡量減少擠乳次數讓牛休息", baseEffect:-3, msg:"休息非主要措施，可能的高蛋白飼糧問題也可能讓牛隻持續緊迫。", correct:false, reason:"步態問題多半不是擠乳次數造成" }
    ]
  },

  { description: "多數牛為泌乳初期，場均乳脂率2.9%，乳量 28 kg，體態偏瘦。",
    options:[
      { text:"增加飼糧能量濃度", baseEffect:9, msg:"改善負能量平衡並提升乳脂。", correct:true, reason:"泌乳初期乳脂低+瘦常表示能量不足" },
      { text:"大量減少乾草餵飼比例，增加適口性高的精料比例", baseEffect:-4, msg:"會影響瘤胃健康。", correct:false, reason:"纖維不足更降低乳脂，且有瘤胃酸化危機" },
      { text:"讓牛多休息、不調整飼料，注意飲水清潔", baseEffect:-2, msg:"不足以改善問題。", correct:false, reason:"需調整日糧更為關鍵" }
    ]
  },

  { description: "泌乳初期，乳脂率 3.9%，日產乳量 30 kg，精神活力明顯下降。",
    options:[
     { text:"降低精料量以降低代謝壓力", baseEffect:-6, msg:"會使能量更不足。", correct:false, reason:"酮症需提高能量" },
{ text:"增加能量來源、提供丙酸鹽等補充料", baseEffect:8, msg:"改善酮症指標。", correct:true, reason:"酮症主要來自能量不足" },
      { text:"延長擠乳間隔，檢查灑水系統", baseEffect:-3, msg:"無幫助。", correct:false, reason:"擠乳間隔、散熱對酮症無直接改善" }
    ]
  },

  { description: "乳糖 4.3%，乳量 25 kg，P/F=0.89，SCC 36 萬/mL。",
    options:[
      { text:"增加精料以提升乳糖", baseEffect:-4, msg:"無法改善感染。", correct:false, reason:"低乳糖、稍高的SCC與P/F更可能與感染相關，多半不是精料問題" },
      { text:"減少飲水供應以濃縮乳汁", baseEffect:-6, msg:"危險且無效。", correct:false, reason:"造成飲水不足，不改善乳糖" },
{ text:"檢查乳房健康與擠乳程序衛生", baseEffect:8, msg:"控制感染後乳糖會回升。", correct:true, reason:"乳糖偏低且 SCC 高，多為乳房炎" }
    ]
  },

  { description: "乳糖 4.8%，乳量 30 kg，P/F=0.74。",
    options:[
      { text:"提高精料比例以增加蛋白攝取", baseEffect:-4, msg:"風險更高。", correct:false, reason:"精料過高會使 P/F 更低" },
      { text:"維持目前管理策略，定期觀察牛隻飲食精神狀態", baseEffect:-2, msg:"風險持續存在。", correct:false, reason:"需調整飼糧" },
{ text:"調整精粗比，增加有效粗纖維攝取", baseEffect:7, msg:"改善瘤胃穩定度並提高乳脂。", correct:true, reason:"低 P/F 可能有亞酸中毒風險" }
    ]
  },

  { description: "乳糖 5.1%，乳量 32 kg，有些牛出現輕微便秘跡象。",
    options:[
      { text:"提高精料以稀釋乳糖", baseEffect:-5, msg:"完全無效。", correct:false, reason:"乳糖偏高多數是脫水問題，或者營養已經充足，提高精料並非有效改善策略" },
      { text:"減少乾草以降低水分需求", baseEffect:-3, msg:"錯誤且有害。", correct:false, reason:"乾草與乳糖無直接關聯，且減少乾草與水分需求也無直接關係" },
{ text:"檢查飲水設備、增加水槽並改善補水速度", baseEffect:9, msg:"改善脫水並維持產量。", correct:true, reason:"飲水不足會使乳糖偏高" }
    ]
  },
  { description: "MUN 9 mg/dL，乳蛋白率 2.9%，乳量 32kg。",
    options:[
      { text:"減少精料，避免瘤胃酸中毒", baseEffect:-4, msg:"會使蛋白更低。", correct:false, reason:"需增加蛋白來源而非減少" },
	{ text:"增加過瘤胃蛋白比例或調整胺基酸平衡", baseEffect:7, msg:"改善蛋白代謝。", correct:true, reason:"低 MUN + 低蛋白常為蛋白不足，處置合理" },
      { text:"增加水分攝取以稀釋 MUN", baseEffect:-2, msg:"無關。", correct:false, reason:"MUN 與水量無直接關聯，且MUN無須再降低" }
    ]
  },

  { description: "乳蛋白率 2.8%，MUN 21 mg/dL，乳量 28 kg，最近剛調整過精料比例。",
    options:[
      { text:"檢查是否精料過量或蛋白性質不宜，適度提高過瘤胃蛋白比例", baseEffect:8, msg:"改善瘤胃分解蛋白產生氨，降低MUN。", correct:true, reason:"高 MUN 可能與精料或蛋白相關" },
      { text:"提高非蛋白氮比例以平衡 MUN", baseEffect:-4, msg:"會更糟。", correct:false, reason:" 非蛋白氮（比如尿素）會增加 MUN" },
      { text:"減少飲水，以濃縮乳蛋白率", baseEffect:-5, msg:"完全無效。", correct:false, reason:"與 MUN 無直接關聯且可能有害" }
    ]
  },
  { description: "檸檬酸 105 mg/dL，乳脂率 2.9%，乳量 29 kg，牛群常臥地喘氣。",
    options:[
      { text:"精料與芻料同時提高以補足能量與乳脂來源", baseEffect:-4, msg:"忽視熱緊迫問題。", correct:false, reason:"能量過剩、無效餵與飼糧+熱緊迫反而更危險" },
      { text:"減少活動空間以分散熱源，配合乳固形物數值調整飼養策略", baseEffect:-2, msg:" 減少活動空間反而可能阻礙散熱，乳固形物數值也通常相對較不直接顯示代謝問題。", correct:false, reason:"主要是通風與飲水問題" },
{ text:"改善通風、水霧/灑水系統，配合SCC與P/F值調整飼養策略", baseEffect:8, msg:"降低熱緊迫並改善乳脂。", correct:true, reason:"熱緊迫造成低採食、低乳脂，乳腺細胞代謝率降低導致低檸檬酸" }
    ]
  },
  { description: "檸檬酸 210 mg/dL，乳脂率 4.0%，乳量 40 kg。",
    options:[
      { text:"適度補充能量避免過度負能量，並提升水分供應", baseEffect:7, msg:"維持高產穩定性。", correct:true, reason:"高產牛泌乳初期伴隨高檸檬酸" },
      { text:"加強監測，針對乳房炎高危險群進行檢測、隔離或治療", baseEffect:-4, msg:"與DHI顯示之問題較無直接關係。", correct:false, reason:" 高檸檬酸通常非與乳房炎相關" },
      { text:"減少粗料以提升採食速度，間接增加採食量", baseEffect:-5, msg:"瘤胃風險上升。", correct:false, reason:"粗料比例不能輕易減少" }
    ]
  },

  { description: "場均游離脂肪酸 2.0 mmol/100g，乳量 27 kg，乳蛋白率4.0%。",
    options:[
{ text:"泌乳牛全體施用預防性抗生素，鞏固牛群防疫", baseEffect:-3, msg:"無助且可能有害。", correct:false, reason:"不能直接由FFA、乳蛋白確定所有牛隻患有乳房炎" },
{ text:"確認SCC檢測結果，同步檢查儲乳設備與生乳運送冷鏈系統 ", baseEffect:8, msg:"降低脂肪酸指標。", correct:true, reason:" FFA可能與設備有關，加上高乳蛋白有機率與乳房炎相關" },
      { text:"提高精料、確認飲水充足以降低 FFA", baseEffect:-4, msg:"反效果。", correct:false, reason:"高精料反而可能升高 FFA" }
    ]
  },
{ description: "MUN 20 mg/dL，乳蛋白率 4.3%，乳量 25 kg，受孕率降低。",
    options:[
      { text:" 增加非蛋白氮，或額外蛋白補充以嘗試維持目前高乳蛋白率", baseEffect:-4, msg:"可能惡化問題。", correct:false, reason:"精料過量造成 MUN 升高" },
{ text:" 在不增加蛋白攝取總量的前提下，降低瘤胃可分解蛋白比例", baseEffect:9, msg:"改善 MUN 與繁殖。", correct:true, reason:"降低瘤胃可分解蛋白可降低 MUN，可能解決繁殖下降問題" },
      { text:"停止補充精料，以芻料為主要飼糧", baseEffect:-3, msg:"不合理。", correct:false, reason:"需調整不是停止" }
    ]
  },
  { description: "場均乳量26 kg，乳蛋白率 3.2%，乳脂率 4.5%，乳糖 4.7%。",
    options:[
      { text:"提高精料，增加容易吸收的蛋白質攝取以改善P/F值", baseEffect:-4, msg:"缺乏纖維的狀況可能會更嚴重。", correct:false, reason:"精料過高降低乳脂與健康" },
	{ text:"提升乾草有效纖維量，並確認TMR混合的均勻狀況", baseEffect:8, msg:"改善瘤胃狀態。", correct:true, reason:"低P/F可能來自低纖維或挑食" },
      { text:"增加飲水，並加強人員搾乳操作衛生", baseEffect:-3, msg:"無直接效益，可能忽略營養代謝問題。", correct:false, reason:"主因應非飲水或疾病" }
    ]
  },

  {description: "乳量 35 kg，乳蛋白率4.0%， 乳脂率3.9%，近日牛群易眼神渙散呆滯。",
    options:[
      { text:"進行代謝疾病、蹄病與環境壓力評估", baseEffect:9,
        msg:"找出健康問題，改善後恢復精神。", correct:true,
        reason:"高產+精神差可能與臨床疾病相關，或者是缺乏能量" },
      { text:"提高場內通風效果，改善精神食欲 ", baseEffect:-5, msg:"可能無法改善疾病或營養問題。", correct:false,
        reason:"不能斷定為純粹的熱緊迫問題，對於改善降低P/F無幫助" },
      { text:"減少芻料餵與量，尤其是含水量較高的鮮草，以提升粗纖維攝取量", baseEffect:-4, msg:"於改善P/F無益，問題持續或惡化。", correct:false,
        reason:"若問題是缺乏能量，單純增加芻料沒有直接幫助" }
    ]
  },
 {description: "乳量 12 kg，乳蛋白率4.1%， 乳脂率4.6%，SCC 21萬/mL。",
    options:[
      { text:"給予全乳房抗生素治療以降低體細胞數", baseEffect:-3,
        msg:"輕易使用抗生素，反而造成無效支出與防疫危機。", correct:false,
        reason:"21 萬屬正常偏高，不能單純以此決斷力及使用抗生素治療" },
      { text:"提高精料比以及飲水，以提升乳量 ", baseEffect:-5, msg:"非精料補充的問題。", correct:false,
        reason:"情境非能量不足，不能因此增加精料" },
      { text:"漸退式安排該牛進入乾乳", baseEffect:-4, msg:"處置合理，牛隻能更快進入下一個健康泌乳期", correct:true,
        reason:"泌乳後期牛應考慮逐漸乾乳，休息恢復乳腺組織" }
    ]
  },
  { description: "游離脂肪酸 2.2 mmol/100g，乳量 24 kg，SCC 31萬/mL。",
    options:[
      
      { text:"增加精料，彌補乳量下降", baseEffect:-4, msg:"無助。", correct:false, reason:"不是能量問題" },
      { text:"減少精料，避免瘤胃酸中毒", baseEffect:-3, msg:"未解決根本問題。", correct:false, reason:"FFA、SCC多半與營養無直接關係" },
{ text:"確認搾乳設備與牛床清潔", baseEffect:9, msg:"改善乳品質。", correct:true, reason:"FFA、SCC與設備衛生高度相關" },
    ]
  },
 { description: "BHB 130μmol/L，乳量 25 kg，乳脂率 3.2%，牛群體況偏瘦。",
    options:[
      { text:"減少精料給與，避免瘤胃鼓脹惡化", baseEffect:-6, msg:"能量缺乏狀況更糟。", correct:false, reason:"需要提升能量" },
{ text:"增加能量供應，嘗試添加啤酒酵母", baseEffect:8, msg:"改善能量平衡。", correct:true, reason:"高 BHB 多為能量不足（酮症），啤酒酵母提升飼糧適口性" },
      { text:"改善飲水供應，飼糧中提高大豆粕及魚粉比例", baseEffect:-3, msg:" 蛋白質補充無益於改善酮症。", correct:false, reason:"蛋白質補充無益於改善酮症" }
	]
 }
);
</script>
  
  <div id="wrongAnswersDiv" style="margin-top:20px;"></div>
  
</body>
</html>
