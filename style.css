/* Milele Prime — Dashboard JS
   Bybit Futures Edition
   Fetches data from Flask proxy routes and updates the UI.
*/

'use strict';

// ── Config ───────────────────────────────────────────────────────────────────
const ENDPOINTS = {
  wallet:       '/proxy/bybit/wallet',
  positions:    '/proxy/bybit/positions',
  trades:       '/proxy/trades',
  openPos:      '/proxy/open_positions',
  stats:        '/proxy/stats',
};

// ── Helpers ──────────────────────────────────────────────────────────────────
const $  = id => document.getElementById(id);
const fmt = (n, dec=2) => n == null ? '--' : Number(n).toLocaleString('en-US', {minimumFractionDigits: dec, maximumFractionDigits: dec});
const fmtPnl = n => {
  if (n == null) return '--';
  const v = Number(n);
  return (v >= 0 ? '+' : '') + fmt(v);
};
const pnlClass = n => Number(n) >= 0 ? 'pos' : 'neg';

// ── State ─────────────────────────────────────────────────────────────────────
let allTrades = [];
let refreshTimer = null;

// ── Fetch all data ────────────────────────────────────────────────────────────
async function fetchAll() {
  const now = new Date().toLocaleTimeString();
  $('lastUpdated').textContent = `Updated ${now}`;

  await Promise.allSettled([
    fetchWallet(),
    fetchOpenPositions(),
    fetchTrades(),
    fetchStats(),
    fetchBybitStatus(),
  ]);
}

// ── Wallet (Bybit) ────────────────────────────────────────────────────────────
async function fetchWallet() {
  try {
    const resp = await fetch(ENDPOINTS.wallet);
    const data = await resp.json();
    const list = data?.result?.list || [];
    let usdt = null;
    const assets = [];

    for (const acct of list) {
      for (const coin of (acct.coin || [])) {
        const bal = parseFloat(coin.walletBalance || 0);
        if (bal > 0) {
          assets.push({ coin: coin.coin, bal });
          if (coin.coin === 'USDT') usdt = parseFloat(coin.availableToWithdraw || coin.walletBalance || 0);
        }
      }
    }

    $('usdtBalance').textContent = usdt != null ? `$${fmt(usdt)}` : '--';
    $('walletSub').textContent   = usdt != null ? 'Available in wallet' : 'Waiting for wallet response';

    const walletEl = $('walletAssets');
    $('walletAssetsCount').textContent = `${assets.length} assets`;
    if (assets.length === 0) {
      walletEl.innerHTML = '<span class="muted">No wallet data</span>';
      walletEl.classList.add('empty');
    } else {
      walletEl.classList.remove('empty');
      walletEl.innerHTML = assets.map(a =>
        `<div class="wallet-row"><span class="coin-tag">${a.coin}</span><span class="coin-bal">${fmt(a.bal, 4)}</span></div>`
      ).join('');
    }

    $('apiStateChip').textContent  = 'Bybit API OK';
    $('apiStateChip').className    = 'status-chip ok';
  } catch (e) {
    $('usdtBalance').textContent  = '--';
    $('apiStateChip').textContent = 'API Error';
    $('apiStateChip').className   = 'status-chip error';
  }
}

// ── Open Positions (bot internal) ─────────────────────────────────────────────
async function fetchOpenPositions() {
  try {
    const resp = await fetch(ENDPOINTS.openPos);
    const data = await resp.json();
    const el   = $('openPositions');
    $('openCount').textContent        = data.length;
    $('openPositionsPill').textContent = `${data.length} / 34`;

    if (!data.length) {
      el.innerHTML = 'No open positions';
      el.classList.add('empty');
      $('botStateChip').textContent = `Bot Live — 0 open`;
      $('botStateChip').className   = 'status-chip neutral';
      return;
    }

    el.classList.remove('empty');
    $('botStateChip').textContent = `Bot Live — ${data.length} open`;
    $('botStateChip').className   = 'status-chip ok';

    el.innerHTML = data.map(p => {
      const dir = p.direction === 'LONG' ? 'long' : 'short';
      return `
        <div class="pos-card ${dir}">
          <div class="pos-symbol">${p.symbol}</div>
          <div class="pos-dir ${dir}">${p.direction}</div>
          <div class="pos-row"><span>Entry</span><strong>${fmt(p.entry_price, 4)}</strong></div>
          <div class="pos-row"><span>SL</span><strong class="neg">${fmt(p.sl_price, 4)}</strong></div>
          <div class="pos-row"><span>TP</span><strong class="pos">${fmt(p.tp_price, 4)}</strong></div>
          <div class="pos-row"><span>Margin</span><strong>$${fmt(p.margin_usdt)}</strong></div>
          <div class="pos-row"><span>Duration</span><strong>${p.duration || '--'}</strong></div>
        </div>`;
    }).join('');
  } catch (e) {
    $('openPositions').innerHTML = '<span class="muted">Could not load positions</span>';
  }
}

// ── Trades ────────────────────────────────────────────────────────────────────
async function fetchTrades() {
  try {
    const resp = await fetch(ENDPOINTS.trades);
    allTrades  = await resp.json();
    renderTrades();
    updateSummary();
  } catch (e) {
    console.warn('Trades fetch failed:', e);
  }
}

function renderTrades() {
  const search   = $('tradeSearch').value.toUpperCase();
  const outcome  = $('outcomeFilter').value;
  const side     = $('sideFilter').value;
  const strategy = $('strategyFilter').value;

  const filtered = allTrades.filter(t => {
    if (search   && !t.symbol?.toUpperCase().includes(search))   return false;
    if (outcome  !== 'ALL' && t.outcome   !== outcome)           return false;
    if (side     !== 'ALL' && t.direction !== side)              return false;
    if (strategy !== 'ALL' && t.strategy  !== strategy)          return false;
    return true;
  });

  $('tradeCountPill').textContent = `${filtered.length} trades`;

  const tbody = $('tradeRows');
  if (!filtered.length) {
    tbody.innerHTML = '<tr><td colspan="10" class="muted center">No trades match filter</td></tr>';
    return;
  }

  tbody.innerHTML = [...filtered].reverse().map(t => {
    const cls  = t.outcome === 'WIN' ? 'win' : t.outcome === 'LOSS' ? 'loss' : 'neutral';
    const pnl  = parseFloat(t.pnl_usdt || 0);
    const pnlC = pnl >= 0 ? 'pos' : 'neg';
    return `<tr>
      <td><div>${t.open_time  || '--'}</div><div class="muted small">${t.close_time || '--'}</div></td>
      <td><strong>${t.symbol || '--'}</strong></td>
      <td class="muted small">${t.strategy || '--'}</td>
      <td><span class="dir-badge ${(t.direction||'').toLowerCase()}">${t.direction || '--'}</span></td>
      <td>${fmt(t.entry_price, 4)}</td>
      <td>
        <div class="muted small">SL: ${fmt(t.sl_price, 4)}</div>
        <div class="muted small">TP: ${fmt(t.tp_price, 4)}</div>
      </td>
      <td><span class="outcome-badge ${cls}">${t.outcome || '--'}</span></td>
      <td class="${pnlC}">${fmtPnl(t.pnl_usdt)}</td>
      <td class="muted">${fmt(t.fee_usdt, 4)}</td>
      <td class="${pnlC}">${t.pnl_pct != null ? fmtPnl(t.pnl_pct)+'%' : '--'}</td>
    </tr>`;
  }).join('');
}

function updateSummary() {
  const closed = allTrades.filter(t => t.outcome !== 'OPEN');
  const wins   = closed.filter(t => t.outcome === 'WIN');
  const losses = closed.filter(t => t.outcome === 'LOSS');
  const totalPnl = closed.reduce((s, t) => s + parseFloat(t.pnl_usdt || 0), 0);

  const todayUTC = new Date().toISOString().slice(0, 10);
  const todayTrades = closed.filter(t => (t.close_time || '').startsWith(todayUTC));
  const todayPnl = todayTrades.reduce((s, t) => s + parseFloat(t.pnl_usdt || 0), 0);

  const wr = closed.length ? (wins.length / closed.length * 100).toFixed(1) : '0.0';

  $('totalPnl').textContent  = fmtPnl(totalPnl);
  $('totalPnl').className    = `value ${pnlClass(totalPnl)}`;
  $('pnlSub').textContent    = 'Closed trades only';

  $('winRate').textContent   = `${wr}%`;
  $('winSub').textContent    = `${wins.length}W / ${losses.length}L`;

  $('todayPnl').textContent  = fmtPnl(todayPnl);
  $('todayPnl').className    = `value ${pnlClass(todayPnl)}`;

  $('totalTrades').textContent = closed.length;

  // S1 stats
  const s1 = closed.filter(t => t.strategy === 'S1_EMA_CROSS');
  const s1W = s1.filter(t => t.outcome === 'WIN').length;
  const s1L = s1.filter(t => t.outcome === 'LOSS').length;
  const s1Pnl = s1.reduce((a, t) => a + parseFloat(t.pnl_usdt || 0), 0);
  $('s1Trades').textContent  = s1.length;
  $('s1WinRate').textContent = s1.length ? `${(s1W/s1.length*100).toFixed(1)}%` : '0%';
  $('s1Pnl').textContent     = fmtPnl(s1Pnl);
  $('s1Pnl').className       = pnlClass(s1Pnl);
}

// ── Stats (consecutive losses) ────────────────────────────────────────────────
async function fetchStats() {
  try {
    const resp = await fetch(ENDPOINTS.stats);
    const data = await resp.json();
    $('s1Losses').textContent = data?.consec_losses?.S1 ?? 0;
  } catch (e) {}
}

// ── Bybit API status ──────────────────────────────────────────────────────────
async function fetchBybitStatus() {
  const ul = $('statusList');
  const items = [];

  // Wallet check
  try {
    const r = await fetch(ENDPOINTS.wallet);
    const d = await r.json();
    const ok = d?.retCode === 0 || d?.result?.list?.length > 0;
    items.push(`<li>Bybit account API: <strong class="${ok ? 'ok' : 'err'}">${ok ? 'Connected' : 'Error — check API keys'}</strong></li>`);
  } catch (e) {
    items.push(`<li>Bybit account API: <strong class="err">Error — ${e.message}</strong></li>`);
  }

  // Open positions
  try {
    const r = await fetch(ENDPOINTS.openPos);
    const d = await r.json();
    items.push(`<li>Open positions tracker: <strong>${d.length} active</strong></li>`);
  } catch (e) {
    items.push(`<li>Open positions tracker: <strong class="err">Error</strong></li>`);
  }

  // Trade history
  items.push(`<li>Closed trades loaded: <strong>${allTrades.filter(t=>t.outcome!=='OPEN').length}</strong></li>`);

  // Consecutive losses
  try {
    const r = await fetch(ENDPOINTS.stats);
    const d = await r.json();
    const cl = d?.consec_losses || {};
    items.push(`<li>Consecutive losses: <strong>S1=${cl.S1 ?? 0}</strong></li>`);
  } catch (e) {}

  ul.innerHTML = items.join('');
}

// ── Filters ───────────────────────────────────────────────────────────────────
['tradeSearch','outcomeFilter','sideFilter','strategyFilter'].forEach(id => {
  $(id).addEventListener('input', renderTrades);
});

// ── Refresh ───────────────────────────────────────────────────────────────────
$('refreshBtn').addEventListener('click', fetchAll);

$('refreshSelect').addEventListener('change', function () {
  clearInterval(refreshTimer);
  const sec = parseInt(this.value);
  if (sec > 0) refreshTimer = setInterval(fetchAll, sec * 1000);
});

// ── Init ──────────────────────────────────────────────────────────────────────
fetchAll();
const initSec = parseInt($('refreshSelect').value);
if (initSec > 0) refreshTimer = setInterval(fetchAll, initSec * 1000);
