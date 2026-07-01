<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Claude Code Rollout — VisaData</title>
<script>
  // Set theme before first paint so there's no flash of the wrong theme.
  (function(){
    try{
      var prefersLight = window.matchMedia && window.matchMedia('(prefers-color-scheme: light)').matches;
      if(prefersLight){ document.documentElement.setAttribute('data-theme','light'); }
    }catch(e){}
  })();
</script>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=IBM+Plex+Mono:wght@400;500;600;700&family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
<style>
  :root{
    --ink:#0E1116;
    --panel:#161B24;
    --panel-raised:#1C2330;
    --hair:#2A3242;
    --text:#E7EBF2;
    --muted:#8B95A9;
    --muted-2:#5C6577;
    --signal:#5FE1A0;
    --signal-dim:#2C5E45;
    --warn:#F0A868;
    --warn-dim:#5A4326;
    --blue:#7CA9F0;
    --banner-border:#6B5330;
    --banner-text:#E6D6BC;
    --shadow:0 24px 60px -20px rgba(0,0,0,.6);
    --radius:10px;
    color-scheme:dark;
  }
  :root[data-theme="light"]{
    --ink:#F3F5F7;
    --panel:#FFFFFF;
    --panel-raised:#EDF1F4;
    --hair:#D7DEE6;
    --text:#151A22;
    --muted:#4C5560;
    --muted-2:#6B7480;
    --signal:#1C8F5E;
    --signal-dim:#C7EDDA;
    --warn:#B5691A;
    --warn-dim:#F5E3CC;
    --blue:#2E5FCB;
    --banner-border:#E0C393;
    --banner-text:#5A4620;
    --shadow:0 20px 50px -24px rgba(20,30,45,.18);
    color-scheme:light;
  }
  /* The terminal mockup and code blocks are meant to look like a real
     terminal window — they stay dark in both page themes, same as most
     editors keep a dark integrated terminal even in light mode. */
  .term, pre.code, .codetab{
    --panel:#161B24;
    --panel-raised:#1C2330;
    --hair:#2A3242;
    --text:#E7EBF2;
    --muted:#8B95A9;
    --muted-2:#5C6577;
    --signal:#5FE1A0;
    --signal-dim:#2C5E45;
    --warn:#F0A868;
    --warn-dim:#5A4326;
    --blue:#7CA9F0;
  }
  *{box-sizing:border-box;}
  html,body{margin:0;padding:0;background:var(--ink);transition:background .25s ease;}
  body{
    font-family:'Inter',sans-serif;
    color:var(--text);
    -webkit-font-smoothing:antialiased;
    line-height:1.55;
    transition:color .25s ease;
  }
  ::selection{background:var(--signal-dim);color:#fff;}
  a{color:var(--blue);}
  .mono{font-family:'IBM Plex Mono',monospace;}
  .wrap{max-width:920px;margin:0 auto;padding:0 24px;}
  .eyebrow{
    font-family:'IBM Plex Mono',monospace;
    text-transform:uppercase;
    letter-spacing:.14em;
    font-size:11.5px;
    color:var(--signal);
    display:flex;align-items:center;gap:8px;
  }
  .eyebrow::before{content:"";width:7px;height:7px;border-radius:50%;background:var(--signal);box-shadow:0 0 8px var(--signal);}

  /* ---------- HERO ---------- */
  .hero{padding:64px 0 40px;}
  .hero h1{
    font-size:clamp(30px,4.4vw,46px);
    line-height:1.08;
    margin:16px 0 10px;
    font-weight:800;
    letter-spacing:-0.01em;
  }
  .hero h1 .hl{color:var(--signal);}
  .hero p.sub{
    color:var(--muted);
    font-size:17px;
    max-width:58ch;
    margin:0 0 32px;
  }
  .chiprow{display:flex;flex-wrap:wrap;gap:8px;margin-bottom:36px;}
  .chip{
    font-family:'IBM Plex Mono',monospace;
    font-size:12.5px;
    color:var(--text);
    background:var(--panel-raised);
    border:1px solid var(--hair);
    padding:6px 12px;
    border-radius:999px;
  }
  .chip b{color:var(--warn);font-weight:600;}

  /* ---------- TERMINAL MOCKUP ---------- */
  .term{
    background:linear-gradient(180deg,#12161F,#0E1116);
    border:1px solid var(--hair);
    border-radius:12px;
    overflow:hidden;
    box-shadow:var(--shadow);
  }
  .term-bar{
    display:flex;align-items:center;gap:8px;
    padding:11px 14px;
    background:#171C26;
    border-bottom:1px solid var(--hair);
  }
  .term-bar .dot{width:10px;height:10px;border-radius:50%;}
  .term-bar .dot:nth-child(1){background:#F0716B;}
  .term-bar .dot:nth-child(2){background:#F0C06B;}
  .term-bar .dot:nth-child(3){background:#6BD08A;}
  .term-bar .label{
    margin-left:10px;
    font-family:'IBM Plex Mono',monospace;
    font-size:12px;
    color:var(--muted);
  }
  .term-body{padding:20px 22px 22px;font-family:'IBM Plex Mono',monospace;font-size:13.5px;}
  .term-line{color:var(--muted);margin-bottom:6px;}
  .term-line .prompt{color:var(--signal);}
  .term-line .cmd{color:var(--text);}
  .statusline{
    margin-top:14px;
    padding:12px 14px;
    background:var(--panel);
    border:1px solid var(--hair);
    border-radius:8px;
    display:flex;flex-wrap:wrap;gap:14px;align-items:center;
    font-size:13px;
  }
  .sl-item{display:flex;align-items:center;gap:6px;color:var(--text);}
  .sl-item .lab{color:var(--muted-2);}
  .sl-bar{width:70px;height:6px;background:#232B3A;border-radius:3px;overflow:hidden;display:inline-block;}
  .sl-bar i{display:block;height:100%;width:18%;background:var(--signal);}
  .cost{color:var(--warn);font-weight:600;}

  /* ---------- SECTION SHELL ---------- */
  section{padding:56px 0;border-top:1px solid var(--hair);}
  .stitle{
    font-size:26px;font-weight:800;letter-spacing:-.01em;margin:10px 0 8px;
  }
  .slede{color:var(--muted);max-width:64ch;margin:0 0 30px;font-size:15.5px;}

  /* ---------- IMPORTANT BANNER ---------- */
  .banner{
    display:flex;gap:14px;align-items:flex-start;
    background:var(--warn-dim);
    border:1px solid var(--banner-border);
    border-left:4px solid var(--warn);
    border-radius:8px;
    padding:16px 18px;
    margin-bottom:8px;
  }
  .banner .ico{font-size:18px;line-height:1;}
  .banner b{color:var(--warn);}
  .banner p{margin:4px 0 0;color:var(--banner-text);font-size:14.5px;}

  /* ---------- DAY TIMELINE ---------- */
  .timeline{position:relative;}
  .timeline::before{
    content:"";position:absolute;left:23px;top:6px;bottom:6px;width:1px;
    background:linear-gradient(var(--signal-dim),var(--hair));
  }
  .day{
    position:relative;
    display:grid;
    grid-template-columns:48px 1fr;
    gap:20px;
    margin-bottom:36px;
  }
  .day:last-child{margin-bottom:0;}
  .day-num{
    width:48px;height:48px;border-radius:50%;
    background:var(--panel-raised);
    border:1px solid var(--hair);
    display:flex;align-items:center;justify-content:center;
    font-family:'IBM Plex Mono',monospace;font-weight:700;font-size:16px;
    color:var(--signal);z-index:1;
  }
  .day-card{
    background:var(--panel);
    border:1px solid var(--hair);
    border-radius:var(--radius);
    padding:22px 24px;
  }
  .day-card h3{margin:0 0 4px;font-size:19px;font-weight:700;}
  .day-tag{
    font-family:'IBM Plex Mono',monospace;
    font-size:11px;color:var(--muted-2);text-transform:uppercase;letter-spacing:.08em;
    margin-bottom:10px;
  }
  .day-why{color:var(--muted);font-size:14.5px;margin:8px 0 16px;}
  .day-why b{color:var(--text);}
  .steps{margin:0;padding-left:0;list-style:none;}
  .steps li{
    display:flex;gap:10px;font-size:14px;color:var(--text);margin-bottom:8px;
  }
  .steps li::before{content:"→";color:var(--signal);flex-shrink:0;}
  pre.code{
    background:#0C0F14;
    border:1px solid var(--hair);
    border-radius:8px;
    padding:14px 16px;
    overflow-x:auto;
    font-family:'IBM Plex Mono',monospace;
    font-size:12.8px;
    color:#CFE8DA;
    margin:12px 0 0;
  }
  pre.code .c-mut{color:var(--muted-2);}
  pre.code .c-sig{color:var(--signal);}
  .codetabs{display:flex;gap:6px;margin:14px 0 0;}
  .codetab{
    font-family:'IBM Plex Mono',monospace;font-size:11px;
    background:var(--panel-raised);border:1px solid var(--hair);
    padding:4px 10px;border-radius:6px 6px 0 0;color:var(--muted);
  }
  .codetab.active{color:var(--signal);border-bottom-color:#0C0F14;background:#0C0F14;}

  /* ---------- 2-COL CARDS ---------- */
  .grid2{display:grid;grid-template-columns:1fr 1fr;gap:16px;}
  @media(max-width:720px){.grid2{grid-template-columns:1fr;}}
  .card{
    background:var(--panel);border:1px solid var(--hair);border-radius:var(--radius);
    padding:20px 22px;
  }
  .card .tag{
    display:inline-block;font-family:'IBM Plex Mono',monospace;font-size:11px;
    padding:3px 9px;border-radius:999px;margin-bottom:12px;
  }
  .tag.claudemd{background:var(--signal-dim);color:var(--signal);}
  .tag.auto{background:#28324A;color:var(--blue);}
  .card h4{margin:0 0 8px;font-size:16px;}
  .card p{color:var(--muted);font-size:14px;margin:0;}
  .card ul{margin:12px 0 0;padding-left:18px;color:var(--muted);font-size:13.5px;}
  .card ul li{margin-bottom:6px;}

  .no-plugin{
    display:flex;align-items:center;gap:12px;
    background:var(--panel-raised);border:1px solid var(--hair);
    border-radius:8px;padding:14px 18px;margin-top:20px;
  }
  .no-plugin .check{
    width:26px;height:26px;border-radius:50%;background:var(--signal-dim);
    display:flex;align-items:center;justify-content:center;color:var(--signal);font-weight:700;flex-shrink:0;
  }
  .no-plugin span{font-size:14px;color:var(--text);}
  .no-plugin code{color:var(--signal);}

  /* ---------- MCP DIAGRAM ---------- */
  .mcp-diagram{
    display:flex;align-items:center;justify-content:space-between;gap:10px;
    background:var(--panel);border:1px solid var(--hair);border-radius:var(--radius);
    padding:32px 24px;margin-top:8px;flex-wrap:wrap;
  }
  .node{
    flex:1;min-width:150px;text-align:center;
    background:var(--panel-raised);border:1px solid var(--hair);border-radius:10px;
    padding:18px 14px;
  }
  .node.center{border-color:var(--signal-dim);box-shadow:0 0 0 1px var(--signal-dim) inset;}
  .node .icon{font-size:22px;margin-bottom:8px;}
  .node .t{font-family:'IBM Plex Mono',monospace;font-size:13px;font-weight:600;color:var(--text);}
  .node .d{font-size:11.5px;color:var(--muted);margin-top:4px;}
  .arrow{color:var(--muted-2);font-family:'IBM Plex Mono',monospace;font-size:18px;padding:0 4px;}
  @media(max-width:640px){.mcp-diagram{flex-direction:column;}.arrow{transform:rotate(90deg);}}

  /* ---------- TABLE ---------- */
  table{width:100%;border-collapse:collapse;margin-top:8px;}
  th,td{text-align:left;padding:12px 14px;font-size:13.8px;border-bottom:1px solid var(--hair);}
  th{color:var(--muted-2);font-family:'IBM Plex Mono',monospace;font-size:11px;text-transform:uppercase;letter-spacing:.06em;font-weight:600;}
  td{color:var(--text);}
  td.pick{color:var(--signal);font-weight:600;}
  tr:last-child td{border-bottom:none;}

  /* ---------- FOOTER ---------- */
  footer{padding:48px 0 80px;color:var(--muted-2);font-size:13px;}
  footer .beyond{display:flex;flex-wrap:wrap;gap:8px;margin:14px 0 28px;}
  footer p{max-width:70ch;}
  footer a{color:var(--muted);}
  .divider{height:1px;background:var(--hair);margin:28px 0;}

  /* ---------- THEME TOGGLE ---------- */
  .theme-toggle{
    position:fixed;top:18px;right:18px;z-index:50;
    width:40px;height:40px;
    display:flex;align-items:center;justify-content:center;
    background:var(--panel-raised);
    border:1px solid var(--hair);
    border-radius:50%;
    font-size:16px;
    cursor:pointer;
    color:var(--text);
    box-shadow:var(--shadow);
    transition:background .2s ease,border-color .2s ease,transform .15s ease;
  }
  .theme-toggle:hover{transform:translateY(-1px);border-color:var(--signal);}
  .theme-toggle:active{transform:translateY(0);}
  @media(max-width:520px){
    .theme-toggle{top:12px;right:12px;width:36px;height:36px;font-size:14px;}
  }
</style>
</head>
<body>

<button class="theme-toggle" id="themeToggle" type="button" aria-label="Switch between light and dark theme" title="Switch theme">🌙</button>

<div class="wrap">

  <!-- HERO -->
  <div class="hero">
    <span class="eyebrow">Rollout plan · Prepared for daily 30-min syncs</span>
    <h1>Claude Code, from zero<br>to <span class="hl">shipped</span> — VisaData</h1>
    <p class="sub">A 5-day setup plan for a team starting from nothing: Windows machines, VS Code's integrated terminal, GCP Workstation, and two concrete integration asks — internal APIs and the internal database.</p>
    <div class="chiprow">
      <div class="chip">🪟 Windows-native, <b>not</b> WSL-first</div>
      <div class="chip">💻 VS Code integrated terminal</div>
      <div class="chip">☁️ GCP Workstation</div>
      <div class="chip">🧠 No memory plugin needed</div>
    </div>

    <div class="term">
      <div class="term-bar">
        <div class="dot"></div><div class="dot"></div><div class="dot"></div>
        <div class="label">PS C:\Users\visadata\workspace — VS Code integrated terminal</div>
      </div>
      <div class="term-body">
        <div class="term-line"><span class="prompt">PS&gt;</span> <span class="cmd">claude</span></div>
        <div class="term-line">Welcome to Claude Code. Loaded CLAUDE.md (visadata-api) · auto memory: on</div>
        <div class="statusline">
          <div class="sl-item"><span class="lab">model</span> Sonnet 4.6</div>
          <div class="sl-item"><span class="lab">dir</span> 📁 visadata-api</div>
          <div class="sl-item"><span class="lab">ctx</span> <span class="sl-bar"><i></i></span> 18%</div>
          <div class="sl-item"><span class="lab">session</span> <span class="cost">$0.42</span></div>
          <div class="sl-item"><span class="lab">today</span> <span class="cost">$3.85</span></div>
        </div>
      </div>
    </div>
  </div>

  <!-- IMPORTANT DETAIL -->
  <section style="padding-top:0;border-top:none;">
    <div class="banner">
      <div class="ico">🪟</div>
      <div>
        <b>Everything below assumes native Windows, run from VS Code's built-in terminal — no WSL.</b>
        <p>Claude Code has shipped first-class native Windows support (PowerShell tool + Git-for-Windows Bash tool). Install Git for Windows so Claude gets a real Bash tool; without it, Claude falls back to PowerShell only, which is fine but changes which scripts work. Either way — <b>do not</b> route this team through WSL setup. It adds a second filesystem, a second set of env vars, and a support burden you don't need for a first rollout.</p>
      </div>
    </div>
  </section>

  <!-- DAY PLAN -->
  <section>
    <span class="eyebrow">Week 1</span>
    <h2 class="stitle">Five 30-minute sessions</h2>
    <p class="slede">Same order as before, re-grounded for Windows + VS Code. Each day ends with something the team can see working before the call ends.</p>

    <div class="timeline">

      <div class="day">
        <div class="day-num">1</div>
        <div class="day-card">
          <div class="day-tag">Foundations</div>
          <h3>Install, verify Bash tool, ship the statusline</h3>
          <p class="day-why">Nothing else works until Claude Code can run from VS Code's terminal and reach the API through <b>GCP Workstation's</b> egress path. Cost visibility is the cheapest win to show value on day one — do it live.</p>
          <ul class="steps">
            <li>Install natively (winget or the PowerShell installer) — open VS Code's integrated terminal first, not a separate window, so the team gets used to working there</li>
            <li>Install Git for Windows if it's missing — unlocks Claude's Bash tool instead of PowerShell-only fallback</li>
            <li>Confirm outbound access from the Workstation (proxy / allowlist for <span class="mono">api.anthropic.com</span>) — same class of problem as any internal-network egress issue you've hit before</li>
            <li>Run <span class="mono">/statusline</span> inside Claude Code and describe what to show: model, cost, context %</li>
          </ul>
          <pre class="code"><span class="c-mut"># PowerShell — native install</span>
<span class="c-sig">irm https://claude.ai/install.ps1 | iex</span>

<span class="c-mut"># verify it's callable from the VS Code terminal, not just a standalone shell</span>
claude --version

<span class="c-mut"># inside a session</span>
/statusline show model, session cost, and context usage as a compact one-liner</pre>
        </div>
      </div>

      <div class="day">
        <div class="day-num">2</div>
        <div class="day-card">
          <div class="day-tag">Persistent context</div>
          <h3>CLAUDE.md — no plugin, it's built in</h3>
          <p class="day-why">This is the highest-leverage file in the whole rollout, and it costs nothing to set up. <b>Skip any "memory plugin" you see recommended online</b> — Claude Code has shipped this natively since v2.1.59.</p>
          <ul class="steps">
            <li>Run <span class="mono">/init</span> in the repo root — Claude interviews the team and drafts a first CLAUDE.md</li>
            <li>Keep it to facts, not tutorials: build commands, conventions, "always/never" rules, GCP Workstation quirks</li>
            <li>Commit it to git — it's a team asset, same as your <span class="mono">config.yaml</span> pattern</li>
            <li>Leave auto memory on (default) — it'll pick up corrections through the week without anyone touching a file</li>
          </ul>
          <pre class="code"><span class="c-mut">PS&gt;</span> claude
&gt; /init
<span class="c-mut"># Claude drafts CLAUDE.md from the codebase + a few questions</span>
&gt; /memory
<span class="c-mut"># browse what auto-memory has picked up so far — plain markdown, editable</span></pre>
          <div class="no-plugin">
            <div class="check">✓</div>
            <span>No install, no config: <code>CLAUDE.md</code> (you write it) + <code>MEMORY.md</code> (Claude writes it) ship in Claude Code itself.</span>
          </div>
        </div>
      </div>

      <div class="day">
        <div class="day-num">3</div>
        <div class="day-card">
          <div class="day-tag">Integration — internal API, part 1</div>
          <h3>Point Claude at the codebase, get an endpoint inventory</h3>
          <p class="day-why">Before wrapping anything, find out what actually exists. This is a plain codebase-analysis task — <b>it does not need Fable 5</b>, whatever they've got in Claude Code already (Sonnet 5, Opus 4.8) handles it. Model choice isn't the lever here.</p>
          <ul class="steps">
            <li>Run this inside the repo — no MCP servers needed yet, just Claude reading the code</li>
            <li>Ask for method, path, handler file, auth requirement, and request/response shape where inferable</li>
            <li>Have Claude flag anything it's <i>inferring</i> vs. reading directly — don't let a guess look like a fact</li>
            <li>Save the output as <span class="mono">docs/endpoint-inventory.md</span> and commit it — it's useful on its own, independent of MCP</li>
          </ul>
          <pre class="code"><span class="c-mut">PS&gt;</span> claude
&gt; Scan this repo and produce an inventory of every HTTP API endpoint it
  exposes. For each one, capture: method, path, handler function/file,
  required auth, request/response shape if inferable. Output as a
  markdown table, grouped by router/controller file. Flag anything
  you're inferring vs. reading directly.</pre>
          <div class="banner" style="margin-top:16px;">
            <div class="ico">⚖️</div>
            <div>
              <b>Frame this internally as code review, not "pointing an LLM at our codebase."</b>
              <p>Fable 5 was briefly pulled under a US export control directive in June — the government's stated concern was specifically about models reading a codebase to find flaws. That's not this: an endpoint inventory is read-only, defensive, everyday work, same category as static analysis. But at a bank, worth naming it accurately to compliance up front rather than letting the phrase "analyze our codebase with an LLM" travel on its own.</p>
            </div>
          </div>
        </div>
      </div>

      <div class="day">
        <div class="day-num">4</div>
        <div class="day-card">
          <div class="day-tag">Integration — internal API, part 2</div>
          <h3>Codegen the MCP server from the inventory</h3>
          <p class="day-why">Turn yesterday's table into working tools instead of hand-writing a wrapper. The inventory becomes the spec; Claude writes the server.</p>
          <ul class="steps">
            <li>Feed Claude the inventory and ask it to generate one MCP tool per endpoint, using the existing internal API client for auth/transport</li>
            <li>Register the result at <b>project scope</b> so it's git-committed and shared, not one dev's local config</li>
            <li>Verify with <span class="mono">/mcp</span> before trusting it in a real task</li>
          </ul>
          <pre class="code"><span class="c-mut">&gt;</span> Using docs/endpoint-inventory.md, generate an MCP server (stdio,
  Node) that exposes one tool per endpoint. Reuse our existing internal
  API client in src/lib/apiClient.ts for auth and requests — don't
  hand-roll a new HTTP layer.

<span class="c-mut"># from VS Code's terminal, once the server file exists</span>
claude mcp add --transport stdio internal-api `
  --scope project `
  -- node .\mcp-servers\internal-api\index.js

<span class="c-mut"># inside Claude Code</span>
/mcp   <span class="c-mut"># confirm "internal-api: connected"</span></pre>
          <p class="day-why" style="margin-top:14px;">Note the backtick line-continuation — that's PowerShell syntax, not Bash's <span class="mono">\</span>. Small thing, but it's exactly the kind of detail that stalls a first live demo.</p>
        </div>
      </div>


      <div class="day">
        <div class="day-num">5</div>
        <div class="day-card">
          <div class="day-tag">Org-level visibility</div>
          <h3>Cost tracking beyond one dev's statusline</h3>
          <p class="day-why">The statusline from day 1 is per-developer. What leadership actually asked for is aggregate spend — that needs OpenTelemetry pointed at whatever GCP Workstation already uses for observability.</p>
          <ul class="steps">
            <li>Enable telemetry export and point it at a collector reachable from the Workstation</li>
            <li>Confirm the <span class="mono">claude_code.cost.usage</span> metric is flowing, filterable by model and user</li>
            <li>Frame this to the team as governance, not surveillance — do it after days 1–4 have already built trust in the tool</li>
          </ul>
          <pre class="code"><span class="c-mut"># PowerShell — persist for the session</span>
$env:CLAUDE_CODE_ENABLE_TELEMETRY = "1"
$env:OTEL_METRICS_EXPORTER = "otlp"
$env:OTEL_EXPORTER_OTLP_ENDPOINT = "http://&lt;your-collector&gt;:4317"</pre>
        </div>
      </div>

    </div>
  </section>

  <!-- MEMORY EXPLAINER -->
  <section>
    <span class="eyebrow">Reference</span>
    <h2 class="stitle">Two memory layers, both built in</h2>
    <p class="slede">Worth 5 minutes on day 2 so nobody goes shopping for a third-party memory plugin.</p>
    <div class="grid2">
      <div class="card">
        <span class="tag claudemd">You write this</span>
        <h4>CLAUDE.md</h4>
        <p>Instructions loaded fresh at the start of every session. The place for anything a new teammate would need explained once.</p>
        <ul>
          <li>Build/test/lint commands</li>
          <li>Repo layout &amp; conventions</li>
          <li>"Always/never" rules</li>
          <li>Committed to git, shared by the whole team</li>
        </ul>
      </div>
      <div class="card">
        <span class="tag auto">Claude writes this</span>
        <h4>Auto memory (MEMORY.md)</h4>
        <p>Notes Claude saves itself from corrections and repeated patterns during a session. On by default since v2.1.59.</p>
        <ul>
          <li>Debugging insights, gotchas</li>
          <li>Personal, machine-local — not shared across the team automatically</li>
          <li>Review via <span class="mono">/memory</span>, edit or delete like any markdown file</li>
          <li>Toggle: <span class="mono">CLAUDE_CODE_DISABLE_AUTO_MEMORY=1</span></li>
        </ul>
      </div>
    </div>
  </section>

  <!-- MCP DIAGRAM -->
  <section>
    <span class="eyebrow">Reference</span>
    <h2 class="stitle">How the two integrations actually connect</h2>
    <p class="slede">Same protocol, two servers. This is the whole shape of days 3 and 4.</p>
    <div class="mcp-diagram">
      <div class="node">
        <div class="icon">🖥️</div>
        <div class="t">Claude Code</div>
        <div class="d">VS Code terminal<br>GCP Workstation</div>
      </div>
      <div class="arrow">↔</div>
      <div class="node center">
        <div class="icon">🔌</div>
        <div class="t">MCP servers</div>
        <div class="d">project-scoped<br>.mcp.json, committed</div>
      </div>
      <div class="arrow">↔</div>
      <div class="node">
        <div class="icon">🌐</div>
        <div class="t">Internal API</div>
        <div class="d">custom stdio server<br>wraps existing client</div>
      </div>
    </div>
    <div class="mcp-diagram" style="margin-top:14px;">
      <div class="node" style="flex:0;visibility:hidden;min-width:150px;"></div>
      <div class="arrow" style="visibility:hidden;">↔</div>
      <div class="node">
        <div class="icon">🗄️</div>
        <div class="t">Internal DB</div>
        <div class="d">Postgres MCP<br><b style="color:var(--warn)">read-only role</b></div>
      </div>
    </div>
  </section>

  <!-- DECISIONS -->
  <section>
    <span class="eyebrow">Decide early</span>
    <h2 class="stitle">Two calls not to let slide</h2>
    <table>
      <tr><th>Decision</th><th>Option A</th><th>Option B</th><th>Pick</th></tr>
      <tr>
        <td>MCP config scope</td>
        <td>User-level (per developer)</td>
        <td>Project-level, git-committed</td>
        <td class="pick">B — shared team asset</td>
      </tr>
      <tr>
        <td>DB credentials</td>
        <td>Full read/write role</td>
        <td>Scoped read-only role</td>
        <td class="pick">B — no write access until trust is earned</td>
      </tr>
      <tr>
        <td>Shell environment</td>
        <td>WSL2 + Bash</td>
        <td>Native Windows + Git-for-Windows Bash</td>
        <td class="pick">B — no second filesystem to support</td>
      </tr>
    </table>
  </section>

  <footer>
    <div class="divider"></div>
    <span class="eyebrow" style="color:var(--muted-2)">Beyond week 1</span>
    <div class="beyond">
      <div class="chip">Internal DB via Postgres MCP — <b>read-only role</b></div>
      <div class="chip">Slash commands for repeat workflows</div>
      <div class="chip">Custom subagents per domain</div>
      <div class="chip">Permission allowlist tuning</div>
      <div class="chip">Hooks for auto-format / lint</div>
    </div>
    <p><b style="color:var(--muted);">Sourcing note:</b> command syntax and feature availability checked against Anthropic's official docs (<a href="https://code.claude.com/docs/en/mcp" target="_blank">code.claude.com/docs/en/mcp</a>, <a href="https://code.claude.com/docs/en/memory" target="_blank">.../memory</a>, <a href="https://code.claude.com/docs/en/monitoring-usage" target="_blank">.../monitoring-usage</a>, <a href="https://code.claude.com/docs/en/quickstart" target="_blank">.../quickstart</a>). Verify exact flags against <span class="mono">claude --version</span> before the calls — this moves fast.</p>
  </footer>

</div>

<script>
  (function(){
    var root = document.documentElement;
    var btn = document.getElementById('themeToggle');

    function setIcon(){
      var isLight = root.getAttribute('data-theme') === 'light';
      btn.textContent = isLight ? '☀️' : '🌙';
    }
    setIcon();

    btn.addEventListener('click', function(){
      var isLight = root.getAttribute('data-theme') === 'light';
      if(isLight){
        root.removeAttribute('data-theme');
      }else{
        root.setAttribute('data-theme','light');
      }
      setIcon();
    });
  })();
</script>

</body>
</html>
