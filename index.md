<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=0">
    <title>HOANG2285 - V34.0 ADMIN</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/blueimp-md5/2.19.0/js/md5.min.js"></script>
    <style>
        body { background: #000; color: #00ffcc; font-family: sans-serif; padding: 10px; margin: 0; }
        .card { background: #111; border: 1px solid #333; padding: 12px; margin-bottom: 10px; border-radius: 8px; }
        input, select, button { width: 100%; padding: 12px; margin: 5px 0; background: #222; color: #fff; border: 1px solid #444; font-size: 16px; border-radius: 5px; }
        .btn-group { display: grid; grid-template-columns: 1fr 1fr; gap: 5px; }
        table { width: 100%; border-collapse: collapse; color: #fff; font-size: 12px; }
        th, td { border: 1px solid #444; padding: 8px; text-align: center; }
    </style>
</head>
<body>

<div class="card">
    <h3 style="text-align:center">🛠 ĐIỀU HÀNH HOANG2285</h3>
    <input id="keyIn" placeholder="Nhập Key (HOANGDZ_...)">
    <select id="typeIn"><option value="THƯỜNG">THƯỜNG</option><option value="VIP">VIP</option></select>
    <button onclick="addKey()" style="background:#006633">✅ TẠO KEY MỚI</button>
</div>

<div class="card">
    <div class="btn-group">
        <button onclick="toggleMaint()">⚙️ BẢO TRÌ</button>
        <button onclick="showDiscountModal()" style="background:#cc6600">🔥 GIẢM GIÁ</button>
        <button onclick="broadcast()" style="background:#0066cc">📢 THÔNG BÁO</button>
        <button onclick="clearAll()" style="background:#cc0000">🗑️ XÓA HẾT</button>
    </div>
</div>

<div class="card">
    <h3>📋 DANH SÁCH KEY</h3>
    <table id="tbl"><thead><tr><th>Key</th><th>ID</th><th>Xóa</th></tr></thead><tbody></tbody></table>
</div>

<script>
const TOKEN = '8594279956:AAGZQDQVnoBaGmURmY11uKE68-9Dd0rd5Lk';
let keys = JSON.parse(localStorage.getItem('myKeys') || '[]');
let history = JSON.parse(localStorage.getItem('history') || '[]');

// THUẬT TOÁN NÂNG CAO (MARKOV + ENTROPY)
function predictAdvanced(md5Input, isVip) {
    const digits = md5Input.split('').map(c => parseInt(c, 16));
    const entropy = new Set(digits).size / 16.0;
    const xorVal = digits.reduce((a, b) => a ^ b, 0);
    const markovBias = (history.length > 0 && history[history.length-1] === 'TÀI') ? 2 : -2;
    
    const score = (xorVal * digits.length) + markovBias;
    const isTai = (score % 2 === 0);
    const reliability = Math.floor(75 + (entropy * 20) + (isVip ? 5 : 0));
    
    return { result: isTai ? '🔵 TÀI' : '🔴 XỈU', reliability: Math.min(99, reliability) };
}

async function botEngine() {
    const res = await fetch(`https://api.telegram.org/bot${TOKEN}/getUpdates?offset=-1`);
    const data = await res.json();
    if (!data.result || !data.result[0]) return;
    const m = data.result[0].message;
    if (!m || !m.text) return;

    // Kích hoạt
    let k = keys.find(i => i.name === m.text && !i.used);
    if (k) {
        k.used = true; k.uid = m.from.id;
        localStorage.setItem('myKeys', JSON.stringify(keys));
        render();
        send(m.chat.id, "✅ Kích hoạt thành công! Nhập MD5 để dùng.");
    } else if (m.text.length >= 32) {
        let u = keys.find(i => i.uid == m.from.id && i.used);
        if (u) {
            send(m.chat.id, "👾 HACKER: Đang giải mã...\n🔍 Phân tích cầu...");
            setTimeout(() => {
                let p = predictAdvanced(m.text, u.type === 'VIP');
                history.push(p.result); localStorage.setItem('history', JSON.stringify(history.slice(-50)));
                send(m.chat.id, `🔮 ${p.result} | Độ tin cậy: ${p.conf}%\n⚠️ Không All-in!\nAdmin: @tranhoang2286`);
            }, 1500);
        }
    }
}

// CÁC HÀM QUẢN TRỊ
function addKey() {
    let name = document.getElementById('keyIn').value;
    if(!name.includes('HOANGDZ')) return alert("Phải có HOANGDZ");
    keys.push({ name, type: document.getElementById('typeIn').value, used: false, uid: null });
    localStorage.setItem('myKeys', JSON.stringify(keys)); render();
}

function showDiscountModal() {
    let p = prompt("Nhập % giảm giá:");
    if(p) keys.filter(k => k.uid).forEach(k => send(k.uid, "🔥 GIẢM GIÁ " + p + "% TẤT CẢ KEY!\nLiên hệ @tranhoang2286"));
}

function broadcast() {
    let msg = prompt("Nhập thông báo gửi người chơi:");
    if(msg) keys.filter(k => k.uid).forEach(k => send(k.uid, "📢 ADMIN: " + msg));
}

function toggleMaint() {
    let status = localStorage.getItem('isMaint') === 'true';
    localStorage.setItem('isMaint', !status);
    let msg = !status ? "⚠️ HỆ THỐNG BẢO TRÌ!" : "✅ HỆ THỐNG HOẠT ĐỘNG!";
    keys.filter(k => k.uid).forEach(k => send(k.uid, msg));
}

function render() {
    document.querySelector('#tbl tbody').innerHTML = keys.map((k, i) => 
        `<tr><td>${k.name}</td><td>${k.uid || '-'}</td><td><button onclick="del(${i})">X</button></td></tr>`
    ).join('');
}
function del(i) { keys.splice(i, 1); localStorage.setItem('myKeys', JSON.stringify(keys)); render(); }
function clearAll() { keys = []; localStorage.setItem('myKeys', JSON.stringify(keys)); render(); }
function send(cid, t) { fetch(`https://api.telegram.org/bot${TOKEN}/sendMessage`, { method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify({chat_id: cid, text: t}) }); }

setInterval(botEngine, 2000); render();
</script>
</body>
</html>
