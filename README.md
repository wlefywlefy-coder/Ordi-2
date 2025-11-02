# Ordi-2
<!doctype html>
<html lang="ar" dir="rtl">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>ORDI Smart Dashboard</title>
<style>
:root{--bg:#0f172a;--card:#111827;--fg:#e5e7eb;--muted:#94a3b8;--ok:#10b981;--bad:#ef4444;--warn:#f59e0b}
*{box-sizing:border-box} body{margin:0;background:var(--bg);color:var(--fg);font-family:system-ui,-apple-system,Segoe UI,Roboto,Inter,Arial}
.wrap{max-width:960px;margin:20px auto;padding:0 14px}
header{display:flex;gap:12px;align-items:center;justify-content:space-between;margin:8px 0 14px}
h1{margin:0;font-size:18px;font-weight:800} .btn{background:#1f2937;border:1px solid #334155;color:var(--fg);border-radius:10px;padding:8px 12px;cursor:pointer}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:12px}
.card{background:var(--card);border:1px solid #1f2937;border-radius:12px;padding:12px}
.k{font-size:12px;color:var(--muted);margin-bottom:6px} .v{font-size:22px;font-weight:800}
.small{font-size:12px;color:var(--muted);margin-top:6px} .decision{font-size:24px;font-weight:900}
.buy{color:var(--ok)} .wait{color:var(--warn)} .sell{color:var(--bad)}
.pill{display:inline-block;font-size:12px;color:var(--muted);border:1px dashed #334155;border-radius:999px;padding:2px 8px}
</style>
</head>
<body>
<div class="wrap">
  <header>
    <h1>ORDI Smart Dashboard</h1>
    <div style="display:flex;gap:8px;align-items:center">
      <button class="btn" id="refresh">تحديث</button>
      <button class="btn" id="auto">تلقائي: إيقاف</button>
      <span id="ts" class="pill">—</span>
    </div>
  </header>

  <div class="grid">
    <div class="card"><div class="k">سعر ORDI (Binance Spot)</div><div class="v" id="price">—</div></div>
    <div class="card"><div class="k">RSI(14) • إطار 30m (من klines)</div><div class="v" id="rsi">—</div></div>
    <div class="card"><div class="k">Funding Rate (Binance Perp)</div><div class="v" id="funding">—</div></div>
    <div class="card"><div class="k">Open Interest (Binance Perp)</div><div class="v" id="oi">—</div></div>
    <div class="card"><div class="k">BTC Dominance (CoinGecko)</div><div class="v" id="btcd">—</div></div>
    <div class="card"><div class="k">USDT Dominance (CoinGecko)</div><div class="v" id="usdtd">—</div></div>
    <div class="card" style="grid-column:1/-1">
      <div class="k">القرار</div>
      <div class="decision" id="decision">—</div>
      <div class="small" id="why">—</div>
    </div>
  </div>

  <div class="small" style="margin-top:10px">
    المصادر: Binance (Spot & Perps) + CoinGecko Global. قد تختلف BTC.D/USDT.D قليلًا عن TradingView لاختلاف المنهجية.
  </div>
</div>

<script>
const $=id=>document.getElementById(id);
const fmt=(n,d=4)=>Number(n).toLocaleString(undefined,{minimumFractionDigits:d,maximumFractionDigits:d});
const fmtPct=n=>fmt(n,2)+'%';

async function j(url){ const r=await fetch(url,{cache:'no-store'}); if(!r.ok) throw new Error(url+' -> '+r.status); return r.json(); }

// RSI(14) من أسعار الإغلاق
function rsi14(closes, p=14){
  if(!closes||closes.length<p+1) return NaN;
  let g=0,l=0;
  for(let i=1;i<=p;i++){ const ch=closes[i]-closes[i-1]; if(ch>0) g+=ch; else l+=-ch; }
  let ag=g/p, al=l/p;
  for(let i=p+1;i<closes.length;i++){
    const ch=closes[i]-closes[i-1];
    const cg=ch>0?ch:0, cl=ch<0?-ch:0;
    ag=(ag*(p-1)+cg)/p; al=(al*(p-1)+cl)/p;
  }
  if(al===0) return 100;
  const rs=ag/al; return 100 - (100/(1+rs));
}

async function load(){
  try{
    const [tick, kl, prem, oiJ, glob] = await Promise.all([
      j('https://api.binance.com/api/v3/ticker/price?symbol=ORDIUSDT'),
      j('https://api.binance.com/api/v3/klines?symbol=ORDIUSDT&interval=30m&limit=120'),
      j('https://fapi.binance.com/fapi/v1/premiumIndex?symbol=ORDIUSDT'),
      j('https://fapi.binance.com/fapi/v1/openInterest?symbol=ORDIUSDT'),
      j('https://api.coingecko.com/api/v3/global')
    ]);

    const price = Number(tick.price);
    const closes = kl.map(k=>Number(k[4]));
    const rsi = rsi14(closes.slice(-100));
    const funding = Number(prem.lastFundingRate)*100; // %
    const oi = Number(oiJ.openInterest); // عقود
    const btcd = Number(glob.data.market_cap_percentage.btc);
    const usdtd = Number(glob.data.market_cap_percentage.usdt);

    $('price').textContent = fmt(price,4)+' USDT';
    $('rsi').textContent = isFinite(rsi)? fmt(rsi,2):'—';
    $('funding').textContent = fmt(funding,4)+'%';
    $('oi').textContent = oi.toLocaleString();
    $('btcd').textContent = fmt(btcd,2)+'%';
    $('usdtd').textContent = fmt(usdtd,2)+'%';

    // قرار مبسّط وفق قواعدك (قابل للتعديل)
    let bull=0,bear=0, why=[];
    if(rsi<=42){ bull+=2; why.push('RSI منخفض'); }
    if(rsi>=65){ bear+=2; why.push('RSI مرتفع'); }
    if(funding<=0){ bull+=2; why.push('Funding سلبي/محايد'); }
    if(funding>=0.02){ bear+=2; why.push('Funding مرتفع'); }
    if(btcd<=59.9){ bull++; why.push('BTC.D منخفض'); }
    if(btcd>=60.1){ bear++; why.push('BTC.D مرتفع'); }
    if(usdtd<=4.95){ bull++; why.push('USDT.D منخفض'); }
    if(usdtd>=5.05){ bear++; why.push('USDT.D مرتفع'); }

    const delta = bull-bear;
    let dec='انتظار', cls='wait';
    if(delta>=2){ dec='شراء (سبوت تدريجي)'; cls='buy'; }
    else if(delta<=-2){ dec='تقليل تعرّض/تجنّب'; cls='sell'; }
    $('decision').className='decision '+cls;
    $('decision').textContent = dec;
    $('why').textContent = 'Bull '+bull+' / Bear '+bear+' • '+why.join(' • ');

    $('ts').textContent = 'تم التحديث: '+new Date().toLocaleString();
  }catch(e){
    $('ts').textContent = 'خطأ: '+e.message;
  }
}

$('refresh').addEventListener('click', load);
let t=null, on=false;
$('auto').addEventListener('click', ()=>{
  on=!on; $('auto').textContent = 'تلقائي: ' + (on?'تشغيل':'إيقاف');
  if(t) clearInterval(t);
  if(on){ load(); t=setInterval(load, 60*1000); }
});
load();
</script>
</body>
</html>
