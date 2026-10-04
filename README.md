const state = {
  symbol: 'BTCUSD',
  isPaused: false,
  feed: [],
  prices: [],
  lastPrice: 0,
  currentBid: 0,
  currentAsk: 0,
  openPrice: 0,
  previousClose: 0,
  lastUpdate: null,
  lastDirection: 'flat'
};

const currencyFormatter = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'USD',
  minimumFractionDigits: 2,
  maximumFractionDigits: 2
});

const numberFormatter = new Intl.NumberFormat('en-US', {
  minimumFractionDigits: 0,
  maximumFractionDigits: 0
});

const $ = (selector) => document.querySelector(selector);

const ui = {
  symbolLabel: $('#symbolLabel'),
  marketName: $('#marketName'),
  lastPrice: $('#lastPrice'),
  metricChange: $('#metricChange'),
  metricHigh: $('#metricHigh'),
  metricLow: $('#metricLow'),
  metricSpread: $('#metricSpread'),
  metricVolume: $('#metricVolume'),
  bidValue: $('#bidValue'),
  askValue: $('#askValue'),
  openValue: $('#openValue'),
  previousCloseValue: $('#previousCloseValue'),
  lastUpdatedValue: $('#lastUpdatedValue'),
  tickTableBody: $('#tickTableBody'),
  toggleFeed: $('#toggleFeed'),
  statusPill: $('#statusPill'),
  connectionStatus: $('#connectionStatus'),
  chart: $('#priceChart')
};

function seedData() {
  const base = 42680.12;
  const now = Date.now();

  for (let i = 0; i < 80; i += 1) {
    const variation = (Math.sin(i / 3) * 26) + (Math.random() * 42 - 21);
    const price = base + variation + i * 0.35;
    const bid = price - 1.6;
    const ask = price + 1.4;
    const volume = 17 + Math.floor(Math.random() * 90);

    state.feed.push({
      time: new Date(now - (80 - i) * 1000).toISOString(),
      price: Number(price.toFixed(2)),
      bid: Number(bid.toFixed(2)),
      ask: Number(ask.toFixed(2)),
      volume
    });
  }

  state.prices = state.feed.map((tick) => tick.price);
  state.lastPrice = state.feed[state.feed.length - 1].price;
  state.currentBid = state.feed[state.feed.length - 1].bid;
  state.currentAsk = state.feed[state.feed.length - 1].ask;
  state.openPrice = state.feed[0].price;
  state.previousClose = state.feed[Math.max(0, state.feed.length - 10)].price;
  state.lastUpdate = new Date();
}

function formatTime(date) {
  return new Date(date).toLocaleTimeString([], {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  });
}

function formatChange(current, previous) {
  const delta = ((current - previous) / previous) * 100;
  const prefix = delta >= 0 ? '+' : '';
  return `${prefix}${delta.toFixed(2)}%`;
}

function renderTable() {
  const rows = state.feed.slice(-8).reverse();

  ui.tickTableBody.innerHTML = rows
    .map((tick) => {
      const directionClass = tick.price >= state.previousClose ? 'price-up' : 'price-down';
      return `
        <tr>
          <td>${formatTime(tick.time)}</td>
          <td class="${directionClass}">${currencyFormatter.format(tick.price)}</td>
          <td>${currencyFormatter.format(tick.bid)}</td>
          <td>${currencyFormatter.format(tick.ask)}</td>
          <td>${numberFormatter.format(tick.volume)}</td>
        </tr>
      `;
    })
    .join('');
}

function renderMetrics() {
  const changePct = ((state.lastPrice - state.previousClose) / state.previousClose) * 100;
  const high = Math.max(...state.prices);
  const low = Math.min(...state.prices);
  const spread = state.currentAsk - state.currentBid;
  const volume = state.feed.reduce((sum, tick) => sum + tick.volume, 0);

  ui.lastPrice.textContent = currencyFormatter.format(state.lastPrice);
  ui.metricChange.textContent = `${changePct >= 0 ? '+' : ''}${changePct.toFixed(2)}%`;
  ui.metricChange.style.color = changePct >= 0 ? 'var(--green)' : 'var(--red)';
  ui.metricHigh.textContent = currencyFormatter.format(high);
  ui.metricLow.textContent = currencyFormatter.format(low);
  ui.metricSpread.textContent = currencyFormatter.format(spread);
  ui.metricVolume.textContent = numberFormatter.format(volume);
  ui.bidValue.textContent = currencyFormatter.format(state.currentBid);
  ui.askValue.textContent = currencyFormatter.format(state.currentAsk);
  ui.openValue.textContent = currencyFormatter.format(state.openPrice);
  ui.previousCloseValue.textContent = currencyFormatter.format(state.previousClose);
  ui.lastUpdatedValue.textContent = state.lastUpdate ? formatTime(state.lastUpdate) : '--:--:--';

  ui.lastPrice.style.color = changePct >= 0 ? 'var(--green)' : 'var(--red)';
}

function drawChart() {
  const canvas = ui.chart;
  const ctx = canvas.getContext('2d');
  const width = canvas.width;
  const height = canvas.height;
  const values = state.prices.slice(-45);

  const min = Math.min(...values);
  const max = Math.max(...values);
  const padding = 24;

  ctx.clearRect(0, 0, width, height);

  const gridColor = 'rgba(148, 163, 184, 0.2)';
  ctx.strokeStyle = gridColor;
  ctx.lineWidth = 1;

  for (let i = 0; i <= 4; i += 1) {
    const y = padding + ((height - padding * 2) / 4) * i;
    ctx.beginPath();
    ctx.moveTo(padding, y);
    ctx.lineTo(width - padding, y);
    ctx.stroke();
  }

  const gradient = ctx.createLinearGradient(0, 0, 0, height);
  gradient.addColorStop(0, 'rgba(103, 232, 249, 0.5)');
  gradient.addColorStop(1, 'rgba(103, 232, 249, 0.03)');

  ctx.beginPath();
  values.forEach((value, index) => {
    const x = padding + (index / Math.max(values.length - 1, 1)) * (width - padding * 2);
    const normalized = (value - min) / Math.max(max - min, 0.0001);
    const y = height - padding - normalized * (height - padding * 2);

    if (index === 0) {
      ctx.moveTo(x, y);
    } else {
      ctx.lineTo(x, y);
    }
  });

  ctx.lineTo(width - padding, height - padding);
  ctx.lineTo(padding, height - padding);
  ctx.closePath();
  ctx.fillStyle = gradient;
  ctx.fill();

  ctx.beginPath();
  values.forEach((value, index) => {
    const x = padding + (index / Math.max(values.length - 1, 1)) * (width - padding * 2);
    const normalized = (value - min) / Math.max(max - min, 0.0001);
    const y = height - padding - normalized * (height - padding * 2);

    if (index === 0) {
      ctx.moveTo(x, y);
    } else {
      ctx.lineTo(x, y);
    }
  });
  ctx.strokeStyle = 'rgba(103, 232, 249, 1)';
  ctx.lineWidth = 2.5;
  ctx.stroke();
}

function render() {
  renderMetrics();
  renderTable();
  drawChart();
}

function updateStatus() {
  const status = state.isPaused ? 'Paused' : 'Live';
  ui.connectionStatus.textContent = status;
  ui.statusPill.style.borderColor = state.isPaused ? 'rgba(251, 191, 36, 0.4)' : 'rgba(52, 211, 153, 0.4)';
  ui.statusPill.style.background = state.isPaused ? 'rgba(251, 191, 36, 0.08)' : 'rgba(52, 211, 153, 0.08)';
  ui.statusPill.style.color = state.isPaused ? '#fef3c7' : '#d1fae5';
  ui.statusPill.querySelector('.status-dot').style.background = state.isPaused ? 'var(--amber)' : 'var(--green)';
  ui.statusPill.querySelector('.status-dot').style.boxShadow = state.isPaused
    ? '0 0 12px rgba(251, 191, 36, 0.9)'
    : '0 0 12px rgba(52, 211, 153, 0.9)';
}

function pushTick(rawTick) {
  if (!rawTick || !rawTick.price) return;

  const tick = {
    time: rawTick.time || new Date().toISOString(),
    price: Number(rawTick.price),
    bid: Number(rawTick.bid ?? rawTick.price - 1.5),
    ask: Number(rawTick.ask ?? rawTick.price + 1.5),
    volume: Number(rawTick.volume ?? 0)
  };

  state.feed.push(tick);
  state.prices.push(tick.price);
  state.lastPrice = tick.price;
  state.currentBid = tick.bid;
  state.currentAsk = tick.ask;
  state.lastUpdate = new Date(tick.time);
  state.previousClose = state.feed[Math.max(0, state.feed.length - 11)].price;

  if (state.feed.length > 120) {
    state.feed.shift();
    state.prices.shift();
  }

  render();
}

function simulateFeed() {
  setInterval(() => {
    if (state.isPaused) return;

    const previous = state.lastPrice || 42680.12;
    const drift = (Math.random() - 0.5) * 18;
    const price = Number((previous + drift).toFixed(2));
    const bid = Number((price - (Math.random() * 4 + 1.2)).toFixed(2));
    const ask = Number((price + (Math.random() * 4 + 1.2)).toFixed(2));
    const volume = Math.floor(15 + Math.random() * 120);

    pushTick({
      time: new Date().toISOString(),
      price,
      bid,
      ask,
      volume
    });
  }, 1300);
}

function handleToggleFeed() {
  state.isPaused = !state.isPaused;
  updateStatus();
  ui.toggleFeed.textContent = state.isPaused ? 'Resume feed' : 'Pause feed';
}

function configureTickerFromWindow() {
  const globalSend = window.send;

  if (typeof globalSend === 'function') {
    window.__tickDashboardSend = globalSend;
  }
}

function init() {
  seedData();
  ui.symbolLabel.textContent = state.symbol;
  ui.marketName.textContent = 'Bitcoin / USD';
  ui.toggleFeed.addEventListener('click', handleToggleFeed);
  updateStatus();
  render();
  simulateFeed();
  configureTickerFromWindow();
}

window.addEventListener('load', init);
window.dashboard = {
  pushTick,
  setSymbol(symbol) {
    state.symbol = symbol;
    ui.symbolLabel.textContent = state.symbol;
  },
  toggle: handleToggleFeed
};

if (window.send && typeof window.send === 'object') {
  const originalSend = window.send;
  window.send = function sendProxy(payload) {
    if (payload && payload.ticks && Array.isArray(payload.ticks)) {
      payload.ticks.forEach(pushTick);
    }
    if (payload && payload.ticks_history && Array.isArray(payload.ticks_history)) {
      payload.ticks_history.forEach(pushTick);
    }
    return originalSend.call(this, payload);
  };
}
