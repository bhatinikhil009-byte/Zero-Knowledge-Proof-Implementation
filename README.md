<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Zero-Knowledge Proof Implementation</title>
<style>
  :root{
    --ink:#0e1b2c;
    --paper:#ffffff;
    --accent:#3aa1a0;
    --good:#1c8b4b;
    --bad:#e2483d;
    --line:#e7e5df;
  }
  *{box-sizing:border-box;}
  body{
    margin:0;
    min-height:100vh;
    display:flex;
    align-items:center;
    justify-content:center;
    background:var(--paper);
    font-family:'Arial Black','Helvetica Neue',Arial,sans-serif;
    padding:24px;
  }
  .card{
    width:100%;
    max-width:520px;
  }
  h1{
    font-size:24px;
    line-height:1.15;
    text-transform:uppercase;
    letter-spacing:-0.5px;
    color:var(--ink);
    margin:0 0 8px 0;
  }
  .subtitle{
    font-family:Arial, sans-serif;
    font-size:13px;
    color:#777;
    margin:0 0 22px 0;
    line-height:1.5;
  }
  label{
    display:block;
    font-family:Arial, sans-serif;
    font-size:12px;
    font-weight:bold;
    color:var(--ink);
    margin:0 0 6px 4px;
    text-transform:uppercase;
    letter-spacing:0.5px;
  }
  input[type="number"], input[type="text"]{
    width:100%;
    padding:14px 20px;
    border-radius:999px;
    border:2px solid var(--line);
    font-family:Arial, sans-serif;
    font-size:15px;
    outline:none;
    margin-bottom:16px;
    transition:border-color .15s ease;
  }
  input:focus{border-color:var(--ink);}
  .row{
    display:flex;
    gap:12px;
  }
  .row > div{flex:1;}
  button{
    width:100%;
    padding:18px;
    border:none;
    border-radius:999px;
    background:var(--ink);
    color:#fff;
    font-family:'Arial Black','Helvetica Neue',Arial,sans-serif;
    font-size:15px;
    letter-spacing:1px;
    text-transform:uppercase;
    cursor:pointer;
    transition:transform .1s ease, opacity .15s ease;
  }
  button:hover{opacity:.92;}
  button:active{transform:scale(0.98);}

  .result{
    margin-top:24px;
    display:none;
  }
  .result.show{display:block;}

  .verdict-banner{
    display:flex;
    align-items:center;
    gap:10px;
    padding:14px 18px;
    border-radius:14px;
    font-family:Arial, sans-serif;
    font-weight:bold;
    font-size:14px;
    margin-bottom:18px;
  }
  .verdict-banner.pass{background:#e7f5ee;color:var(--good);}
  .verdict-banner.fail{background:#fbeae8;color:var(--bad);}

  .steps{
    list-style:none;
    padding:0;
    margin:0;
    font-family:'SFMono-Regular',Consolas,Menlo,monospace;
    font-size:12.5px;
    color:#222;
  }
  .steps li{
    padding:10px 14px;
    border-bottom:1px solid var(--line);
    word-break:break-all;
    line-height:1.6;
  }
  .steps li:last-child{border-bottom:none;}
  .steps .who{
    display:inline-block;
    font-family:Arial, sans-serif;
    font-weight:bold;
    font-size:10px;
    text-transform:uppercase;
    letter-spacing:0.5px;
    padding:2px 8px;
    border-radius:999px;
    margin-right:8px;
  }
  .who.prover{background:#e6eefc;color:#2255aa;}
  .who.verifier{background:#f4e9fb;color:#7a3bb0;}
  .who.public{background:#fff4dc;color:#a3730b;}

  .note{
    font-family:Arial, sans-serif;
    font-size:11px;
    color:#999;
    line-height:1.5;
    margin-top:16px;
    border-top:1px solid var(--line);
    padding-top:12px;
  }
</style>
</head>
<body>

<div class="card">
  <h1>Zero-Knowledge Proof<br>Implementation</h1>
  <p class="subtitle">
    Schnorr identification protocol: the Prover convinces the Verifier they know a secret
    number <em>x</em>, without ever revealing <em>x</em> itself.
  </p>

  <label for="secretInput">Prover's secret (x)</label>
  <input type="number" id="secretInput" value="42" min="1">

  <div class="row">
    <div>
      <label for="pInput">Prime modulus (p)</label>
      <input type="number" id="pInput" value="2147483647" min="5">
    </div>
    <div>
      <label for="gInput">Generator (g)</label>
      <input type="number" id="gInput" value="7" min="2">
    </div>
  </div>

  <button id="submitBtn">Submit</button>

  <div class="result" id="result">
    <div class="verdict-banner" id="verdictBanner"></div>
    <ul class="steps" id="steps"></ul>
    <div class="note">
      The Verifier never learns x — only that the Prover could correctly respond to a random
      challenge, which is only possible if they truly know the secret. Run it again and the
      random challenge/commitment will differ, but the proof still holds.
    </div>
  </div>
</div>

<script>
const secretInput = document.getElementById('secretInput');
const pInput = document.getElementById('pInput');
const gInput = document.getElementById('gInput');
const submitBtn = document.getElementById('submitBtn');
const result = document.getElementById('result');
const verdictBanner = document.getElementById('verdictBanner');
const stepsList = document.getElementById('steps');

function modPow(base, exp, mod) {
  base = ((base % mod) + mod) % mod;
  let result = 1n;
  while (exp > 0n) {
    if (exp & 1n) result = (result * base) % mod;
    exp >>= 1n;
    base = (base * base) % mod;
  }
  return result;
}

function randomBigIntBelow(max) {
  const r = BigInt(Math.floor(Math.random() * Number(max > 1000000n ? 1000000n : max)));
  return r === 0n ? 1n : r;
}

function addStep(who, whoClass, text) {
  const li = document.createElement('li');
  li.innerHTML = `<span class="who ${whoClass}">${who}</span>${text}`;
  stepsList.appendChild(li);
}

submitBtn.addEventListener('click', () => {
  stepsList.innerHTML = '';
  result.classList.add('show');

  try {
    const x = BigInt(secretInput.value);
    const p = BigInt(pInput.value);
    const g = BigInt(gInput.value);

    if (x <= 0n || p <= 2n || g <= 1n) throw new Error('Inputs must be positive, p > 2, g > 1');

    const y = modPow(g, x, p);
    addStep('Public', 'public', `Public parameters: g = ${g}, p = ${p}`);
    addStep('Prover', 'prover', `Computes public key y = g^x mod p = ${y} (x is kept secret)`);

    const k = randomBigIntBelow(p - 1n);
    const r = modPow(g, k, p);
    addStep('Prover', 'prover', `Picks random nonce k, sends commitment r = g^k mod p = ${r}`);

    const c = randomBigIntBelow(1000n);
    addStep('Verifier', 'verifier', `Sends random challenge c = ${c}`);

    const s = k + c * x;
    addStep('Prover', 'prover', `Computes response s = k + c·x = ${s} (reveals s, never x or k)`);

    const lhs = modPow(g, s, p);
    const rhs = (r * modPow(y, c, p)) % p;
    addStep('Verifier', 'verifier', `Checks g^s mod p (${lhs}) against r·y^c mod p (${rhs})`);

    const valid = lhs === rhs;
    verdictBanner.className = 'verdict-banner ' + (valid ? 'pass' : 'fail');
    verdictBanner.textContent = valid
      ? '✓ Proof accepted — Prover knows x, but x was never revealed'
      : '✗ Proof rejected — values did not match (try different inputs)';

  } catch (err) {
    verdictBanner.className = 'verdict-banner fail';
    verdictBanner.textContent = '✗ Error: ' + err.message;
  }
});
</script>

</body>
</html>
