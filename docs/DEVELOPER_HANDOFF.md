# Write with Us — Developer Handoff

## What it is

"Write with Us" is a customer-facing AI tool embedded on Quillingcard.com. It helps shoppers write personalized greeting card messages. It appears as a floating chat-style widget in the corner of the storefront — customers enter the occasion, recipient, and tone, and the tool generates three message options they can copy directly onto their card.

**Live test link:** https://writer-6vn.pages.dev/ — this is a standalone preview of the tool (not embedded in Shopify) for testing the full flow end to end.

## How it works (architecture)

There are three pieces, and they talk to each other in this order:

```
Shopper's browser (widget on Shopify storefront)
        │
        │  POST { prompt: "..." }
        ▼
Cloudflare Worker  (card-writer.drewgarber31.workers.dev)
        │
        │  Holds the OpenAI API key as an encrypted secret
        │  Forwards the request to OpenAI, adds the key server-side
        ▼
OpenAI API  (gpt-4o-mini)
        │
        │  Returns 3 generated message options
        ▼
Response flows back through the Worker to the widget, which displays it
```

**Why the Worker exists:** the widget's JavaScript runs in the customer's browser, which means anything in that code is publicly visible (view-source). If we called OpenAI directly from the widget, our API key would be exposed to anyone who looked at the page source. The Cloudflare Worker sits in between, keeps the key server-side as a secret, and is the only thing that's allowed to actually call OpenAI.

## Components / files

| File | Purpose | Where it lives |
|---|---|---|
| `quillingcard-writing-assistant.html` | The widget itself (HTML/CSS/JS) | Pasted into Shopify's `theme.liquid` |
| `cloudflare-worker.js` | Proxy script that holds the OpenAI key and forwards requests | Deployed as a Cloudflare Worker |
| `write-with-us-standalone.html` / `write-with-us-pages.zip` | A standalone version of the same widget on its own page, for sharing a preview link without touching Shopify | Deployed to Cloudflare Pages |

The Shopify widget and the standalone Pages version are two different front-ends that both call the **same** Cloudflare Worker as their backend.

## Current cost

- **OpenAI (gpt-4o-mini):** $0.15 / 1M input tokens, $0.60 / 1M output tokens. Each generation is roughly 400 tokens total — well under $0.001 per use.
- **Cloudflare Workers:** Free tier covers 100,000 requests/day. We are nowhere near that.
- Realistic total cost at current traffic: **$0–$2/month.**

---

## Part 1: Shopify implementation

1. Shopify admin → **Online Store → Themes**
2. On the live theme, click the **⋯** menu → **Edit code**
3. In the file list, open **Layout → theme.liquid**
4. Scroll to the closing `</body>` tag near the bottom of the file
5. Paste the full contents of `quillingcard-writing-assistant.html` directly above `</body>`
6. Click **Save**

The widget will now appear on every storefront page as a floating button in the bottom-right corner.

**Note on the "email me these messages" feature:** it submits through Shopify's native `/contact` form (`form_type=contact`) via a background `fetch` call, so messages get delivered to whatever email address is set under **Settings → Notifications** in the Shopify admin. No third-party form service is used.

---

## Part 2: Backend setup (Cloudflare Worker)

### 2.1 Create the Worker

1. Go to **dash.cloudflare.com** → sign in (or create a free account)
2. **Workers & Pages** → **Create application**
3. Choose **Start with Hello World!** (a plain JS template with a code editor — do **not** choose "Upload your static files," which skips the code editor and doesn't support secrets)
4. Name it (currently: `card-writer`)
5. Deploy the placeholder, then open the code editor and replace all of it with the contents of `cloudflare-worker.js`
6. **Save and Deploy**

### 2.2 Add the OpenAI API key as a secret

1. On the Worker's page, go to **Settings → Variables and Secrets**
2. **Add**:
   - Name: `OPENAI_API_KEY`
   - Value: the actual OpenAI API key (from platform.openai.com → API keys)
   - Type: **Secret** (encrypted — never store it as plaintext)
3. Save (redeploy if prompted)

### 2.3 CORS note (already fixed, but worth knowing)

The Worker must respond to the browser's CORS preflight (`OPTIONS`) request *before* it checks whether the request method is `POST`. If the OPTIONS check happens after the method check, preflight requests get rejected and the browser blocks the whole call with a CORS error. This is already handled correctly in the current version of `cloudflare-worker.js` — if the Worker code is ever rewritten, preserve that order.

### 2.4 Worker URL

Current production Worker: `https://card-writer.drewgarber31.workers.dev`

This is a POST-only endpoint. A plain GET/browser visit to that URL will correctly return `405 Method Not Allowed` — that's expected, not a bug.

---

## Part 3: Standalone preview link (optional, not part of Shopify)

For sharing a live demo link without touching the Shopify theme:

1. **Workers & Pages** → **Create application** → **Pages** tab → **Upload assets**
2. Upload `write-with-us-pages.zip` (contains `index.html` — the standalone page with the Worker URL already wired in)
3. Deploy — Cloudflare gives a `*.pages.dev` link
4. Current live link: `writer-6vn.pages.dev`

This page calls the same Cloudflare Worker as the Shopify widget, so both stay in sync automatically — no separate backend needed for the preview.

---

## Known limitations / next steps for engineering to consider

- **CORS is currently open (`Access-Control-Allow-Origin: *`)** — works fine for now, but should be locked to `https://quillingcard.com` (and the Pages preview domain, if kept) before/after wider rollout, to prevent other sites from calling our Worker.
- **No rate limiting** — currently nothing stops one visitor from generating unlimited messages. Given the low per-call cost this is a low-priority item, but worth adding (e.g. Cloudflare's built-in rate limiting rules) if abuse becomes a concern.
- **No analytics/logging** — there's no tracking yet on how often the widget is opened, how many messages are generated, or copy/email conversion. Recommend adding basic event tracking if this becomes a permanent feature.
- **Model choice** — currently using `gpt-4o-mini` for cost/speed. This can be swapped in `cloudflare-worker.js` by changing the `model` field in the request body, with no changes needed on the widget side.

---

## Appendix: Shopify widget code (`quillingcard-writing-assistant.html`)

This is the exact code pasted into `theme.liquid` above `</body>`. It calls the Cloudflare Worker at `https://card-writer.drewgarber31.workers.dev` — no API key is ever present in this file.

```html
<link href="https://fonts.googleapis.com/css2?family=Cardo:ital,wght@0,400;0,700;1,400&family=DM+Sans:wght@400;500;600;700&display=swap" rel="stylesheet">
<style>
  .qc-widget * { box-sizing: border-box; font-family: 'DM Sans', sans-serif; }
  .qc-widget { position: fixed; bottom: 24px; right: 24px; z-index: 9999; }
  .qc-bubble {
    background: #000; color: #fff; border: none; border-radius: 999px;
    padding: 14px 22px; font-size: 14px; letter-spacing: .03em; cursor: pointer;
    box-shadow: 0 8px 24px rgba(0,0,0,.35); display: flex; align-items: center; gap: 8px;
    transition: transform .15s ease;
  }
  .qc-bubble:hover { transform: translateY(-2px); }
  .qc-panel {
    display: none; position: absolute; bottom: 64px; right: 0; width: 360px; max-height: 560px;
    background: #fff; border-radius: 16px; box-shadow: 0 16px 48px rgba(44,45,46,.25);
    overflow: hidden; border: 1px solid #e5e0e3;
  }
  .qc-panel.open { display: flex; flex-direction: column; }
  .qc-head { background: #af95a6; color: #fff; padding: 18px 20px; }
  .qc-head-top { display: flex; justify-content: space-between; align-items: center; }
  .qc-head-title { font-family: 'Cardo', serif; font-size: 22px; }
  .qc-head-sub { font-size: 13px; color: #fff; opacity: .9; margin-top: 4px; }
  .qc-close { background: none; border: none; color: #fff; font-size: 20px; cursor: pointer; line-height: 1; }
  .qc-body { padding: 18px; overflow-y: auto; flex: 1; }
  .qc-section { background: #fff; border: 1px solid #e9e2d8; border-radius: 12px; padding: 16px; margin-bottom: 14px; }
  .qc-section-head { display: flex; align-items: center; gap: 10px; margin-bottom: 14px; cursor: default; }
  .qc-num { width: 22px; height: 22px; border-radius: 50%; background: #af95a6; color: #fff; font-size: 12px; font-weight: 700; display: flex; align-items: center; justify-content: center; flex-shrink: 0; }
  .qc-section-title { font-family: 'Cardo', serif; font-size: 17px; color: #2c2d2e; }
  .qc-section-title.small { font-size: 12px; text-transform: uppercase; letter-spacing: .08em; font-weight: 700; color: #2c2d2e; font-family: 'DM Sans', sans-serif; }
  .qc-optional { font-size: 11px; border: 1px solid #ddd; border-radius: 999px; padding: 2px 10px; color: #888; margin-left: auto; }
  .qc-label { font-size: 11px; text-transform: uppercase; letter-spacing: .08em; color: #af95a6; margin-bottom: 6px; display: block; font-weight: 700; }
  .qc-row { display: flex; gap: 10px; }
  .qc-row > div { flex: 1; }
  .qc-input, .qc-select, .qc-textarea {
    width: 100%; padding: 10px 12px; border: 1px solid #ddd7db; border-radius: 8px;
    background: #fff; font-size: 14px; margin-bottom: 14px; color: #2c2d2e;
  }
  .qc-textarea { resize: vertical; min-height: 56px; }
  .qc-btn {
    width: 100%; background: #2c2d2e; color: #fff; border: none; border-radius: 999px;
    padding: 14px; font-size: 14px; font-weight: 700; cursor: pointer; letter-spacing: .02em;
  }
  .qc-btn:disabled { opacity: .6; cursor: default; }
  .qc-btn-secondary { background: transparent; color: #af95a6; border: 1px solid #af95a6; margin-top: 8px; border-radius: 8px; }
  .qc-shop { display: block; text-align: center; text-decoration: none; background: #2c2d2e; color: #fff; margin-top: 10px; border-radius: 999px; padding: 14px; font-weight: 700; }
  .qc-result { background: #fff; border: 1px solid #e9e2d8; border-radius: 10px; padding: 14px; margin-bottom: 12px; }
  .qc-result-text { font-family: 'Cardo', serif; font-size: 16px; color: #2c2d2e; line-height: 1.5; margin-bottom: 10px; white-space: pre-wrap; }
  .qc-copy { font-size: 12px; color: #af95a6; background: none; border: none; cursor: pointer; font-weight: 700; padding: 0; }
  .qc-copy.copied { color: #2c2d2e; }
  .qc-loading { text-align: center; color: #af95a6; font-size: 13px; padding: 20px 0; }
  .qc-back { font-size: 12px; color: #af95a6; background: none; border: none; cursor: pointer; margin: 0 0 16px; padding: 0; text-decoration: underline; display: block; }

  @media (max-width: 480px) {
    .qc-widget { bottom: 16px; right: 16px; left: 16px; }
    .qc-bubble { width: 100%; justify-content: center; font-size: 13px; padding: 13px 18px; }
    .qc-panel { width: 100%; left: 0; right: 0; max-height: 75vh; }
    .qc-tooltip { width: 100%; left: 0; right: 0; box-sizing: border-box; }
    .qc-tooltip::after { right: 32px; }
  }
  .qc-tooltip {
    position: absolute; bottom: 68px; right: 0; width: 260px;
    background: #2c2d2e; color: #fff; border-radius: 14px; padding: 16px 18px;
    font-size: 14px; line-height: 1.5; box-shadow: 0 10px 30px rgba(0,0,0,.35);
    animation: qcPop .25s ease;
  }
  .qc-tooltip::after {
    content: ''; position: absolute; bottom: -10px; right: 28px;
    border-width: 10px 10px 0 10px; border-style: solid;
    border-color: #2c2d2e transparent transparent transparent;
  }
  .qc-tooltip-x {
    position: absolute; top: 8px; right: 10px; background: none; border: none;
    color: #aaa; font-size: 17px; cursor: pointer; line-height: 1;
  }
  @keyframes qcPop { from { opacity: 0; transform: translateY(6px); } to { opacity: 1; transform: translateY(0); } }
</style>

<div class="qc-widget">
  <div class="qc-panel" id="qcPanel">
    <div class="qc-head">
      <div class="qc-head-top">
        <span class="qc-head-title">Write with Us</span>
        <button class="qc-close" onclick="qcToggle(false)">×</button>
      </div>
      <div class="qc-head-sub" id="qcSubhead">Tell us about them — we'll write the rest.</div>
    </div>
    <div class="qc-body" id="qcBody">
      <!-- filled by JS -->
    </div>
  </div>
  <button class="qc-bubble" onclick="qcBubbleClick()">✦ Help me write this card</button>
  <div class="qc-tooltip" id="qcTooltip" style="display:none;">
    <button class="qc-tooltip-x" onclick="qcTooltipClose()">×</button>
    Stuck on what to write? Tap here and we'll help you write the perfect card message ✦
  </div>
</div>

<script>
(function(){
  const state = { occasion:'', recipient:'', tone:'Heartfelt', notes:'', results:[] };

  setTimeout(()=>{
    document.getElementById('qcTooltip').style.display = 'block';
  }, 2000);

  window.qcTooltipClose = function(){
    document.getElementById('qcTooltip').style.display = 'none';
  };

  window.qcToggle = function(open){
    document.getElementById('qcPanel').classList.toggle('open', open);
    qcTooltipClose();
    if(open) renderForm();
  };

  window.qcBubbleClick = function(){
    const isOpen = document.getElementById('qcPanel').classList.contains('open');
    qcToggle(!isOpen);
  };

  window.renderForm = function(){
    document.getElementById('qcSubhead').textContent = "Tell us about them — we'll write the rest.";
    document.getElementById('qcBody').innerHTML = `
      <div class="qc-section">
        <div class="qc-section-head">
          <span class="qc-num">1</span>
          <span class="qc-section-title">Make it personal</span>
        </div>

        <label class="qc-label">Occasion</label>
        <input class="qc-input" id="qcOccasion" placeholder="Birthday, anniversary, thank you..." value="${state.occasion}">

        <label class="qc-label">Who's it for?</label>
        <input class="qc-input" id="qcRecipient" placeholder="My mom, a close friend, my boss..." value="${state.recipient}">

        <label class="qc-label">Tone</label>
        <select class="qc-select" id="qcTone">
          ${['Heartfelt','Playful & funny','Elegant & poetic','Simple & warm'].map(t =>
            `<option ${t===state.tone?'selected':''}>${t}</option>`).join('')}
        </select>

        <label class="qc-label">Anything to include? (optional)</label>
        <textarea class="qc-textarea" id="qcNotes" placeholder="An inside joke, a memory, why you're grateful...">${state.notes}</textarea>
      </div>

      <div class="qc-section">
        <div class="qc-section-head">
          <span class="qc-num">2</span>
          <span class="qc-section-title small">Quick Write</span>
          <span class="qc-optional">Optional</span>
        </div>
        <div class="qc-row">
          <div>
            <label class="qc-label">Occasion</label>
            <select class="qc-select" id="qcQuickOccasion" onchange="qcQuickFill()">
              <option value="">Pick one</option>
              ${['Birthday','Anniversary','Thank You','Congratulations','Sympathy','Just Because','Wedding','New Baby'].map(o=>`<option>${o}</option>`).join('')}
            </select>
          </div>
          <div>
            <label class="qc-label">Recipient</label>
            <select class="qc-select" id="qcQuickRecipient" onchange="qcQuickFill()">
              <option value="">Pick one</option>
              ${['Best Friend','Mom','Dad','Partner','Sibling','Coworker','Boss','Grandparent'].map(r=>`<option>${r}</option>`).join('')}
            </select>
          </div>
        </div>
      </div>

      <button class="qc-btn" onclick="qcGenerate()">Write My Message ✦</button>
    `;
  }

  window.qcQuickFill = function(){
    const o = document.getElementById('qcQuickOccasion').value;
    const r = document.getElementById('qcQuickRecipient').value;
    if(o) document.getElementById('qcOccasion').value = o;
    if(r) document.getElementById('qcRecipient').value = r;
  };

  function renderLoading(){
    document.getElementById('qcSubhead').textContent = "One moment...";
    document.getElementById('qcBody').innerHTML = `<div class="qc-loading" id="qcLoadingText">Reading your details...</div>`;
    const steps = ["Reading your details...", "Finding the right words...", "Polishing your message..."];
    let i = 0;
    const el = document.getElementById('qcLoadingText');
    const interval = setInterval(()=>{
      i++;
      if(i < steps.length && el) el.textContent = steps[i];
      else clearInterval(interval);
    }, 700);
  }

  function renderResults(){
    document.getElementById('qcSubhead').textContent = "Here's what we came up with.";
    document.getElementById('qcBody').innerHTML = `
      <button class="qc-back" onclick="renderForm()">← Edit details</button>
      ${state.results.map((r,i) => `
        <div class="qc-result">
          <div class="qc-result-text">${r}</div>
          <button class="qc-copy" onclick="qcCopy(this, ${i})">Copy this message</button>
        </div>
      `).join('')}
      <button class="qc-btn qc-btn-secondary" onclick="qcGenerate()">Give me new options</button>

      <div class="qc-section" style="margin-top:14px;">
        <label class="qc-label">Get these emailed to your inbox</label>
        <input class="qc-input" id="qcEmail" type="email" placeholder="you@example.com">
        <button class="qc-btn qc-btn-secondary" id="qcEmailBtn" onclick="qcEmailResults()">Email Me These Messages</button>
        <div id="qcEmailStatus" style="font-size:12px;color:#af95a6;margin-top:6px;"></div>
      </div>
    `;
  }

  window.qcEmailResults = async function(){
    const email = document.getElementById('qcEmail').value.trim();
    const statusEl = document.getElementById('qcEmailStatus');
    if(!email){ document.getElementById('qcEmail').focus(); return; }

    const btn = document.getElementById('qcEmailBtn');
    btn.disabled = true;
    statusEl.textContent = 'Sending...';

    const body = `Occasion: ${state.occasion}\nRecipient: ${state.recipient}\nTone: ${state.tone}\n\n` +
      state.results.map((r,i)=>`Option ${i+1}:\n${r}`).join('\n\n');

    const formData = new URLSearchParams();
    formData.append('form_type', 'contact');
    formData.append('utf8', '✓');
    formData.append('contact[email]', email);
    formData.append('contact[body]', body);

    try {
      await fetch('/contact#ContactForm', {
        method: 'POST',
        mode: 'no-cors',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: formData.toString()
      });
      statusEl.textContent = 'Sent ✓ Check your inbox soon.';
    } catch(e){
      statusEl.textContent = "Something went wrong — try again.";
    }
    btn.disabled = false;
  };

  window.qcCopy = function(btn, i){
    navigator.clipboard.writeText(state.results[i]);
    btn.textContent = 'Copied ✓';
    btn.classList.add('copied');
    setTimeout(()=>{ btn.textContent = 'Copy this message'; btn.classList.remove('copied'); }, 1800);
  };

  window.qcGenerate = async function(){
    state.occasion = document.getElementById('qcOccasion')?.value || state.occasion;
    state.recipient = document.getElementById('qcRecipient')?.value || state.recipient;
    state.tone = document.getElementById('qcTone')?.value || state.tone;
    state.notes = document.getElementById('qcNotes')?.value || state.notes;

    renderLoading();

    const prompt = `Write 3 short handwritten-style greeting card messages for a Quilling Card (an artisanal, elegant paper-quilled card brand).
Occasion: ${state.occasion || 'a special occasion'}
Recipient: ${state.recipient || 'someone special'}
Tone: ${state.tone}
Details to weave in: ${state.notes || 'none given'}

Rules: each message 2-4 sentences, sounds genuinely personal (not generic), no em dashes, no cheesy greeting-card clichés. Return ONLY valid JSON: an array of 3 strings, nothing else.`;

    try{
      const response = await fetch("https://card-writer.drewgarber31.workers.dev", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ prompt })
      });
      const data = await response.json();
      let raw = data.choices?.[0]?.message?.content?.trim() || '[]';
      raw = raw.replace(/```json|```/g,'').trim();
      state.results = JSON.parse(raw);
    } catch(e){
      state.results = ["We couldn't write that just now — tap 'Give me new options' to try again."];
    }
    renderResults();
  };
})();
</script>

```
