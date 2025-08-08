<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Math Card Terminal</title>
  <style>
    :root{--bg:#0b1220;--panel:#0f1724;--text:#e6eef8;--muted:#9fb0c8;--accent:#7ee3a2}
    body{font-family:Menlo,Monaco,Consolas,monospace;background:var(--bg);color:var(--text);display:flex;align-items:center;justify-content:center;height:100vh;margin:0}
    .wrap{width:920px;max-width:95%;display:grid;grid-template-columns:1fr 420px;gap:18px}
    .card{background:linear-gradient(180deg,var(--panel),#0b1320);border-radius:8px;padding:18px;box-shadow:0 8px 24px rgba(2,6,23,.6)}
    .ui h1{margin:0 0 10px;font-size:18px}
    .question{font-size:14px;color:var(--muted);min-height:44px}
    .controls{display:flex;gap:8px;margin-top:12px}
    input[type="number"], input[type="text"], button{padding:8px 10px;border-radius:6px;border:1px solid rgba(255,255,255,.06);background:#071025;color:var(--text);outline:none}
    button{cursor:pointer}
    .terminal{background:#020615;border-radius:6px;padding:14px;height:480px;overflow:auto;font-size:13px;line-height:1.5;color:var(--text)}
    .term-input{display:flex;gap:8px;margin-top:8px}
    .term-input input{flex:1;padding:8px;border-radius:6px;border:1px solid rgba(255,255,255,.06);background:#071025;color:var(--text)}
    .muted{color:var(--muted);font-size:13px}
    .green{color:var(--accent)}
    .btn-ghost{background:transparent;border:1px solid rgba(255,255,255,.04)}
    @media (max-width:900px){.wrap{grid-template-columns:1fr}}
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card ui">
      <h1>Math Card Answer Checker</h1>
      <div class="question" id="question">Type <span class="muted">load &lt;number&gt;</span> in the terminal to open a card, or use the controls here.</div>

      <div class="controls">
        <input type="number" id="cardNumber" placeholder="Card #" />
        <button id="btnLoad">Load</button>
        <input type="text" id="playerAnswer" placeholder="Enter answer (for GUI)" />
        <button id="btnCheck">Check</button>
      </div>

      <div id="result" style="margin-top:12px;font-size:14px"></div>

      <hr style="margin:14px 0;border:0;border-top:1px solid rgba(255,255,255,.04)" />

      <div class="muted">Quick commands: <span class="green">load &lt;n&gt;</span>, <span class="green">check &lt;answer&gt;</span>, <span class="green">show</span>, <span class="green">list</span>, <span class="green">help</span></div>
    </div>

    <div class="card">
      <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:8px">
        <strong>Terminal</strong>
        <div class="muted">Type commands below — press Enter</div>
      </div>

      <div class="terminal" id="terminal" aria-live="polite"></div>

      <div class="term-input">
        <input id="termInput" autocomplete="off" placeholder="e.g. load 1    |    check -0.5    |    help" />
        <button id="termSend">Send</button>
      </div>
    </div>
  </div>

  <script>
    const correctAnswers = {
      1: {
        type: "Rational Equation",
        question: "1/(x+1) = 2. What is x?",
        answers: ["-0.5", "-1/2"]
      },
      2: {
        type: "Inequality",
        question: "2x - 3 > 5. What is the solution?",
        answers: ["x > 4"]
      },
      3: {
        type: "Function",
        question: "f(x) = x² + 1. Find f(3)",
        answers: ["10"]
      }
    };

    let currentCard = null;

    // UI helpers
    const terminal = document.getElementById('terminal');
    const questionEl = document.getElementById('question');
    const resultEl = document.getElementById('result');
    const cardNumberInput = document.getElementById('cardNumber');
    const playerAnswerInput = document.getElementById('playerAnswer');

    function appendTerm(text, type = 'out'){
      const el = document.createElement('div');
      el.innerHTML = text;
      terminal.appendChild(el);
      terminal.scrollTop = terminal.scrollHeight;
    }

    function showQuestion(card){
      const data = correctAnswers[card];
      if(!data){ questionEl.innerText = 'Card not found.'; return; }
      questionEl.innerText = `${data.type}: ${data.question}`;
      resultEl.innerText = '';
    }

    function loadQuestionNumber(card){
      const data = correctAnswers[card];
      if(!data){
        appendTerm(`<span class='muted'>Card ${card} not found.</span>`);
        currentCard = null;
        showQuestion(null);
        return;
      }
      currentCard = card;
      cardNumberInput.value = card;
      showQuestion(card);
      appendTerm(`<span class='green'>&gt;</span> Loaded card ${card}: <span class='muted'>${data.type}</span> ${data.question}`);
    }

    function checkAnswerText(answerText){
      if(!currentCard){ appendTerm("<span class='muted'>Please load a question first (use 'load &lt;n&gt;').</span>"); return; }
      const playerAnswer = String(answerText).trim().toLowerCase();
      const correctList = correctAnswers[currentCard].answers.map(a=>String(a).trim().toLowerCase());
      if(correctList.includes(playerAnswer)){
        appendTerm(`<span class='green'>✅ Correct! — ${answerText}</span>`);
        resultEl.innerText = '✅ Correct!';
      } else {
        appendTerm(`<span class='muted'>❌ Wrong — ${answerText}</span>`);
        resultEl.innerText = '❌ Wrong.';
      }
    }

    // Terminal command parser
    function runCommand(line){
      const raw = String(line||'').trim();
      if(!raw) return;
      appendTerm(`<span style="color:#7fb6ff">$</span> ${escapeHtml(raw)}`);

      const parts = raw.split(/\s+/);
      const cmd = parts[0].toLowerCase();
      const arg = parts.slice(1).join(' ');

      if(cmd === 'help'){
        appendTerm("<div class='muted'>Commands:\n - load &lt;number&gt;  : load a card\n - check &lt;answer&gt; : check an answer for the current card\n - show               : show current question\n - list               : list card numbers and types\n - clear              : clear terminal</div>");
        return;
      }

      if(cmd === 'load'){
        const n = parseInt(arg, 10);
        if(Number.isNaN(n)){
          appendTerm("<span class='muted'>Please provide a valid card number (example: load 1).</span>");
          return;
        }
        loadQuestionNumber(n);
        return;
      }

      if(cmd === 'check'){
        if(!arg){ appendTerm("<span class='muted'>Provide an answer to check. Example: check -0.5</span>"); return; }
        checkAnswerText(arg);
        return;
      }

      if(cmd === 'show'){
        if(!currentCard){ appendTerm("<span class='muted'>No card loaded.</span>"); return; }
        const d = correctAnswers[currentCard];
        appendTerm(`<span class='muted'>Card ${currentCard}: ${d.type} — ${d.question}</span>`);
        return;
      }

      if(cmd === 'list'){
        const lines = Object.keys(correctAnswers).map(k => `<div class='muted'>${k}: ${correctAnswers[k].type}</div>`).join('');
        appendTerm(lines);
        return;
      }

      if(cmd === 'clear'){
        terminal.innerHTML = '';
        return;
      }

      appendTerm("<span class='muted'>Unknown command. Type 'help' for available commands.</span>");
    }

    function escapeHtml(str){
      return String(str).replace(/[&<>\"']/g, function(m){return {'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":"&#39;"}[m];});
    }

    // Wire up UI controls
    document.getElementById('btnLoad').addEventListener('click', ()=>{
      const n = parseInt(cardNumberInput.value);
      if(Number.isNaN(n)) return;
      loadQuestionNumber(n);
    });

    document.getElementById('btnCheck').addEventListener('click', ()=>{
      const ans = playerAnswerInput.value;
      if(!ans) return;
      checkAnswerText(ans);
    });

    const termInput = document.getElementById('termInput');
    document.getElementById('termSend').addEventListener('click', ()=>{ runCommand(termInput.value); termInput.value=''; termInput.focus(); });
    termInput.addEventListener('keydown', (e)=>{ if(e.key === 'Enter'){ runCommand(termInput.value); termInput.value=''; } });

    // Seed welcome text
    appendTerm('<div class="muted">Welcome to Math Card Terminal. Type <span class="green">help</span> to begin.</div>');
    appendTerm('<div class="muted">Try: <span class="green">load 1</span> then <span class="green">check -0.5</span></div>');
  </script>
</body>
</html>
