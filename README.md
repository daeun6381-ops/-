<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Grade Master Dashboard</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.0.0/css/all.min.css" rel="stylesheet">
    
    <script type="importmap">
    {
      "imports": {
        "firebase/app": "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js",
        "firebase/auth": "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js",
        "firebase/firestore": "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js"
      }
    }
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Pretendard:wght@400;600;700;800&display=swap');
        body { 
            font-family: 'Pretendard', sans-serif; 
            background-color: #f1f5f9; 
            color: #1e293b; 
            overflow-x: hidden; 
        }
        .tab-content { transition: all 0.3s ease; }
        input::-webkit-outer-spin-button,
        input::-webkit-inner-spin-button { -webkit-appearance: none; margin: 0; }
        .grade-input-card { transition: transform 0.2s; }
        .grade-input-card:focus-within { transform: translateY(-2px); border-color: #3b82f6; }
        
        /* 잠금 화면 스타일 */
        #lockScreen {
            backdrop-filter: blur(20px);
            background-color: rgba(255, 255, 255, 0.9);
        }
    </style>
</head>
<body class="min-h-screen pb-24">

    <!-- 잠금 화면 (PIN 입력) -->
    <div id="lockScreen" class="fixed inset-0 z-[100] flex flex-col items-center justify-center hidden">
        <div class="text-center">
            <div class="w-16 h-16 bg-blue-600 rounded-2xl flex items-center justify-center text-white text-2xl mb-6 mx-auto shadow-lg shadow-blue-200">
                <i class="fas fa-lock"></i>
            </div>
            <h2 class="text-2xl font-black mb-2">보안 잠금</h2>
            <p class="text-slate-400 text-sm mb-8">비밀번호 4자리를 입력하세요</p>
            <div class="flex gap-4 justify-center mb-8">
                <input type="password" maxlength="1" class="pin-input w-12 h-16 bg-white border-2 border-slate-200 rounded-xl text-center text-2xl font-bold focus:border-blue-500 outline-none" shadow-sm>
                <input type="password" maxlength="1" class="pin-input w-12 h-16 bg-white border-2 border-slate-200 rounded-xl text-center text-2xl font-bold focus:border-blue-500 outline-none" shadow-sm>
                <input type="password" maxlength="1" class="pin-input w-12 h-16 bg-white border-2 border-slate-200 rounded-xl text-center text-2xl font-bold focus:border-blue-500 outline-none" shadow-sm>
                <input type="password" maxlength="1" class="pin-input w-12 h-16 bg-white border-2 border-slate-200 rounded-xl text-center text-2xl font-bold focus:border-blue-500 outline-none" shadow-sm>
            </div>
            <p id="lockError" class="text-red-500 text-xs font-bold opacity-0 transition-opacity">비밀번호가 일치하지 않습니다.</p>
        </div>
    </div>

    <header class="bg-white border-b border-slate-200 sticky top-0 z-50">
        <div class="max-w-2xl mx-auto px-6 py-4 flex justify-between items-center">
            <h1 class="text-xl font-black text-blue-600 flex items-center gap-2">
                <i class="fas fa-chart-line"></i> GRADE MASTER
            </h1>
            <div id="statusBadge" class="text-[10px] font-bold bg-slate-100 text-slate-400 px-3 py-1 rounded-full uppercase tracking-widest">
                Connecting...
            </div>
        </div>
    </header>

    <main id="app" class="max-w-2xl mx-auto px-4 mt-8">
        <div class="grid grid-cols-2 gap-4 mb-8">
            <div class="bg-gradient-to-br from-blue-600 to-blue-700 p-6 rounded-[2rem] text-white shadow-lg shadow-blue-200/50">
                <p class="text-[10px] font-bold opacity-80 mb-1 tracking-widest uppercase">Target Rank</p>
                <input type="text" id="goalRank" placeholder="순위 입력" onchange="window.saveData()" class="bg-transparent text-2xl font-black w-full outline-none placeholder:text-blue-300/50">
            </div>
            <div class="bg-white p-6 rounded-[2rem] border border-slate-200 shadow-sm">
                <p class="text-[10px] font-bold text-slate-400 mb-1 tracking-widest uppercase">Target Grade</p>
                <input type="text" id="goalGrade" placeholder="등급 입력" onchange="window.saveData()" class="bg-transparent text-2xl font-black w-full outline-none placeholder:text-slate-200">
            </div>
        </div>

        <nav class="flex justify-around mb-8 bg-white/50 backdrop-blur-md rounded-2xl p-1 border border-slate-200">
            <button onclick="showTab('chart')" id="btn-chart" class="flex-1 py-3 rounded-xl font-bold text-sm transition-all text-blue-600 bg-white shadow-sm">분석</button>
            <button onclick="showTab('input')" id="btn-input" class="flex-1 py-3 rounded-xl font-bold text-sm transition-all text-slate-400">기록</button>
            <button onclick="showTab('manage')" id="btn-manage" class="flex-1 py-3 rounded-xl font-bold text-sm transition-all text-slate-400">과목</button>
            <button onclick="showTab('settings')" id="btn-settings" class="flex-1 py-3 rounded-xl font-bold text-sm transition-all text-slate-400">설정</button>
        </nav>

        <!-- 탭 내용들 (기존과 동일) -->
        <section id="view-chart" class="tab-content animate-in fade-in slide-in-from-bottom-4 duration-500">
            <div class="bg-white p-6 rounded-[2.5rem] shadow-sm border border-slate-200 mb-6">
                <div class="flex justify-between items-center mb-6">
                    <h3 class="font-black text-slate-800">성적 추이</h3>
                    <select id="chartSelector" onchange="updateChart()" class="text-xs font-bold bg-slate-100 border-none rounded-full px-4 py-2 outline-none text-slate-600 cursor-pointer"></select>
                </div>
                <div class="h-64 relative">
                    <canvas id="mainChart"></canvas>
                    <div id="noDataOverlay" class="absolute inset-0 flex flex-col items-center justify-center bg-white/90 hidden">
                        <i class="fas fa-folder-open text-slate-200 text-4xl mb-2"></i>
                        <p class="text-slate-400 font-bold text-sm">기록을 먼저 추가해 주세요</p>
                    </div>
                </div>
            </div>
            <div class="bg-white rounded-[2.5rem] shadow-sm border border-slate-200 overflow-hidden">
                <div class="px-8 py-6 border-b border-slate-100 flex justify-between items-center">
                    <h3 class="font-black text-slate-800">최근 시험 결과</h3>
                    <span id="recordCount" class="text-[10px] font-black bg-blue-50 text-blue-600 px-2 py-1 rounded">0 RECORDS</span>
                </div>
                <div id="historyList" class="divide-y divide-slate-50"></div>
            </div>
        </section>

        <section id="view-input" class="tab-content hidden">
            <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                <div class="mb-8">
                    <h3 class="font-black text-2xl mb-2">성적 추가</h3>
                    <p class="text-sm text-slate-400">시험 결과를 입력하고 추이를 분석하세요.</p>
                </div>
                <div class="space-y-6">
                    <div>
                        <label class="text-[10px] font-bold text-slate-400 ml-4 mb-2 block uppercase tracking-widest">Exam Name</label>
                        <input type="text" id="examName" placeholder="예: 1학기 기말고사" class="w-full p-5 bg-slate-50 rounded-2xl outline-none focus:ring-2 focus:ring-blue-100 transition border border-transparent">
                    </div>
                    <div id="subjectGrid" class="grid grid-cols-2 gap-4"></div>
                    <button onclick="addRecord()" class="w-full bg-slate-900 text-white py-5 rounded-2xl font-black text-lg hover:bg-black active:scale-[0.98] transition-all shadow-lg shadow-slate-200">
                        데이터 저장하기
                    </button>
                </div>
            </div>
        </section>

        <section id="view-manage" class="tab-content hidden">
            <div class="bg-white p-8 rounded-[2.5rem] shadow-sm border border-slate-200">
                <h3 class="font-black text-2xl mb-6">과목 리스트</h3>
                <div class="flex gap-2 mb-8">
                    <input type="text" id="subjectInput" placeholder="새 과목 이름" class="flex-1 p-4 bg-slate-50 rounded-2xl outline-none border border-transparent focus:border-blue-200">
                    <button onclick="addSubject()" class="bg-blue-600 text-white px-6 rounded-2xl font-bold hover:bg-blue-700 transition-colors">추가</button>
                </div>
                <div id="subjectTags" class="flex flex-wrap gap-2"></div>
            </div>
        </section>

        <section id="view-settings" class="tab-content hidden">
            <div class="space-y-4">
                <div class="bg-white p-8 rounded-[2.5rem] border border-slate-200 mb-4">
                    <h3 class="font-black text-xl mb-4 text-slate-800">보안 설정</h3>
                    <div class="flex items-center justify-between p-4 bg-slate-50 rounded-2xl mb-4">
                        <div>
                            <p class="text-sm font-bold text-slate-700">비밀번호 잠금</p>
                            <p class="text-[10px] text-slate-400">앱 실행 시 PIN 번호를 확인합니다.</p>
                        </div>
                        <input type="password" id="pinSetting" maxlength="4" placeholder="4자리" 
                               onchange="window.updatePin(this.value)"
                               class="w-20 p-2 text-center bg-white border border-slate-200 rounded-xl outline-none font-bold">
                    </div>
                    <p class="text-[10px] text-red-400 text-center italic">* 비밀번호를 비워두면 잠금이 해제됩니다.</p>
                </div>

                <div class="bg-white p-8 rounded-[2.5rem] border border-slate-200">
                    <h3 class="font-black text-xl mb-2 text-red-600">위험 구역</h3>
                    <p class="text-sm text-slate-400 mb-6">데이터 초기화 (모든 기록 삭제)</p>
                    <button onclick="clearAll()" class="w-full py-4 bg-red-50 text-red-600 font-bold rounded-2xl hover:bg-red-600 hover:text-white transition-all">
                        모든 데이터 초기화
                    </button>
                </div>
            </div>
        </section>
    </main>

    <div id="toast" class="fixed bottom-24 left-1/2 -translate-x-1/2 bg-slate-900 text-white px-6 py-3 rounded-full opacity-0 pointer-events-none transition-all duration-300 text-sm font-bold z-[100] shadow-xl">
        메시지 내용
    </div>

    <script type="module">
        import { initializeApp } from "firebase/app";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "firebase/auth";
        import { getFirestore, doc, setDoc, onSnapshot } from "firebase/firestore";

        const firebaseConfig = {
            apiKey: "AIzaSyBv14g9crV8vGbobK5cdVxwqlrr0EiM0fA",
            authDomain: "bokk-55eae.firebaseapp.com",
            projectId: "bokk-55eae",
            storageBucket: "bokk-55eae.firebasestorage.app",
            messagingSenderId: "757990079253",
            appId: "1:757990079253:web:13c7b920f1283cb86bc71c",
            measurementId: "G-D6FLNPXR58"
        };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = 'grade-master-v1';

        let appState = {
            subjects: ['국어', '수학', '영어', '과학'],
            records: [],
            goals: { rank: '', grade: '' },
            pin: "" // 비밀번호 추가
        };
        let chartInstance = null;
        let isSyncing = false;
        let unsubscribe = null;
        let isUnlocked = false;

        // 잠금 화면 관련 로직
        const pinInputs = document.querySelectorAll('.pin-input');
        pinInputs.forEach((input, index) => {
            input.addEventListener('input', (e) => {
                if (e.target.value.length === 1 && index < 3) pinInputs[index + 1].focus();
                checkPin();
            });
            input.addEventListener('keydown', (e) => {
                if (e.key === 'Backspace' && !e.target.value && index > 0) pinInputs[index - 1].focus();
            });
        });

        function checkPin() {
            const entered = Array.from(pinInputs).map(i => i.value).join('');
            if (entered.length === 4) {
                if (entered === appState.pin) {
                    document.getElementById('lockScreen').classList.add('hidden');
                    isUnlocked = true;
                } else {
                    const err = document.getElementById('lockError');
                    err.classList.replace('opacity-0', 'opacity-100');
                    pinInputs.forEach(i => i.value = "");
                    pinInputs[0].focus();
                    setTimeout(() => err.classList.replace('opacity-100', 'opacity-0'), 2000);
                }
            }
        }

        window.updatePin = (val) => {
            appState.pin = val;
            window.saveData();
            showToast(val ? "비밀번호가 설정되었습니다." : "비밀번호가 해제되었습니다.");
        };

        onAuthStateChanged(auth, async (user) => {
            if (user) {
                const badge = document.getElementById('statusBadge');
                badge.innerText = "Cloud Sync";
                badge.className = "text-[10px] font-bold bg-green-50 text-green-600 px-3 py-1 rounded-full uppercase tracking-widest";

                if (unsubscribe) unsubscribe();
                const userDocRef = doc(db, 'artifacts', appId, 'users', user.uid, 'mainData', 'state');
                
                unsubscribe = onSnapshot(userDocRef, (docSnap) => {
                    if (docSnap.exists() && !isSyncing) {
                        const data = docSnap.data();
                        appState = data;
                        
                        // 잠금 처리 여부 판단
                        if (appState.pin && !isUnlocked) {
                            document.getElementById('lockScreen').classList.remove('hidden');
                        } else {
                            document.getElementById('lockScreen').classList.add('hidden');
                        }
                        
                        renderAll();
                    }
                }, (error) => {
                    console.error("Sync Error:", error);
                });
            } else {
                try { await signInAnonymously(auth); } catch (e) { showToast("연결 오류"); }
            }
        });

        window.saveData = async () => {
            if (!auth.currentUser) return;
            appState.goals.rank = document.getElementById('goalRank').value;
            appState.goals.grade = document.getElementById('goalGrade').value;
            isSyncing = true;
            try {
                const userDocRef = doc(db, 'artifacts', appId, 'users', auth.currentUser.uid, 'mainData', 'state');
                await setDoc(userDocRef, appState);
            } catch (e) { console.error(e); }
            finally { isSyncing = false; renderAll(); }
        };

        // 기타 기능 함수 (생략 없이 유지)
        window.addSubject = () => {
            const input = document.getElementById('subjectInput');
            const name = input.value.trim();
            if (!name || appState.subjects.includes(name)) return;
            appState.subjects.push(name);
            input.value = '';
            window.saveData();
        };

        window.removeSubject = (name) => {
            if (!confirm(`'${name}' 과목을 삭제하시겠습니까?`)) return;
            appState.subjects = appState.subjects.filter(s => s !== name);
            window.saveData();
        };

        window.addRecord = () => {
            const name = document.getElementById('examName').value.trim();
            if (!name) return showToast("이름을 입력하세요.");
            const scores = {};
            let total = 0, count = 0;
            appState.subjects.forEach(sub => {
                const el = document.getElementById(`input-${sub}`);
                if (el && el.value) {
                    const val = parseFloat(el.value);
                    scores[sub] = val;
                    total += val;
                    count++;
                }
            });
            if (count === 0) return showToast("점수를 입력하세요.");
            appState.records.push({ id: Date.now(), name, date: new Date().toLocaleDateString(), scores, average: parseFloat((total / count).toFixed(2)) });
            document.getElementById('examName').value = '';
            window.saveData();
            window.showTab('chart');
        };

        window.deleteRecord = (id) => {
            if (!confirm("삭제하시겠습니까?")) return;
            appState.records = appState.records.filter(r => r.id !== id);
            window.saveData();
        };

        window.clearAll = () => {
            if (!confirm("초기화하시겠습니까?")) return;
            appState.records = [];
            window.saveData();
        };

        function renderAll() {
            if (document.getElementById('goalRank')) document.getElementById('goalRank').value = appState.goals.rank || '';
            if (document.getElementById('goalGrade')) document.getElementById('goalGrade').value = appState.goals.grade || '';
            if (document.getElementById('pinSetting')) document.getElementById('pinSetting').value = appState.pin || '';
            
            const grid = document.getElementById('subjectGrid');
            if (grid) {
                grid.innerHTML = appState.subjects.map(sub => `
                    <div class="grade-input-card relative bg-slate-50 border border-slate-100 rounded-2xl p-4">
                        <label class="text-[9px] font-black text-slate-400 absolute top-2 left-4 uppercase tracking-tighter">${sub}</label>
                        <input type="number" id="input-${sub}" placeholder="0.0" step="0.1"
                               class="w-full bg-transparent text-xl font-bold text-right outline-none mt-2 pr-2">
                    </div>
                `).join('');
            }

            const tags = document.getElementById('subjectTags');
            if (tags) {
                tags.innerHTML = appState.subjects.map(sub => `
                    <div class="group flex items-center gap-2 bg-white px-4 py-2 rounded-full border border-slate-200">
                        <span class="text-sm font-bold text-slate-600">${sub}</span>
                        <button onclick="removeSubject('${sub}')" class="text-slate-300 hover:text-red-500 transition-colors">
                            <i class="fas fa-times-circle"></i>
                        </button>
                    </div>
                `).join('');
            }

            const historyList = document.getElementById('historyList');
            if (historyList) {
                historyList.innerHTML = [...appState.records].reverse().map(r => `
                    <div class="p-6 flex items-center justify-between hover:bg-slate-50 transition-colors">
                        <div><h4 class="font-black text-slate-800">${r.name}</h4><p class="text-[10px] font-bold text-slate-300 tracking-wider uppercase">${r.date}</p></div>
                        <div class="flex items-center gap-6"><div class="text-right"><span class="text-2xl font-black text-blue-600">${r.average}</span></div>
                        <button onclick="deleteRecord(${r.id})" class="text-slate-200 hover:text-red-400 transition-colors p-2"><i class="fas fa-trash-alt"></i></button></div>
                    </div>
                `).join('') || `<div class="p-12 text-center text-slate-300 text-sm font-bold">기록이 없습니다</div>`;
            }

            const selector = document.getElementById('chartSelector');
            if (selector) {
                const currentSelected = selector.value || 'average';
                selector.innerHTML = `<option value="average">전체 평균</option>` + appState.subjects.map(s => `<option value="${s}">${s}</option>`).join('');
                selector.value = currentSelected;
            }
            window.updateChart();
        }

        window.updateChart = () => {
            const ctx = document.getElementById('mainChart')?.getContext('2d');
            if (!ctx) return;
            const overlay = document.getElementById('noDataOverlay');
            if (appState.records.length === 0) { overlay.classList.remove('hidden'); if (chartInstance) chartInstance.destroy(); return; }
            overlay.classList.add('hidden');
            const key = document.getElementById('chartSelector').value;
            const labels = appState.records.map(r => r.name);
            const data = appState.records.map(r => key === 'average' ? r.average : (r.scores[key] || null));
            if (chartInstance) chartInstance.destroy();
            chartInstance = new Chart(ctx, {
                type: 'line',
                data: { labels, datasets: [{ data, borderColor: '#3b82f6', borderWidth: 4, tension: 0.4, pointBackgroundColor: '#fff', pointBorderColor: '#3b82f6', pointBorderWidth: 3, pointRadius: 6 }] },
                options: { responsive: true, maintainAspectRatio: false, scales: { y: { reverse: true, min: 1, max: 9 }, x: { grid: { display: false } } }, plugins: { legend: { display: false } } }
            });
        };

        window.showTab = (tabId) => {
            document.querySelectorAll('.tab-content').forEach(el => el.classList.add('hidden'));
            document.getElementById(`view-${tabId}`).classList.remove('hidden');
            document.querySelectorAll('nav button').forEach(btn => {
                btn.classList.remove('text-blue-600', 'bg-white', 'shadow-sm');
                btn.classList.add('text-slate-400');
            });
            document.getElementById(`btn-${tabId}`).classList.add('text-blue-600', 'bg-white', 'shadow-sm');
            if (tabId === 'chart') window.updateChart();
        };

        function showToast(msg) {
            const toast = document.getElementById('toast');
            if (!toast) return;
            toast.innerText = msg;
            toast.classList.replace('opacity-0', 'opacity-100');
            setTimeout(() => toast.classList.replace('opacity-100', 'opacity-0'), 2500);
        }
    </script>
</body>
</html>
