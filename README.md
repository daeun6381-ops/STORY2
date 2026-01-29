<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>우리들만의 비밀 기록장</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken, signOut } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, doc, setDoc, deleteDoc, query, orderBy } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Firebase 설정 (환경 변수 자동 주입 대응)
        const firebaseConfig = typeof __firebase_config !== 'undefined' 
            ? JSON.parse(__firebase_config) 
            : {
                authDomain: "story2-60345.firebaseapp.com",
                projectId: "story2-60345",
                storageBucket: "story2-60345.firebasestorage.app",
                messagingSenderId: "64911818684",
                appId: "1:64911818684:web:fc18e4a06917b8f38435b9"
            };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'couple-diary-final';
        
        const ANNIVERSARY_DATE = new Date('2025-05-28');
        const SECRET_CODE = "EVOL";

        let photos = [];
        let messages = [];
        let currentTheme = 'pink';
        let currentView = 'home';
        let currentUser = null;

        const THEMES = {
            pink: { primary: 'bg-pink-500', text: 'text-pink-600', light: 'bg-pink-50', border: 'border-pink-200', bg: 'bg-[#FFF9FB]', memoSelf: 'bg-pink-500 text-white', memoOther: 'bg-white text-gray-700' },
            blue: { primary: 'bg-blue-500', text: 'text-blue-600', light: 'bg-blue-50', border: 'border-blue-200', bg: 'bg-[#F9FBFF]', memoSelf: 'bg-blue-500 text-white', memoOther: 'bg-white text-gray-700' },
            green: { primary: 'bg-emerald-500', text: 'text-emerald-600', light: 'bg-emerald-50', border: 'border-emerald-200', bg: 'bg-[#F9FFF9]', memoSelf: 'bg-emerald-500 text-white', memoOther: 'bg-white text-gray-700' },
            dark: { primary: 'bg-gray-800', text: 'text-gray-800', light: 'bg-gray-100', border: 'border-gray-300', bg: 'bg-[#F5F5F5]', memoSelf: 'bg-gray-700 text-white', memoOther: 'bg-white text-gray-700' }
        };

        window.addEventListener('load', () => {
            const passInput = document.getElementById('pass-input');
            if (passInput) {
                passInput.value = '';
                passInput.focus();
            }
        });

        async function startApp() {
            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('app-screen').classList.remove('hidden');
            
            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'photos'), (snap) => {
                photos = snap.docs.map(d => ({id: d.id, ...d.data()})).sort((a,b) => new Date(b.date) - new Date(a.date));
                renderContent();
            });

            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'messages'), (snap) => {
                messages = snap.docs.map(d => ({id: d.id, ...d.data()})).sort((a,b) => a.timestamp - b.timestamp);
                renderContent();
            });

            onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'appearance'), (snap) => {
                if (snap.exists()) {
                    currentTheme = snap.data().themeId || 'pink';
                    updateThemeUI();
                }
            });
        }

        window.checkCode = async () => {
            const input = document.getElementById('pass-input');
            const btn = document.getElementById('connect-btn');
            const error = document.getElementById('error-msg');
            const val = input.value.trim().toUpperCase();

            if (val === SECRET_CODE) {
                btn.innerText = "Connecting...";
                btn.disabled = true;
                
                try {
                    if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                        const cred = await signInWithCustomToken(auth, __initial_auth_token);
                        currentUser = cred.user;
                    } else {
                        const cred = await signInAnonymously(auth);
                        currentUser = cred.user;
                    }
                    startApp();
                } catch (err) {
                    console.error(err);
                    error.innerText = "인증 오류: " + err.code;
                    error.classList.remove('hidden');
                    btn.innerText = "Connect";
                    btn.disabled = false;
                }
            } else {
                error.innerText = "코드가 일치하지 않습니다.";
                error.classList.remove('hidden');
                input.value = '';
                input.focus();
            }
        };

        function updateThemeUI() {
            const theme = THEMES[currentTheme];
            document.body.className = `min-h-screen ${theme.bg} transition-colors duration-500`;
            const headerDday = document.getElementById('header-dday');
            if(headerDday) {
                const diff = Math.floor((new Date() - ANNIVERSARY_DATE) / (1000 * 60 * 60 * 24)) + 1;
                headerDday.innerText = `D+${diff}`;
                headerDday.className = `font-black text-2xl ${theme.text}`;
            }
            renderContent();
        }

        window.setView = (view) => {
            currentView = view;
            renderContent();
        };

        window.sendMemo = async () => {
            const input = document.getElementById('memo-input');
            if (!input.value.trim() || !currentUser) return;
            await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'messages'), {
                text: input.value,
                timestamp: Date.now(),
                authorId: currentUser.uid
            });
            input.value = '';
        };

        function renderContent() {
            const container = document.getElementById('main-content');
            if (!container) return;
            const theme = THEMES[currentTheme];

            if (currentView === 'home') {
                container.innerHTML = `
                    <div class="space-y-12 animate-fade-in">
                        <div class="flex overflow-x-auto gap-6 pb-6 no-scrollbar snap-x px-4">
                            ${photos.map(p => `
                                <div class="snap-center shrink-0 w-72 h-96 bg-white rounded-[2rem] overflow-hidden shadow-xl border border-white/50 relative group">
                                    <img src="${p.url}" class="w-full h-full object-cover group-hover:scale-105 transition-transform duration-700">
                                    <div class="absolute bottom-0 inset-x-0 p-6 bg-gradient-to-t from-black/80 via-black/20 to-transparent text-white">
                                        <p class="font-bold text-lg mb-1 truncate">${p.caption}</p>
                                        <p class="text-[10px] font-medium tracking-wider opacity-60">${p.date}</p>
                                    </div>
                                </div>
                            `).join('') || '<div class="w-full py-24 text-center text-gray-300 font-bold italic">아직 기록된 사진이 없어요</div>'}
                        </div>
                        
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-8 px-4">
                            <div class="bg-white/70 backdrop-blur-sm p-8 rounded-[2.5rem] shadow-xl border border-white relative overflow-hidden group">
                                <div class="absolute top-0 right-0 p-4 opacity-5 group-hover:opacity-10 transition-opacity">
                                    <i data-lucide="camera" class="w-24 h-24"></i>
                                </div>
                                <h3 class="text-xl font-black mb-6 flex items-center gap-2 ${theme.text}"><i data-lucide="camera" class="w-5 h-5"></i> 오늘 기록하기</h3>
                                <input type="text" id="upload-caption" placeholder="무슨 일이 있었나요?" class="w-full p-4 bg-white/50 rounded-2xl mb-4 outline-none font-bold border-2 border-transparent focus:border-gray-100">
                                <input type="date" id="upload-date" value="${new Date().toISOString().split('T')[0]}" class="w-full p-4 bg-white/50 rounded-2xl mb-6 outline-none font-bold border-2 border-transparent focus:border-gray-100">
                                <label class="block w-full py-4 ${theme.primary} text-white text-center rounded-2xl font-black cursor-pointer active:scale-95 transition-all shadow-lg hover:brightness-110">
                                    사진 선택 및 저장
                                    <input type="file" id="file-input" class="hidden" accept="image/*" onchange="uploadPhoto(this)">
                                </label>
                            </div>

                            <div class="bg-white/70 backdrop-blur-sm p-8 rounded-[2.5rem] shadow-xl border border-white flex flex-col h-[500px]">
                                <h3 class="text-xl font-black mb-6 flex items-center gap-2 ${theme.text}"><i data-lucide="message-circle" class="w-5 h-5"></i> 우리들의 쪽지</h3>
                                <div id="memo-list" class="flex-1 overflow-y-auto space-y-4 mb-4 pr-2 no-scrollbar">
                                    ${messages.map(m => `
                                        <div class="flex flex-col ${m.authorId === currentUser?.uid ? 'items-end' : 'items-start'}">
                                            <div class="${m.authorId === currentUser?.uid ? theme.memoSelf : theme.memoOther} px-5 py-3 rounded-[1.2rem] max-w-[85%] shadow-sm border border-black/5">
                                                <p class="font-bold text-sm">${m.text}</p>
                                            </div>
                                        </div>
                                    `).join('')}
                                </div>
                                <div class="flex gap-2">
                                    <input type="text" id="memo-input" placeholder="하고 싶은 말..." class="flex-1 p-4 bg-white/50 rounded-2xl outline-none font-bold border-2 border-transparent focus:border-gray-100" onkeydown="if(event.key==='Enter') sendMemo()">
                                    <button onclick="sendMemo()" class="${theme.primary} p-4 rounded-2xl text-white shadow-lg active:scale-90 transition-all">
                                        <i data-lucide="send"></i>
                                    </button>
                                </div>
                            </div>
                        </div>
                    </div>
                `;
            } else if (currentView === 'gallery') {
                container.innerHTML = `
                    <div class="px-4 animate-fade-in">
                        <div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
                            ${photos.map(p => `
                                <div class="aspect-square relative group overflow-hidden rounded-2xl shadow-md border-2 border-white">
                                    <img src="${p.url}" class="w-full h-full object-cover group-hover:scale-110 transition-transform duration-500">
                                    <div class="absolute inset-0 bg-black/40 opacity-0 group-hover:opacity-100 transition-opacity flex items-center justify-center p-4">
                                        <p class="text-white text-xs font-bold text-center">${p.caption}</p>
                                    </div>
                                </div>
                            `).join('')}
                        </div>
                    </div>
                `;
            } else if (currentView === 'theme') {
                container.innerHTML = `
                    <div class="px-4 max-w-md mx-auto space-y-4 animate-fade-in">
                        ${Object.keys(THEMES).map(id => `
                            <button onclick="saveTheme('${id}')" class="w-full p-6 rounded-[2rem] bg-white/70 border-4 ${currentTheme === id ? THEMES[id].border : 'border-white'} flex items-center justify-between transition-all shadow-md group">
                                <div class="flex items-center gap-4">
                                    <div class="w-10 h-10 rounded-full ${THEMES[id].primary} shadow-inner group-hover:scale-110 transition-transform"></div>
                                    <span class="font-black text-gray-700 uppercase tracking-widest">${id}</span>
                                </div>
                                ${currentTheme === id ? '<i data-lucide="check" class="text-gray-800"></i>' : ''}
                            </button>
                        `).join('')}
                    </div>
                `;
            }
            lucide.createIcons();
            const list = document.getElementById('memo-list');
            if(list) list.scrollTop = list.scrollHeight;
        }

        window.uploadPhoto = async (input) => {
            const file = input.files[0];
            if (!file || !currentUser) return;
            const reader = new FileReader();
            reader.onload = async () => {
                await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'photos'), {
                    url: reader.result,
                    caption: document.getElementById('upload-caption').value || "추억 한 조각",
                    date: document.getElementById('upload-date').value,
                    timestamp: Date.now(),
                    authorId: currentUser.uid
                });
            };
            reader.readAsDataURL(file);
        };

        window.saveTheme = async (id) => {
            currentTheme = id;
            updateThemeUI();
            await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'appearance'), { themeId: id }, { merge: true });
        };

    </script>
    <style>
        @import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css');
        body { font-family: 'Pretendard', sans-serif; -webkit-tap-highlight-color: transparent; }
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .animate-fade-in { animation: fadeIn 0.6s cubic-bezier(0.22, 1, 0.36, 1); }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(20px); } to { opacity: 1; transform: translateY(0); } }
        
        #pass-input { -webkit-text-security: disc; }
    </style>
</head>
<body class="bg-[#FFF9FB]">

    <!-- LOGIN SCREEN -->
    <div id="login-screen" class="fixed inset-0 z-[100] bg-[#FFF9FB] flex items-center justify-center p-6">
        <div class="bg-white p-12 rounded-[4rem] shadow-2xl w-full max-w-md text-center border-[12px] border-white animate-fade-in">
            <div class="w-24 h-24 bg-pink-100 rounded-[2.5rem] flex items-center justify-center mx-auto mb-10 shadow-inner rotate-12">
                <i data-lucide="heart" class="text-pink-500 fill-current w-12 h-12"></i>
            </div>
            <h1 class="text-4xl font-black mb-3 text-gray-800 tracking-tighter uppercase italic">LOVE DIARY</h1>
            <p class="text-gray-300 text-[11px] font-bold mb-12 tracking-[0.4em] uppercase">Private Couple Space</p>
            
            <div class="space-y-4">
                <input type="text" 
                       id="pass-input" 
                       placeholder="ENTER CODE" 
                       autocomplete="off"
                       class="w-full py-6 bg-gray-50 rounded-3xl border-4 border-transparent focus:border-pink-100 outline-none text-center font-black text-2xl tracking-[0.3em] transition-all">
                <p id="error-msg" class="text-red-400 text-[10px] font-bold hidden uppercase tracking-widest"></p>
                <button onclick="checkCode()" 
                        id="connect-btn"
                        class="w-full bg-pink-500 text-white font-black py-6 rounded-3xl shadow-xl shadow-pink-100 active:scale-95 transition-all text-xl uppercase tracking-[0.1em]">
                    Start
                </button>
            </div>
        </div>
    </div>

    <!-- MAIN APP SCREEN -->
    <div id="app-screen" class="hidden">
        <header class="bg-white/70 backdrop-blur-xl sticky top-0 z-50 px-8 py-6 border-b border-white">
            <div class="max-w-5xl mx-auto flex items-center justify-between">
                <div class="flex items-center gap-4">
                    <div class="w-10 h-10 bg-pink-50 rounded-2xl flex items-center justify-center shadow-sm">
                        <i data-lucide="heart" class="text-pink-500 fill-current w-5 h-5"></i>
                    </div>
                    <span id="header-dday" class="font-black text-2xl tracking-tighter">D+0</span>
                </div>
                <button onclick="location.reload()" class="p-3 bg-white rounded-2xl text-gray-400 shadow-sm border border-gray-50 active:rotate-180 transition-transform duration-700">
                    <i data-lucide="refresh-cw" size="20"></i>
                </button>
            </div>
        </header>

        <main class="max-w-5xl mx-auto py-10 space-y-12">
            <nav class="flex justify-center gap-3 p-3 bg-white/40 backdrop-blur-md rounded-[2.5rem] w-fit mx-auto shadow-sm border border-white">
                <button onclick="setView('home')" class="px-10 py-3 rounded-[1.8rem] font-black text-sm text-gray-400 hover:bg-white hover:text-gray-800 transition-all shadow-sm">HOME</button>
                <button onclick="setView('gallery')" class="px-10 py-3 rounded-[1.8rem] font-black text-sm text-gray-400 hover:bg-white hover:text-gray-800 transition-all shadow-sm">GALLERY</button>
                <button onclick="setView('theme')" class="px-10 py-3 rounded-[1.8rem] font-black text-sm text-gray-400 hover:bg-white hover:text-gray-800 transition-all shadow-sm">THEME</button>
            </nav>

            <div id="main-content"></div>
        </main>
    </div>

</body>
</html>
