<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Fitness Dashboard Pro</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/html2pdf.js/0.10.1/html2pdf.bundle.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root {
            --bg: #1a1a2e; --card-bg: #16213e; --text: #ffffff;
            --input-bg: #0f3460; --border: #2e2e48;
            --primary: #00d2ff; --secondary: #3a7bd5;
            --protein: #ff4d4d; --fats: #ffcc00; --carbs: #4dff88; --water: #00a8ff;
        }

        [data-theme="light"] {
            --bg: #f0f2f5; --card-bg: #ffffff; --text: #1a1a2e;
            --input-bg: #f8f9fa; --border: #dee2e6; --primary: #3a7bd5;
        }

        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); color: var(--text); display: flex; justify-content: center; padding: 20px; transition: 0.3s; }
        .container { background: var(--card-bg); padding: 2rem; border-radius: 20px; box-shadow: 0 15px 35px rgba(0,0,0,0.2); width: 100%; max-width: 500px; position: relative; }
        .theme-toggle { position: absolute; top: 20px; right: 20px; cursor: pointer; background: none; border: none; color: var(--primary); font-size: 1.2rem; }
        h1 { text-align: center; color: var(--primary); margin: 0 0 20px 0; }
        
        .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-bottom: 10px; }
        label { display: block; margin-bottom: 5px; font-size: 0.75rem; color: #888; text-transform: uppercase; font-weight: bold; }
        input, select { width: 100%; padding: 12px; border: 2px solid var(--border); border-radius: 10px; background: var(--input-bg); color: var(--text); font-size: 1rem; box-sizing: border-box; }
        
        button { cursor: pointer; font-weight: bold; transition: 0.2s; border-radius: 10px; border: none; }
        .main-btn { width: 100%; padding: 15px; background: linear-gradient(to right, var(--primary), var(--secondary)); color: white; margin-top: 10px; }
        .btn-clear { background: transparent; color: #e94560; border: 1px solid #e94560; padding: 5px 15px; font-size: 0.8rem; margin-top: 20px; }

        #result { margin-top: 20px; padding: 15px; background: rgba(125,125,125,0.05); border-radius: 15px; display: none; }
        .macro-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin: 15px 0; text-align: center; }
        .macro-box { padding: 10px; border-radius: 12px; border: 2px solid transparent; font-size: 0.8rem; }
        .p-box { border-color: var(--protein); color: var(--protein); }
        .f-box { border-color: var(--fats); color: var(--fats); }
        .c-box { border-color: var(--carbs); color: var(--carbs); }
        
        .chart-container { margin-top: 25px; height: 200px; }
        .water-controls { display: flex; align-items: center; justify-content: space-between; margin-top: 15px; padding-top: 15px; border-top: 1px solid var(--border); }
    </style>
</head>
<body>

<div class="container" id="capture-area">
    <button class="theme-toggle" onclick="toggleTheme()" id="theme-icon">🌙</button>
    <h1>Fitness Dash</h1>
    
    <div class="grid-2">
        <div><label>Gender</label><select id="gender"><option value="male">Male</option><option value="female">Female</option></select></div>
        <div><label>Age</label><input type="number" id="age" placeholder="Age"></div>
    </div>
    <div class="grid-2">
        <div><label>Weight (kg)</label><input type="number" id="weight" placeholder="kg"></div>
        <div><label>Height (cm)</label><input type="number" id="height" placeholder="cm"></div>
    </div>
    
    <label>Activity</label>
    <select id="activity" style="margin-bottom: 10px;">
        <option value="1.2">Sedentary</option>
        <option value="1.55">Moderate</option>
        <option value="1.725">Very Active</option>
    </select>

    <button class="main-btn" onclick="calculate()">Update & Save Stats</button>

    <div id="result">
        <div style="text-align:center">Daily Maintenance: <strong id="maint" style="color:var(--primary)"></strong></div>
        
        <div class="macro-grid">
            <div class="macro-box p-box">P <strong id="prot"></strong></div>
            <div class="macro-box f-box">F <strong id="fats"></strong></div>
            <div class="macro-box c-box">C <strong id="carbs"></strong></div>
        </div>

        <div class="water-controls">
            <span>Water: <strong id="water-stat" style="color:var(--water)">0</strong> / <span id="water-goal"></span></span>
            <button onclick="addWater()" style="background:var(--water); color:white; padding:5px 10px; border-radius:5px;">+ Cup</button>
        </div>

        <div class="chart-container">
            <canvas id="historyChart"></canvas>
        </div>

        <button onclick="downloadPDF()" style="width:100%; margin-top:20px; padding:10px; background:#27ae60; color:white;">Download PDF Report</button>
    </div>
    
    <center><button class="btn-clear" onclick="clearData()">Wipe All Data</button></center>
</div>

<script>
    let myChart;
    let history = JSON.parse(localStorage.getItem('fitnessHistory')) || [];

    window.onload = () => {
        const theme = localStorage.getItem('theme') || 'dark';
        document.documentElement.setAttribute('data-theme', theme);
        if(localStorage.getItem('userStats')) {
            const d = JSON.parse(localStorage.getItem('userStats'));
            document.getElementById('weight').value = d.weight;
            document.getElementById('height').value = d.height;
            document.getElementById('age').value = d.age;
            calculate();
        }
    };

    function toggleTheme() {
        const t = document.documentElement.getAttribute('data-theme') === 'dark' ? 'light' : 'dark';
        document.documentElement.setAttribute('data-theme', t);
        localStorage.setItem('theme', t);
    }

    function calculate() {
        const w = parseFloat(document.getElementById('weight').value);
        const h = parseFloat(document.getElementById('height').value);
        const a = parseFloat(document.getElementById('age').value);
        if(!w || !h || !a) return;

        const bmr = (document.getElementById('gender').value === 'male') ? (10*w + 6.25*h - 5*a + 5) : (10*w + 6.25*h - 5*a - 161);
        const tdee = Math.round(bmr * document.getElementById('activity').value);
        
        // Update UI
        document.getElementById('maint').innerText = tdee + " kcal";
        document.getElementById('prot').innerText = Math.round((tdee*0.3)/4) + "g";
        document.getElementById('fats').innerText = Math.round((tdee*0.25)/9) + "g";
        document.getElementById('carbs').innerText = Math.round((tdee*0.45)/4) + "g";
        document.getElementById('water-goal').innerText = Math.round(w*33) + "ml";
        document.getElementById('result').style.display = 'block';

        // Save History (only if weight is different from last entry)
        if(history.length === 0 || history[history.length-1].w !== w) {
            history.push({ d: new Date().toLocaleDateString(), w: w });
            if(history.length > 7) history.shift();
            localStorage.setItem('fitnessHistory', JSON.stringify(history));
        }
        
        localStorage.setItem('userStats', JSON.stringify({weight:w, height:h, age:a}));
        initChart();
    }

    function addWater() {
        let val = parseInt(document.getElementById('water-stat').innerText);
        document.getElementById('water-stat').innerText = val + 250;
    }

    function initChart() {
        const ctx = document.getElementById('historyChart').getContext('2d');
        if(myChart) myChart.destroy();
        myChart = new Chart(ctx, {
            type: 'line',
            data: {
                labels: history.map(i => i.d),
                datasets: [{ label: 'Weight (kg)', data: history.map(i => i.w), borderColor: '#00d2ff', tension: 0.3 }]
            },
            options: { maintainAspectRatio: false, plugins: { legend: { display: false } } }
        });
    }

    function downloadPDF() {
        html2pdf().from(document.getElementById('capture-area')).save('Fitness_Report.pdf');
    }

    function clearData() { localStorage.clear(); location.reload(); }
</script>
</body>
</html>
