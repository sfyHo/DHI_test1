<!DOCTYPE html>
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
  <p id="result"></p>
  <p>目前收益：<span id="score">0</span> 萬 NTD</p>
<p id="result"></p>
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
  }, 3000);
}
</script>

<script>
/* ------------------------------
   計分模型：收益遞減 + 規模放大
   baseEffect 單位：萬 NTD
------------------------------ */
function calculateEffectiveGain(baseEffect) {
  // (A) 收益越高，提升難度（使用絕對值以避免 score 為負時奇怪行為）
  const diminishing = 100 / (100 + Math.abs(score) + 0.0001);
  // (B) herdSize 對變動幅度的放大（以 30 頭為基準）
  const scale = herdSize / 30;
  const eff = baseEffect * diminishing * scale;
  return Math.round(eff); // 回傳整數（萬 NTD）
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

  // 小延遲後自動下一題或結束（2 秒）
  setTimeout(() => {
    current++;
    if (current >= scenarios.length) {
      endGame();
    } else {
      loadQuestion();
    }
  }, 2000);
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
    div.innerHTML = "<p>恭喜！沒有錯題。</p>";
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
  { description: "乳量下降 10%，體細胞上升至 380k。",
    options:[
      { text:"改善牛床乾燥度與墊料", baseEffect:10, msg:"體細胞下降，乳量回升！", correct:true, reason:"改善環境可降低乳房炎風險" },
      { text:"提高精料比例以刺激乳量", baseEffect:-5, msg:"短期可能增加但增加疾病風險。", correct:false, reason:"精料過高會增加瘤胃酸中毒與健康風險" },
      { text:"減少擠乳次數以讓乳房休息", baseEffect:-6, msg:"可能導致脹奶、細菌滋生。", correct:false, reason:"減少擠乳會增加乳房感染與產量下降風險" }
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
  { description: "全場平均乳量 24 kg，牛群食慾下降。",
    options:[
      { text:"檢查環境溫度、通風與飲水系統是否有問題", baseEffect:8, msg:"發現熱緊迫/飲水問題並改善，DMI 回升。", correct:true, reason:"熱緊迫與飲水問題常導致食慾下降" },
      { text:"立即提高精料比例大量補能", baseEffect:-5, msg:"短期可能增加但風險較高。", correct:false, reason:"盲目提高精料可能造成瘤胃問題" },
      { text:"減少牛群活動量以省能量", baseEffect:-2, msg:"非根本解決方案。", correct:false, reason:"減少活動不會恢復食慾" }
    ]
  },
  { description: "全場平均乳量 26 kg，部分牛隻體況偏瘦。",
    options:[
      { text:"評估個體體況，增加高能量補料與個別照護", baseEffect:7, msg:"體況回升，產量維持或改善。", correct:true, reason:"針對瘦牛補充能量與照護有助恢復" },
      { text:"統一減少飼料以控制成本", baseEffect:-6, msg:"可能惡化體況。", correct:false, reason:"減少飼料會使瘦牛更差" },
      { text:"增加羈留時間以減少群內競爭", baseEffect:3, msg:"可能有小幫助但非主要處方。", correct:false, reason:"管理調整有幫助但需配合營養" }
    ]
  },
  { description: "全場平均乳量 31 kg，糞便偏稀，偶有腹脹現象。",
    options:[
      { text:"檢查並降低精料比例、增加纖維並分批餵食", baseEffect:9, msg:"糞便恢復正常，乳量穩定。", correct:true, reason:"高精料/低纖維常導致瘤胃酸中毒、稀便" },
      { text:"增加抗菌藥物作為預防", baseEffect:-5, msg:"可能掩蓋症狀並影響菌群。", correct:false, reason:"不應先用抗生素替代飼糧調整" },
      { text:"不做任何變動，觀察一週", baseEffect:-2, msg:"問題可能惡化。", correct:false, reason:"需積極改善飼糧與管理" }
    ]
  },
  { description: "檸檬酸 80 mg/dL，乳脂率 3.0%，乳量 25 kg。",
    options:[
      { text:"檢查是否有乳房炎或熱緊迫，並做乳房檢查與環境改善", baseEffect:8, msg:"若為乳房炎或熱緊迫，對症處理可改善產量/品質。", correct:true, reason:"低檸檬酸可能與乳房炎或熱緊迫相關" },
      { text:"增加精料以追高乳脂", baseEffect:-4, msg:"可能無效且有風險。", correct:false, reason:"盲目增加精料非最佳策略" },
      { text:"立即替換全群乳牛品種", baseEffect:-10, msg:"極端且無效。", correct:false, reason:"非短期可行方案" }
    ]
  },
  { description: "體細胞數 9 萬/mL，乳蛋白率 3.6%。",
    options:[
      { text:"維持現狀並加強常規監測", baseEffect:5, msg:"數值屬佳，維持管理並監測即可。", correct:true, reason:"SCC 低且蛋白良好，過度干預反而有風險" },
      { text:"大幅增加抗生素使用以追求更低SCC", baseEffect:-3, msg:"不必要且有抗藥性風險。", correct:false, reason:"過度用藥不可取" },
      { text:"減少飼料以降低乳蛋白", baseEffect:-4, msg:"會造成產量與健康問題。", correct:false, reason:"不宜減少飼料" }
    ]
  },
  { description: "槽乳體細胞數 18 萬/mL，乳房炎病例零星出現。",
    options:[
      { text:"加強擠乳衛生、隔離感染牛與檢測篩查", baseEffect:9, msg:"病例數下降，SCC 改善。", correct:true, reason:"控制感染源與擠乳衛生可有效降低乳房炎" },
      { text:"提高奶站收購標準以懲罰農場", baseEffect:-2, msg:"對當下改善無助，且非內部措施。", correct:false, reason:"外部手段非解決根本" },
      { text:"減少擠乳次數", baseEffect:-5, msg:"可能增加乳房問題。", correct:false, reason:"減少擠乳非良策" }
    ]
  },
  { description: "體細胞數 28 萬/mL，場均日產乳量 27 kg。",
    options:[
      { text:"針對高SCC群體做 CMT 篩檢並實施局部處置", baseEffect:8, msg:"針對性治療可降低感染並提升產量。", correct:true, reason:"CMT 篩檢有助於找出感染牛" },
      { text:"立即淘汰所有SCC高於10萬的牛", baseEffect:-8, msg:"過度且不經濟。", correct:false, reason:"不應大規模淘汰" },
      { text:"完全停用飼料添加劑", baseEffect:-3, msg:"無直接關聯。", correct:false, reason:"非優先措施" }
    ]
  },
  { description: "SCC 55 萬/mL，乳房腫脹。",
    options:[
      { text:"立即隔離並治療腫脹牛，檢查擠乳設備", baseEffect:10, msg:"針對性治療降低疼痛與感染，改善品質。", correct:true, reason:"腫脹+高SCC需即刻處理" },
      { text:"提高日糧能量以彌補損失", baseEffect:-4, msg:"無法處理感染來源。", correct:false, reason:"營養調整非首選" },
      { text:"延後擠乳以讓腫脹消退", baseEffect:-6, msg:"會增加乳房傷害與感染擴散。", correct:false, reason:"延後擠乳有害" }
    ]
  },
  { description: "場均 SCC 85 萬/mL，乳量 29 kg，乳蛋白率 4.1%。",
    options:[
      { text:"做全場檢測、隔離感染牛並檢討擠乳流程與設備", baseEffect:9, msg:"改善感染源，長期可提升品質。", correct:true, reason:"高場均SCC需全面介入" },
      { text:"提高精料以維持乳量", baseEffect:-5, msg:"可能掩蓋問題並惡化瘤胃負擔。", correct:false, reason:"非根本解決" },
      { text:"忽略SCC，只關注乳量", baseEffect:-10, msg:"長期會導致處罰與品質下降。", correct:false, reason:"不可忽視品質指標" }
    ]
  },
  { description: "乳蛋白率 2.9%，乳量 30 kg，乳脂正常。",
    options:[
      { text:"評估蛋白來源與MUN，可能補充高品質蛋白或調整RDP", baseEffect:7, msg:"改善蛋白率且非犧牲產量。", correct:true, reason:"MUN與蛋白率相關，需調整配方" },
      { text:"減少糧食供給以提高濃度", baseEffect:-4, msg:"會降低產量。", correct:false, reason:"減少供給非良策" },
      { text:"立即停止挤乳一次以測試數據", baseEffect:-2, msg:"無幫助且風險高。", correct:false, reason:"不建議" }
    ]
  }
  ];
<script>
/* ------------------------------
   題庫（延續前一段，這裡是後半部題目）
------------------------------ */
// 直接接在前一段 scenarios 陣列後方追加：
scenarios.push(
  { description: "乳蛋白率 4.1%，乳量 30 kg，牛群步態怪異。",
    options:[
      { text:"檢查蹄部健康、牛床與通道防滑性", baseEffect:8, msg:"改善步態並降低受傷風險。", correct:true, reason:"高蛋白+步態異常可能與代謝或蹄病相關" },
      { text:"增加精料追求更高乳蛋白", baseEffect:-5, msg:"可能加劇代謝負擔。", correct:false, reason:"蛋白已高，再追高會增加身體負擔" },
      { text:"減少擠乳次數以讓牛休息", baseEffect:-3, msg:"非主要原因。", correct:false, reason:"步態問題不是擠乳次數造成" }
    ]
  },

  { description: "泌乳初期乳脂率 2.9%，乳量 28 kg，體態偏瘦。",
    options:[
      { text:"增加能量密度並改善過渡期管理", baseEffect:9, msg:"改善負能量平衡並提升乳脂。", correct:true, reason:"泌乳初期乳脂低+瘦常表示能量不足" },
      { text:"減少乾草比例以增加適口性", baseEffect:-4, msg:"會影響瘤胃健康。", correct:false, reason:"纖維不足更降低乳脂" },
      { text:"讓牛多休息、不調整飼料", baseEffect:-2, msg:"不足以改善問題。", correct:false, reason:"需調整日糧" }
    ]
  },

  { description: "乳脂率 3.5%，乳量 27 kg，反芻次數偏少。",
    options:[
      { text:"增加纖維來源、改善TMR均勻度", baseEffect:7, msg:"反芻增加、瘤胃更穩定。", correct:true, reason:"偏少反芻常來自纖維不足或挑食" },
      { text:"提高精料以刺激食慾", baseEffect:-5, msg:"可能降低反芻。", correct:false, reason:"精料過高會壓抑反芻" },
      { text:"減少飲水供應以降低糞便量", baseEffect:-6, msg:"危險且無效。", correct:false, reason:"飲水與反芻無直接關聯" }
    ]
  },

  { description: "乳脂率 3.9%，乳量 29 kg，有輕微酮症跡象。",
    options:[
      { text:"增加能量來源、提供丙酸鹽等補充", baseEffect:8, msg:"改善酮症指標。", correct:true, reason:"酮症主要來自能量不足" },
      { text:"完全停止精料以降低壓力", baseEffect:-6, msg:"會使能量更不足。", correct:false, reason:"酮症需提高能量" },
      { text:"延長擠乳間隔讓牛休息", baseEffect:-3, msg:"無幫助。", correct:false, reason:"擠乳間隔與酮症無直接改善" }
    ]
  },

  { description: "乳糖 4.3%，乳量 25 kg，SCC 53 萬/mL。",
    options:[
      { text:"優先檢查乳房健康與擠乳衛生", baseEffect:8, msg:"控制感染後乳糖會回升。", correct:true, reason:"乳糖偏低且 SCC 高，多為乳房炎" },
      { text:"增加精料以提升乳糖", baseEffect:-4, msg:"無法改善感染。", correct:false, reason:"乳糖與感染相關，不是精料問題" },
      { text:"減少飲水供應以濃縮乳汁", baseEffect:-6, msg:"危險且無效。", correct:false, reason:"飲水不足不改善乳糖" }
    ]
  },

  { description: "乳糖 4.8%，乳量 30 kg，P/F=0.86。",
    options:[
      { text:"調整精粗比，增加有效纖維", baseEffect:7, msg:"改善瘤胃穩定度並提高乳脂。", correct:true, reason:"低 P/F 可能有亞酸中毒風險" },
      { text:"提高精料比例以增加蛋白", baseEffect:-4, msg:"風險更高。", correct:false, reason:"精料過高會使 P/F 更低" },
      { text:"不調整任何管理", baseEffect:-2, msg:"風險持續存在。", correct:false, reason:"需調整飼糧" }
    ]
  },

  { description: "乳糖 5.1%，乳量 32 kg，有輕微脫水狀況。",
    options:[
      { text:"檢查飲水設備、增加水槽並改善補水速度", baseEffect:9, msg:"改善脫水並維持產量。", correct:true, reason:"飲水不足會使乳糖偏高" },
      { text:"提高精料以稀釋乳糖", baseEffect:-5, msg:"完全無效。", correct:false, reason:"乳糖偏高是脫水問題" },
      { text:"減少乾草以降低水分需求", baseEffect:-3, msg:"錯誤且有害。", correct:false, reason:"乾草與乳糖無直接關聯" }
    ]
  },

  { description: "MUN 9 mg/dL，乳蛋白率 2.9%，乳量 26 kg。",
    options:[
      { text:"增加 RDP 來源或提高蛋白質品質", baseEffect:7, msg:"改善蛋白代謝。", correct:true, reason:"低 MUN + 低蛋白常為蛋白不足" },
      { text:"減少精料", baseEffect:-4, msg:"會使蛋白更低。", correct:false, reason:"需增加蛋白而非減少" },
      { text:"增加水分以稀釋 MUN", baseEffect:-2, msg:"無關。", correct:false, reason:"MUN 與水量無直接關聯" }
    ]
  },

  { description: "MUN 18 mg/dL，乳蛋白率 3.3%，乳量 28 kg，最近剛調整過比例。",
    options:[
      { text:"檢查是否精料過量或蛋白分解過高，適度調整", baseEffect:8, msg:"改善 MUN 並提升效率。", correct:true, reason:"高 MUN 可能與精料或蛋白相關" },
      { text:"提高 NPN 來源以平衡 MUN", baseEffect:-4, msg:"會更糟。", correct:false, reason:"NPN 會增加 MUN" },
      { text:"減少飲水", baseEffect:-5, msg:"完全無效。", correct:false, reason:"與 MUN 無直接關聯" }
    ]
  },

  { description: "MUN 20 mg/dL，乳蛋白率 3.2%，乳量 25 kg，受孕率降低。",
    options:[
      { text:"調整蛋白分解與能量平衡，改善日糧結構", baseEffect:9, msg:"改善 MUN 與繁殖。", correct:true, reason:"高 MUN 與繁殖下降有關" },
      { text:"增加精料以補能", baseEffect:-4, msg:"可能惡化問題。", correct:false, reason:"精料過量造成 MUN 升高" },
      { text:"完全停止補料", baseEffect:-3, msg:"不合理。", correct:false, reason:"需調整不是停止" }
    ]
  },

  { description: "檸檬酸 110 mg/dL，乳脂率 2.9%，乳量 26 kg，牛群有熱緊迫。",
    options:[
      { text:"改善通風、水霧/灑水系統，並補充能量", baseEffect:8, msg:"降低熱緊迫並改善乳脂。", correct:true, reason:"熱緊迫降低 DMI 與乳脂" },
      { text:"提高精料以補足能量", baseEffect:-4, msg:"會加劇問題。", correct:false, reason:"精料過高+熱緊迫更危險" },
      { text:"減少活動空間以分散熱源", baseEffect:-2, msg:"無助。", correct:false, reason:"主要是通風與飲水問題" }
    ]
  },

  { description: "檸檬酸 200 mg/dL，乳脂率 4.0%，乳量 40 kg。",
    options:[
      { text:"適度補充能量避免過度負能量，並提升水分供應", baseEffect:7, msg:"維持高產穩定性。", correct:true, reason:"泌乳初期高產伴隨高 citric acid" },
      { text:"大量提高精料追求更高乳量", baseEffect:-4, msg:"風險大。", correct:false, reason:"不宜再壓榨營養" },
      { text:"減少粗料以提升採食速度", baseEffect:-5, msg:"瘤胃風險上升。", correct:false, reason:"粗料不能減" }
    ]
  },

  { description: "檸檬酸 80 mg/dL，乳脂率 3.0%，乳量 25 kg，乳房炎病例增加。",
    options:[
      { text:"檢查擠乳衛生並進行 CMT 篩檢，調整環境", baseEffect:9, msg:"控制乳房炎並改善指標。", correct:true, reason:"低 citric acid + SCC 高常見於乳房炎" },
      { text:"提高精料以提升乳量", baseEffect:-3, msg:"無助。", correct:false, reason:"問題不是能量不足" },
      { text:"減少飼料以降低產量壓力", baseEffect:-4, msg:"錯誤。", correct:false, reason:"需改善感染" }
    ]
  },

  { description: "場均游離脂肪酸 2.0 mmol/100g，乳量 27 kg。",
    options:[
      { text:"改善日糧穩定性，減少挑食並維持採食規律", baseEffect:8, msg:"降低脂肪酸指標。", correct:true, reason:"不穩定採食增 FFA" },
      { text:"提高精料以降低 FFA", baseEffect:-4, msg:"反效果。", correct:false, reason:"高精料會升高 FFA" },
      { text:"減少飲水供應", baseEffect:-3, msg:"無助且有害。", correct:false, reason:"飲水與 FFA 無直接改善" }
    ]
  },

  { description: "游離脂肪酸 1.8 mmol/100g，乳量 25 kg。（搾乳設備清潔不足）",
    options:[
      { text:"全面清潔與檢查搾乳設備，改善衛生流程", baseEffect:9, msg:"降低乳脂損害與 FFA。", correct:true, reason:"FFA 升高可能與衛生與機械問題有關" },
      { text:"提高精料", baseEffect:-5, msg:"完全沒幫助。", correct:false, reason:"此問題與能量無關" },
      { text:"調整飼料結構以增加乳脂", baseEffect:-3, msg:"與 FFA 無關。", correct:false, reason:"乳脂≠FFA" }
    ]
  },

  { description: "場均乳量 29 kg，乳蛋白率 2.9%，乳脂率 3.1%，糞便稀。",
    options:[
      { text:"改善有效纖維含量與TMR混合均勻度", baseEffect:8, msg:"改善瘤胃狀態與糞便。", correct:true, reason:"稀便常來自低纖維或挑食" },
      { text:"提高精料", baseEffect:-4, msg:"會更糟。", correct:false, reason:"精料過高降低乳脂與健康" },
      { text:"減少飲水", baseEffect:-3, msg:"無效。", correct:false, reason:"原因不是飲水" }
    ]
  },

  {
description: "乳量 31 kg，乳脂率 4.0%，乳蛋白率 3.9%，近日常精神不佳。",
    options:[
      { text:"進行全身檢查，含代謝疾病、蹄病與環境壓力評估", baseEffect:9,
        msg:"找出健康問題改善後恢復精神。", correct:true,
        reason:"高產+精神差可能與亞臨床疾病/應激相關" },
      { text:"提高精料以維持高產", baseEffect:-5, msg:"可能增高壓力。", correct:false,
        reason:"高產不代表能再提高精料" },
      { text:"減少粗料以提升採食量", baseEffect:-4, msg:"會傷害瘤胃。", correct:false,
        reason:"粗料不能減" }
    ]
  },

  { description: "場均SCC 35 萬/mL，乳糖 4.4%，乳量 27 kg，乳房炎病例增加。",
    options:[
      { text:"強化擠乳衛生並做乳房炎監測", baseEffect:8, msg:"降低感染率。", correct:true, reason:"SCC 高+病例多需改善衛生" },
      { text:"提高精料補充", baseEffect:-3, msg:"非根本原因。", correct:false, reason:"與能量無直接關聯" },
      { text:"減少飲水", baseEffect:-4, msg:"無助。", correct:false, reason:"非脫水" }
    ]
  },

  { description: "MUN 19 mg/dL，乳蛋白率 3.2%，乳量 28 kg，受孕率降低。",
    options:[
      { text:"調整蛋白平衡與精料比例，降低過量蛋白", baseEffect:8, msg:"改善繁殖。", correct:true, reason:"高MUN 為蛋白過量指標" },
      { text:"增加精料", baseEffect:-5, msg:"更糟。", correct:false, reason:"精料過高提高MUN" },
      { text:"減少乾草供應", baseEffect:-3, msg:"無助。", correct:false, reason:"與繁殖無直接改善" }
    ]
  },

  { description: "檸檬酸 100 mg/dL，乳脂率 2.9%，乳量 25 kg，牛群飲水量大增。",
    options:[
      { text:"檢查是否熱緊迫、飲水設備與日糧能量密度", baseEffect:8, msg:"改善後乳脂可能回升。", correct:true, reason:"熱緊迫與能量不足均會造成飲水上升" },
      { text:"提高精料追求乳脂", baseEffect:-4, msg:"可能惡化狀況。", correct:false, reason:"熱緊迫非精料問題" },
      { text:"限制飲水以降低飲水量", baseEffect:-6, msg:"錯誤且危險。", correct:false, reason:"飲水不能限制" }
    ]
  },

  { description: "游離脂肪酸 2.2 mmol/100g，乳量 24 kg，搾乳設備清潔不足。",
    options:[
      { text:"改善設備清潔並更新墊片，調整擠乳流程", baseEffect:9, msg:"改善乳品質。", correct:true, reason:"FFA 與設備衛生高度相關" },
      { text:"增加精料以彌補乳量下降", baseEffect:-4, msg:"無助。", correct:false, reason:"不是能量問題" },
      { text:"減少粗料", baseEffect:-3, msg:"風險大。", correct:false, reason:"粗料不應減少" }
    ]
  },

  { description: "BHB 150 μmol/L，乳脂率 3.9%，乳量 25 kg，牛群有酮症危機。",
    options:[
      { text:"補充能量、改善過渡期管理並減少體脂動員", baseEffect:10, msg:"改善酮症。", correct:true, reason:"高 BHB 為酮症指標" },
      { text:"停止所有精料", baseEffect:-7, msg:"會更糟。", correct:false, reason:"需增加能量" },
      { text:"增加擠乳頻率", baseEffect:-3, msg:"無助。", correct:false, reason:"不是擠乳造成" }
    ]
  },

  { description: "SCC 80 萬/mL，乳蛋白率 4.1%，乳量 26 kg，牛群有炎症反應。",
    options:[
      { text:"進行乳房炎治療並改善環境清潔與通風", baseEffect:9, msg:"降低炎症。", correct:true, reason:"高 SCC+蛋白高常見於炎症" },
      { text:"提高精料", baseEffect:-3, msg:"錯誤。", correct:false, reason:"非能量問題" },
      { text:"減少粗料", baseEffect:-4, msg:"危險。", correct:false, reason:"粗料不能減" }
    ]
  },

  { description: "全場乳量 25 kg，乳糖 4.3%，乳脂率 3.0%，牛群食慾下降。",
    options:[
      { text:"檢查熱緊迫、環境舒適度與飲水供應", baseEffect:8, msg:"改善採食與乳量。", correct:true, reason:"食慾下降常來自熱緊迫或不舒適" },
      { text:"增加精料", baseEffect:-4, msg:"短期無效且風險高。", correct:false, reason:"非能量問題" },
      { text:"減少全群粗料", baseEffect:-6, msg:"會更糟。", correct:false, reason:"粗料需保持" }
    ]
  },

  { description: "MUN 10 mg/dL，乳蛋白率 2.9%，乳量 24 kg。",
    options:[
      { text:"增加蛋白質來源與日糧品質", baseEffect:8, msg:"改善蛋白率。", correct:true, reason:"低 MUN 代表蛋白不足" },
      { text:"減少精料", baseEffect:-4, msg:"更糟。", correct:false, reason:"需補蛋白" },
      { text:"限制飲水供應", baseEffect:-5, msg:"錯誤。", correct:false, reason:"飲水不能限制" }
    ]
  },

  { description: "檸檬酸 200 mg/dL，乳脂率 4.0%，乳量 30 kg，泌乳初期能量不足。",
    options:[
      { text:"增加高能量來源並改善過渡期管理", baseEffect:8, msg:"改善能量平衡。", correct:true, reason:"泌乳初期高產常能量不足" },
      { text:"提高精料以追求更高產量", baseEffect:-4, msg:"不安全。", correct:false, reason:"不應壓榨" },
      { text:"減少粗料", baseEffect:-5, msg:"危險。", correct:false, reason:"粗料不能減" }
    ]
  },

  { description: "SCC 40 萬/mL，乳量 27 kg，乳糖 4.4%，乳房炎病例增加。",
    options:[
      { text:"優先控制乳房炎，包括環境、擠乳衛生與治療", baseEffect:9, msg:"SCC 改善。", correct:true, reason:"SCC 高+病例多需強化衛生" },
      { text:"提高精料", baseEffect:-3, msg:"非根本原因。", correct:false, reason:"問題非能量" },
      { text:"減少水分供應", baseEffect:-5, msg:"危險。", correct:false, reason:"飲水不能減" }
    ]
  },

  { description: "游離脂肪酸 2.0 mmol/100g，乳量 26 kg，乳房炎病例增加，生乳保鮮程度被乳廠施壓。",
    options:[
      { text:"檢查搾乳衛生、冷卻設備與運送流程", baseEffect:9, msg:"改善保鮮與 FFA。", correct:true, reason:"高 FFA 多來自設備與冷卻不良" },
      { text:"提高精料", baseEffect:-4, msg:"無幫助。", correct:false, reason:"非能量問題" },
      { text:"減少粗料", baseEffect:-3, msg:"錯誤。", correct:false, reason:"粗料不能減" }
    ]
  },

  { description: "BHB 120 μmol/L，乳量 25 kg，乳脂率 3.2%，牛群體況偏瘦。",
    options:[
      { text:"增加能量供應、改善乾草品質並補充芳香劑", baseEffect:8, msg:"改善能量平衡。", correct:true, reason:"高 BHB 多為能量不足" },
      { text:"減少精料", baseEffect:-6, msg:"更糟。", correct:false, reason:"需要提升能量" },
      { text:"限制飲水", baseEffect:-3, msg:"錯誤且危險。", correct:false, reason:"飲水不能限制" }
    ]
  },

  { description: "全場乳量 32 kg，乳蛋白率 3.8%，乳脂率 3.9%，牛群有輕微熱緊迫。",
    options:[
      { text:"改善通風、加強灑水與遮陰，確保充足飲水", baseEffect:9, msg:"降低熱緊迫。", correct:true, reason:"夏季常見問題" },
      { text:"提高精料以維持產量", baseEffect:-4, msg:"風險高。", correct:false, reason:"精料不能用來處理熱緊迫" },
      { text:"減少活動空間", baseEffect:-2, msg:"無助。", correct:false, reason:"改善環境較重要" }
    ]
  }
);
</script>

<!-- 錯題表區塊 -->
<div id="wrongAnswersDiv" style="margin-top:20px;"></div>

</body>
</html>
