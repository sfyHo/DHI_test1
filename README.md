# DHI_test1
DHI game_test_20251202
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <title>DHI 飼養管理小遊戲</title>
  <style>
    body { font-family: Arial; padding: 20px; }
    button { padding: 10px; margin: 5px; }
    #game { max-width: 600px; }
  </style>
</head>
<body>
<div id="game">
  <h2>🐄 DHI 飼養管理小遊戲</h2>
  <p id="scenario"></p>
  <div id="options"></div>
  <p id="result"></p>
  <p>總收益：<span id="score">0</span></p>
  <button onclick="nextQuestion()">下一題</button>
</div>

<script>
const scenarios = [
  {
    description: "乳量下降 10%，體細胞上升至 380k。",
    options: [
      { text: "改善牛床乾燥度與墊料", effect: 8, msg: "體細胞下降，乳量回升！" },
      { text: "濃料比例提高 5%", effect: -3, msg: "乳量未改善，反而有亞臨床乳房炎風險。" },
      { text: "增加擠乳頻率到每日 3 次", effect: 4, msg: "乳量小幅上升。" }
    ]
  },
  {
    description: "泌乳初期（30 DIM）乳脂率僅 2.8%，疑似負能量平衡。",
    options: [
      { text: "提高乾物攝取量、改善日糧適口性", effect: 7, msg: "DMI 上升，乳脂正常化！" },
      { text: "減少飼料量以避免乳脂過高", effect: -4, msg: "問題更嚴重，能量不足！" }
    ]
  }
];

let current = 0;
let score = 0;

function loadQuestion() {
  const s = scenarios[current];
  document.getElementById("scenario").innerText = `情境：${s.description}`;
  document.getElementById("options").innerHTML = "";

  s.options.forEach((opt, idx) => {
    const btn = document.createElement("button");
    btn.textContent = opt.text;
    btn.onclick = () => choose(idx);
    document.getElementById("options").appendChild(btn);
  });
}

function choose(idx) {
  const s = scenarios[current];
  const opt = s.options[idx];

  score += opt.effect;
  document.getElementById("score").innerText = score;
  document.getElementById("result").innerText =
    `結果：${opt.msg}（收益 ${opt.effect > 0 ? "+" : ""}${opt.effect}）`;
}

function nextQuestion() {
  current++;
  document.getElementById("result").innerText = "";

  if (current >= scenarios.length) {
    document.getElementById("scenario").innerText = "遊戲結束！";
    document.getElementById("options").innerHTML = "";
    return;
  }
  loadQuestion();
}

loadQuestion();
</script>
</body>
</html>
