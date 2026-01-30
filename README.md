<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Flashcard Elite 12.0 - Cloud</title>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-auth-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore-compat.js"></script>

    <style>
        /* MANTIVE TODO O SEU LAYOUT ORIGINAL */
        :root { --primary: #2563eb; --danger: #ef4444; --success: #22c55e; --bg: #f8fafc; --card-bg: #ffffff; --text: #1e293b; --border: #e2e8f0; --accent: #f59e0b; }
        [data-theme="dark"] { --bg: #0f172a; --card-bg: #1e293b; --text: #f8fafc; --border: #334155; }
        body { font-family: 'Inter', sans-serif; background: var(--bg); color: var(--text); display: flex; flex-direction: column; align-items: center; padding: 20px; transition: 0.3s; min-height: 100vh; margin: 0; }
        
        /* BARRA DE LOGIN DISCRETA */
        .auth-bar { width: 100%; max-width: 550px; display: flex; justify-content: space-between; align-items: center; margin-bottom: 15px; padding: 10px; background: var(--card-bg); border-radius: 10px; border: 1px solid var(--border); font-size: 0.8rem; }
        
        .section { background: var(--card-bg); padding: 20px; border-radius: 12px; border: 1px solid var(--border); width: 100%; max-width: 550px; margin-bottom: 20px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); box-sizing: border-box; }
        .btn { padding: 10px; border: none; border-radius: 8px; cursor: pointer; font-weight: 600; color: white; transition: 0.2s; font-size: 0.8rem; }
        .btn-ghost { background: transparent; color: var(--primary); border: 1px solid var(--primary); padding: 5px 10px; }
        
        /* Estilos do Card */
        .flashcard-container { perspective: 1000px; width: 100%; height: 260px; cursor: pointer; margin: 15px 0; }
        .flashcard { position: relative; width: 100%; height: 100%; transition: transform 0.4s; transform-style: preserve-3d; }
        .flashcard.flipped { transform: rotateY(180deg); }
        .front, .back { position: absolute; width: 100%; height: 100%; backface-visibility: hidden; display: flex; align-items: center; justify-content: center; padding: 25px; border-radius: 15px; border: 2px solid var(--primary); box-sizing: border-box; text-align: center; background: var(--card-bg); }
        .back { background: var(--primary); color: white; transform: rotateY(180deg); }
        input, select, textarea { width: 100%; padding: 10px; margin-bottom: 8px; border-radius: 8px; border: 1px solid var(--border); background: var(--card-bg); color: var(--text); }
    </style>
</head>
<body>

    <div class="auth-bar">
        <span id="userStatus">Carregando...</span>
        <button class="btn-ghost" id="authBtn" onclick="handleAuth()">Entrar / Cadastrar</button>
    </div>

    <div class="section">
        <div id="timer">25:00</div>
        <button class="btn btn-ghost" onclick="toggleTimer()">Focar</button>
    </div>

    <div class="section">
        <select id="topicSelect" onchange="updateSubtopics()"></select>
        <select id="subtopicSelect" onchange="loadCards()"></select>
        
        <div class="flashcard-container" onclick="this.querySelector('.flashcard').classList.toggle('flipped')">
            <div class="flashcard" id="cardElement">
                <div class="front" id="cardFront">Selecione uma matéria</div>
                <div class="back" id="cardBack">Verso</div>
            </div>
        </div>

        <div style="display:flex; gap:10px;">
            <button class="btn" style="background:var(--danger); flex:1" onclick="markCard(false)">ERREI</button>
            <button class="btn" style="background:var(--success); flex:1" onclick="markCard(true)">ACERTEI</button>
        </div>
    </div>

    <div class="section">
        <input type="text" id="impTopic" placeholder="Matéria">
        <input type="text" id="impSub" placeholder="Assunto">
        <textarea id="bulkArea" placeholder="Frente | Verso"></textarea>
        <button class="btn" style="background:var(--primary); width:100%" onclick="importBulk()">Salvar na Nuvem</button>
    </div>

    <script>
        // CONFIGURAÇÃO DO FIREBASE (COLE SEUS DADOS AQUI)
        const firebaseConfig = {
            apiKey: "COLE_SUA_API_KEY",
            authDomain: "SEU_PROJETO.firebaseapp.com",
            projectId: "SEU_PROJETO",
            storageBucket: "SEU_PROJETO.appspot.com",
            messagingSenderId: "SEU_ID",
            appId: "SEU_APP_ID"
        };

        firebase.initializeApp(firebaseConfig);
        const auth = firebase.auth();
        const db = firebase.firestore();

        let database = {};
        let currentCards = [], currentIndex = 0;

        // LOGIN E CADASTRO
        async function handleAuth() {
            const email = prompt("E-mail:");
            const pass = prompt("Senha (6 dígitos):");
            if (!email || !pass) return;
            try {
                await auth.signInWithEmailAndPassword(email, pass);
            } catch (e) {
                if (confirm("Criar nova conta?")) await auth.createUserWithEmailAndPassword(email, pass);
            }
        }

        auth.onAuthStateChanged(user => {
            if (user) {
                document.getElementById('userStatus').innerText = "Logado: " + user.email;
                document.getElementById('authBtn').innerText = "Sair";
                document.getElementById('authBtn').onclick = () => auth.signOut().then(() => location.reload());
                loadData(user.uid);
            } else {
                document.getElementById('userStatus').innerText = "Modo Offline";
            }
        });

        // SALVAR E CARREGAR
        function sync() {
            const user = auth.currentUser;
            if (user) db.collection("users").doc(user.uid).set({ database });
        }

        function loadData(uid) {
            db.collection("users").doc(uid).get().then(doc => {
                if (doc.exists) { database = doc.data().database || {}; renderMenus(); }
            });
        }

        // FUNÇÕES DO SITE (SIMPLIFICADAS)
        function importBulk() {
            const t = document.getElementById('impTopic').value, s = document.getElementById('impSub').value, text = document.getElementById('bulkArea').value;
            if(!database[t]) database[t] = {}; if(!database[t][s]) database[t][s] = [];
            text.split('\n').forEach(line => {
                const p = line.split('|');
                if(p.length === 2) database[t][s].push({ f: p[0].trim(), v: p[1].trim(), id: Date.now() });
            });
            sync(); renderMenus();
        }

        function renderMenus() {
            const sel = document.getElementById('topicSelect'); sel.innerHTML = "";
            Object.keys(database).forEach(t => { let o = document.createElement('option'); o.value = o.innerText = t; sel.appendChild(o); });
            updateSubtopics();
        }

        function updateSubtopics() {
            const t = document.getElementById('topicSelect').value, sel = document.getElementById('subtopicSelect');
            sel.innerHTML = "";
            if(t) Object.keys(database[t]).forEach(s => { let o = document.createElement('option'); o.value = o.innerText = s; sel.appendChild(o); });
            loadCards();
        }

        function loadCards() {
            const t = document.getElementById('topicSelect').value, s = document.getElementById('subtopicSelect').value;
            currentCards = (t && s) ? database[t][s] : []; currentIndex = 0; showCard();
        }

        function showCard() {
            if(currentCards[currentIndex]) {
                document.getElementById('cardFront').innerText = currentCards[currentIndex].f;
                document.getElementById('cardBack').innerText = currentCards[currentIndex].v;
            }
        }

        function markCard(ok) { if(currentIndex < currentCards.length - 1) { currentIndex++; showCard(); } else alert("Fim!"); }
    </script>
</body>
</html>
