<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>우리들만의 비밀 기록장</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    
    <!-- Firebase SDK & App Logic -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, doc, setDoc, deleteDoc, query, orderBy } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // 1. Firebase 설정
        const firebaseConfig = {
            apiKey: "AIzaSyATTfsDTspXM6iHreZCyO_Apk4GWIk74JM",
            authDomain: "story2-60345.firebaseapp.com",
            projectId: "story2-60345",
            storageBucket: "story2-60345.firebasestorage.app",
            messagingSenderId: "64911818684",
            appId: "1:64911818684:web:fc18e4a06917b8f38435b9",
            measurementId: "G-VVLLW4VWCJ"
        };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'couple-diary-html';
        const ANNIVERSARY_DATE = new Date('2025-05-28');
        const SECRET_CODE = "EVOL";

        let photos = [];
        let messages = [];
        let currentTheme = 'pink';
        let currentView = 'home';
        let isDataSubscribed = false;
        let deleteTarget = null;
        let currentUser = null;

        const THEMES = {
            pink: { primary: 'bg-pink-500', text: 'text-pink-600', light: 'bg-pink-50', border: 'border-pink-200', bg: 'bg-[#FFF9FB]', memoSelf: 'bg-pink-500 text-white', memoOther: 'bg-white text-gray-700' },
            blue: { primary: 'bg-blue-500', text: 'text-blue-600', light: 'bg-blue-50', border: 'border-blue-200', bg: 'bg-[#F9FBFF]', memoSelf: 'bg-blue-500 text-white', memoOther: 'bg-white text-gray-700' },
            green: { primary: 'bg-emerald-500', text: 'text-emerald-600', light: 'bg-emerald-50', border: 'border-emerald-200', bg: 'bg-[#F9FFF9]', memoSelf: 'bg-emerald-500 text-white', memoOther: 'bg-white text-gray-700' },
            dark: { primary: 'bg-gray-800', text: 'text-gray-800', light: 'bg-gray-100', border: 'border-gray-300', bg: 'bg-[#F5F5F5]', memoSelf: 'bg-gray-700 text-white', memoOther: 'bg-white text-gray-700' }
        };

        const getTodayFormatted = () => {
            const now = new Date();
            return `${now.getFullYear()}.${String(now.getMonth() + 1).padStart(2, '0')}.${String(now.getDate()).padStart(2, '0')}`;
        };

        const calculateDDay = (dateStr) => {
            const target = new Date(dateStr);
            const diff = Math.floor((target - ANNIVERSARY_DATE) / (1000 * 60 * 60 * 24)) + 1;
            return diff >= 0 ? `D+${diff}` : `D${diff}`;
        };

        function enterApp() {
            document.getElementById('login-screen').classList.add('hidden');
            document.getElementById('app-screen').classList.remove('hidden');
            startSubscribingData();
            updateUI();
        }

        function updateUI() {
            const theme = THEMES[currentTheme];
            document.body.className = `min-h-screen ${theme.bg} transition-colors duration-500 pb-24`;
            
            const ddayEl = document.getElementById('header-dday');
            if(ddayEl) {
                ddayEl.innerText = calculateDDay(new Date());
                ddayEl.className = `text-xl font-black ${theme.text} uppercase`;
            }
            
            const heartEl = document.getElementById('header-heart');
            if(heartEl) {
                heartEl.setAttribute('class', `${theme.text} fill-current w-[18px] h-[18px]`);
            }

            ['home', 'gallery', 'theme'].forEach(view => {
                const btn = document.getElementById(`tab-${view}`);
                if (btn) {
                    const label = view === 'home' ? '홈' : view === 'gallery' ? '갤러리' : '테마';
                    btn.innerText = label;
                    btn.className = (view === currentView) 
                        ? `px-8 py-3 rounded-2xl text-sm font-black bg-white shadow-md ${theme.text}`
                        : `px-8 py-3 rounded-2xl text-sm font-black text-gray-400 hover:${theme.text}`;
                }
            });

            renderContent();
            lucide.createIcons();
        }

        window.setView = (view) => {
            currentView = view;
            updateUI();
        };

        window.changeTheme = async (themeId) => {
            currentTheme = themeId;
            updateUI();
            try {
                const themeDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'appearance');
                await setDoc(themeDocRef, { themeId }, { merge: true });
            } catch(e) { console.error("Theme save error:", e); }
        };

        window.openDeleteModal = (id, type) => {
            deleteTarget = { id, type };
            document.getElementById('delete-modal').classList.remove('hidden');
            document.getElementById('delete-modal').classList.add('flex');
        };

        window.closeDeleteModal = () => {
            deleteTarget = null;
            document.getElementById('delete-modal').classList.add('hidden');
            document.getElementById('delete-modal').classList.remove('flex');
        };

        window.confirmDelete = async () => {
            if (!deleteTarget) return;
            try {
                const collectionName = deleteTarget.type === 'photo' ? 'photos' : 'messages';
                await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', collectionName, deleteTarget.id));
                closeDeleteModal();
            } catch (err) {
                console.error("Delete error:", err);
            }
        };

        function renderContent() {
            const container = document.getElementById('main-content');
            if(!container) return;
            const theme = THEMES[currentTheme];
            
            if (currentView === 'home') {
                container.innerHTML = `
                    <div class="space-y-12 animate-fade-in">
                        <!-- 스토리 슬라이드 -->
                        <section class="space-y-6">
                            <div class="flex items-end justify-between px-2">
                                <h2 class="text-2xl font-black text-gray-800 tracking-tighter uppercase">STORY</h2>
                                <span class="text-[11px] font-black ${theme.text} opacity-80 italic tracking-tighter animate-pulse">옆으로 넘겨보세요 ➔</span>
                            </div>
                            <div class="flex overflow-x-auto gap-6 pb-8 px-2 no-scrollbar snap-x">
                                ${photos.map(p => `
                                    <div ondblclick="openDeleteModal('${p.id}', 'photo')" class="snap-start shrink-0 w-[280px] h-[380px] relative bg-white rounded-[2rem] overflow-hidden shadow-xl border border-gray-100 cursor-pointer select-none group">
                                        <img src="${p.url}" class="w-full h-full object-cover pointer-events-none group-hover:scale-105 transition-transform duration-700">
                                        <div class="absolute top-4 left-4 ${theme.primary} text-white px-4 py-1.5 rounded-full text-[10px] font-black shadow-md">${calculateDDay(p.date)}</div>
                                        <div class="absolute inset-x-0 bottom-0 bg-gradient-to-t from-black/80 via-black/10 p-6">
                                            <p class="text-white font-bold truncate mb-1 text-sm">${p.caption}</p>
                                            <p class="text-white/50 text-[9px] font-black tracking-widest uppercase">${p.date}</p>
                                        </div>
                                    </div>
                                `).join('') || `<div class="w-full py-24 text-center opacity-30 font-bold italic tracking-tighter">아직 올라온 추억이 없어요</div>`}
                            </div>
                        </section>

                        <!-- 하단 나란히 배치되는 섹션 -->
                        <div class="grid grid-cols-1 lg:grid-cols-2 gap-8 items-stretch">
                            <!-- 왼쪽: 사진 업로드 -->
                            <section class="bg-white rounded-[2.5rem] p-8 shadow-xl border ${theme.border} flex flex-col h-full">
                                <div class="flex items-center gap-3 mb-8">
                                    <div class="p-3 ${theme.light} rounded-2xl"><i data-lucide="camera" class="${theme.text}"></i></div>
                                    <div>
                                        <h3 class="text-lg font-black text-gray-800">사진 올리기</h3>
                                        <p class="text-[10px] text-gray-400 font-black uppercase">Upload Photo</p>
                                    </div>
                                </div>
                                <div class="space-y-5 flex-1">
                                    <div class="space-y-2">
                                        <label class="text-[12px] font-black text-gray-500 ml-2 uppercase">날짜</label>
                                        <input type="date" id="upload-date" value="${new Date().toISOString().split('T')[0]}" class="w-full p-4 bg-gray-50 rounded-2xl border-2 border-gray-100 outline-none focus:border-gray-300 font-bold text-gray-600">
                                    </div>
                                    <div class="space-y-2">
                                        <label class="text-[12px] font-black text-gray-500 ml-2 uppercase">코멘트</label>
                                        <input type="text" id="upload-caption" placeholder="하루 정리하기" class="w-full p-4 bg-gray-50 rounded-2xl border-2 border-gray-100 outline-none focus:border-gray-300 font-bold text-gray-600">
                                    </div>
                                    <label class="block mt-4">
                                        <div id="upload-btn" class="w-full py-5 ${theme.primary} text-white rounded-2xl font-black text-center cursor-pointer hover:shadow-lg active:scale-95 transition-all shadow-md">
                                            사진 선택하기
                                        </div>
                                        <input type="file" id="file-input" class="hidden" accept="image/*">
                                    </label>
                                </div>
                            </section>

                            <!-- 오른쪽: 메모장 -->
                            <section class="bg-white rounded-[2.5rem] p-8 shadow-xl border ${theme.border} flex flex-col h-[530px]">
                                <div class="flex items-center gap-3 mb-6">
                                    <div class="p-3 ${theme.light} rounded-2xl"><i data-lucide="message-circle" class="${theme.text}"></i></div>
                                    <div>
                                        <h3 class="text-lg font-black text-gray-800">메모 남기기</h3>
                                        <p class="text-[10px] text-gray-400 font-black uppercase">Memo Box</p>
                                    </div>
                                </div>
                                
                                <div id="memo-list" class="flex-1 overflow-y-auto space-y-4 mb-4 pr-2 no-scrollbar bg-gray-50/50 rounded-3xl p-4">
                                    ${messages.map(m => {
                                        const isSelf = currentUser && m.authorId === currentUser.uid;
                                        return `
                                            <div ondblclick="openDeleteModal('${m.id}', 'message')" class="flex flex-col ${isSelf ? 'items-end' : 'items-start'} cursor-pointer group">
                                                <div class="${isSelf ? theme.memoSelf : theme.memoOther} p-4 rounded-[1.5rem] ${isSelf ? 'rounded-tr-none' : 'rounded-tl-none'} shadow-sm max-w-[85%] border border-black/5">
                                                    <p class="text-sm font-bold leading-relaxed">${m.text}</p>
                                                </div>
                                                <span class="text-[8px] font-black text-gray-300 mt-1.5 uppercase tracking-tighter">${new Date(m.timestamp).toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'})}</span>
                                            </div>
                                        `;
                                    }).join('') || `<div class="h-full flex flex-col items-center justify-center opacity-20"><i data-lucide="edit-3" class="mb-2"></i><span class="font-bold italic text-sm">메모가 없습니다</span></div>`}
                                </div>

                                <div class="flex gap-2">
                                    <input type="text" id="memo-input" placeholder="여기에 메모를 쓰세요" class="flex-1 p-4 bg-gray-50 rounded-2xl border-2 border-gray-100 outline-none focus:border-gray-300 font-bold text-gray-600 text-sm">
                                    <button onclick="sendMemo()" class="${theme.primary} p-4 rounded-2xl text-white shadow-md active:scale-90 transition-transform">
                                        <i data-lucide="send" size="18"></i>
                                    </button>
                                </div>
                            </section>
                        </div>
                    </div>
                `;
                
                const fileInput = document.getElementById('file-input');
                if(fileInput) fileInput.onchange = handleFileUpload;
                
                const memoInput = document.getElementById('memo-input');
                if(memoInput) {
                    memoInput.onkeydown = (e) => {
                        if(e.code === 'Space') e.stopPropagation();
                        if(e.key === 'Enter') sendMemo();
                    };
                }
                
                const memoList = document.getElementById('memo-list');
                if(memoList) memoList.scrollTop = memoList.scrollHeight;
            } else if (currentView === 'gallery') {
                container.innerHTML = `
                    <div class="grid grid-cols-2 md:grid-cols-4 gap-4 animate-fade-in">
                        ${photos.map(p => `
                            <div ondblclick="openDeleteModal('${p.id}', 'photo')" class="aspect-square relative bg-white rounded-3xl overflow-hidden shadow-md group cursor-pointer select-none">
                                <img src="${p.url}" class="w-full h-full object-cover transition-transform group-hover:scale-110 duration-700 pointer-events-none">
                                <div class="absolute inset-0 bg-gradient-to-t from-black/70 opacity-0 group-hover:opacity-100 transition-opacity p-4 flex flex-col justify-end">
                                    <p class="text-white text-[10px] font-bold truncate">${p.caption}</p>
                                    <p class="text-white/50 text-[8px] font-black uppercase tracking-widest">${p.date}</p>
                                </div>
                            </div>
                        `).join('')}
                    </div>
                `;
            } else if (currentView === 'theme') {
                container.innerHTML = `
                    <div class="max-w-md mx-auto space-y-4 animate-fade-in text-center">
                        <h2 class="text-xl font-black text-gray-800">테마 설정</h2>
                        <p class="text-[10px] text-gray-400 font-black mb-8 uppercase tracking-[0.2em]">취향에 맞는 테마를 골라보세요</p>
                        ${Object.entries(THEMES).map(([id, t]) => `
                            <button onclick="changeTheme('${id}')" class="w-full p-6 bg-white rounded-[2.5rem] border-4 ${currentTheme === id ? t.border + ' ' + t.light : 'border-gray-50'} flex items-center justify-between transition-all hover:scale-[1.02] shadow-sm">
                                <div class="flex items-center gap-4">
                                    <div class="w-10 h-10 ${t.primary} rounded-2xl shadow-inner transform rotate-3"></div>
                                    <span class="font-black text-gray-700 uppercase tracking-tighter text-lg">${id}</span>
                                </div>
                                ${currentTheme === id ? '<div class="bg-gray-800 text-white p-1 rounded-full"><i data-lucide="check" size="16"></i></div>' : ''}
                            </button>
                        `).join('')}
                    </div>
                `;
            }
            lucide.createIcons();
        }

        async function handleFileUpload(e) {
            const file = e.target.files[0];
            if (!file) return;
            
            const btn = document.getElementById('upload-btn');
            const originalText = btn.innerText;
            btn.innerText = "업로드 중...";
            btn.style.opacity = "0.6";

            const reader = new FileReader();
            reader.onloadend = async () => {
                const base64 = reader.result;
                const caption = document.getElementById('upload-caption').value || "하루 정리하기";
                const date = document.getElementById('upload-date').value;

                try {
                    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'photos'), {
                        url: base64, caption, date, timestamp: Date.now(), authorId: currentUser?.uid
                    });
                } catch (err) { console.error(err); }

                btn.innerText = originalText;
                btn.style.opacity = "1";
                document.getElementById('upload-caption').value = "";
                updateUI();
            };
            reader.readAsDataURL(file);
        }

        window.sendMemo = async () => {
            const input = document.getElementById('memo-input');
            const text = input.value.trim();
            if(!text || !currentUser) return;

            try {
                await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'messages'), {
                    text, timestamp: Date.now(), authorId: currentUser.uid
                });
                input.value = "";
            } catch (err) { console.error(err); }
        };

        function startSubscribingData() {
            if (isDataSubscribed || !currentUser) return;
            isDataSubscribed = true;

            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'photos'), (snap) => {
                photos = snap.docs.map(d => ({id: d.id, ...d.data()})).sort((a,b) => new Date(b.date) - new Date(a.date));
                updateUI();
            });

            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'messages'), (snap) => {
                messages = snap.docs.map(d => ({id: d.id, ...d.data()})).sort((a,b) => a.timestamp - b.timestamp);
                updateUI();
            });

            onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'appearance'), (snap) => {
                if (snap.exists()) {
                    currentTheme = snap.data().themeId || 'pink';
                    updateUI();
                }
            });
        }

        onAuthStateChanged(auth, async (user) => {
            if (user) {
                currentUser = user;
                if (localStorage.getItem('isAuth') === 'true') enterApp();
            } else {
                try { await signInAnonymously(auth); } catch (err) {}
            }
        });

        window.checkCode = (e) => {
            if (e) e.preventDefault(); // 폼 제출 시 새로고침 방지
            const val = document.getElementById('pass-input').value.toUpperCase();
            if (val === SECRET_CODE) {
                localStorage.setItem('isAuth', 'true');
                enterApp();
            } else {
                const errEl = document.getElementById('error-msg');
                if(errEl) {
                    errEl.innerText = "코드가 맞지 않아요 😢";
                    errEl.classList.remove('hidden');
                }
            }
        };

        document.addEventListener('DOMContentLoaded', () => {
            const loginDateEl = document.getElementById('login-date');
            if(loginDateEl) loginDateEl.innerText = `2025.05.28 - ${getTodayFormatted()}`;
            lucide.createIcons();
        });

    </script>
    <style>
        @import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css');
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
        .animate-fade-in { animation: fadeIn 0.5s ease-out; }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(15px); } to { opacity: 1; transform: translateY(0); } }
        body { font-family: 'Pretendard', sans-serif; -webkit-tap-highlight-color: transparent; }
        input[type="date"]::-webkit-calendar-picker-indicator { opacity: 0.3; }
    </style>
</head>
<body class="bg-gray-50">

    <!-- 삭제 확인 모달 -->
    <div id="delete-modal" class="fixed inset-0 bg-black/40 z-[200] hidden items-center justify-center p-6 backdrop-blur-md">
        <div class="bg-white rounded-[2.5rem] p-8 w-full max-w-xs text-center shadow-2xl animate-fade-in">
            <div class="w-16 h-16 bg-red-50 rounded-2xl flex items-center justify-center mx-auto mb-4 transform -rotate-6">
                <i data-lucide="trash-2" class="text-red-400"></i>
            </div>
            <h3 class="text-lg font-black text-gray-800 mb-2">지울까요?</h3>
            <p class="text-xs text-gray-400 font-bold mb-8 uppercase tracking-widest">기록을 영구적으로 삭제합니다</p>
            <div class="flex gap-3">
                <button onclick="closeDeleteModal()" class="flex-1 py-4 bg-gray-100 text-gray-500 rounded-2xl font-black text-sm">취소</button>
                <button onclick="confirmDelete()" class="flex-1 py-4 bg-red-500 text-white rounded-2xl font-black text-sm shadow-lg shadow-red-200">삭제</button>
            </div>
        </div>
    </div>

    <!-- 로그인 화면 -->
    <div id="login-screen" class="fixed inset-0 bg-pink-50 z-[100] flex items-center justify-center p-6">
        <div class="bg-white p-10 rounded-[3.5rem] shadow-2xl w-full max-w-sm text-center border-4 border-white">
            <div class="w-20 h-20 bg-pink-50 rounded-3xl flex items-center justify-center mx-auto mb-6 shadow-inner transform rotate-12">
                <i data-lucide="heart" class="text-pink-400 w-10 h-10 fill-current"></i>
            </div>
            <h1 class="text-3xl font-black mb-2 text-gray-800 uppercase italic tracking-tighter">INPUT CODE</h1>
            <p id="login-date" class="text-gray-300 mb-10 text-[10px] font-black tracking-widest"></p>
            
            <form onsubmit="checkCode(event)" class="space-y-4">
                <input type="password" id="pass-input" placeholder="비밀코드 입력" class="w-full px-6 py-5 bg-gray-50 border-2 border-gray-100 rounded-[1.5rem] focus:border-pink-200 outline-none text-center font-black tracking-[0.5em] text-lg">
                <p id="error-msg" class="text-red-400 text-[10px] font-black hidden uppercase"></p>
                <button type="submit" class="w-full bg-pink-500 hover:bg-pink-600 text-white font-black py-5 rounded-[1.5rem] shadow-xl transition-all active:scale-95 text-lg uppercase">Enter</button>
            </form>
        </div>
    </div>

    <!-- 앱 메인 화면 -->
    <div id="app-screen" class="hidden">
        <header class="bg-white/80 backdrop-blur-xl border-b sticky top-0 z-50 px-8 py-5 flex justify-between items-center">
            <div class="flex items-center gap-3">
                <div class="w-10 h-10 rounded-2xl bg-gray-50 flex items-center justify-center shadow-inner">
                    <i id="header-heart" data-lucide="heart"></i>
                </div>
                <h1 id="header-dday" class="font-black text-gray-800">우리들만의 비밀 기록장</h1>
            </div>
            <button onclick="localStorage.removeItem('isAuth'); location.reload();" class="w-10 h-10 flex items-center justify-center text-gray-300 hover:text-gray-600 transition-colors bg-gray-50 rounded-2xl">
                <i data-lucide="log-out" size="18"></i>
            </button>
        </header>

        <main class="max-w-6xl mx-auto p-6 md:p-10 space-y-12">
            <!-- 탭 메뉴 -->
            <div class="flex bg-white/50 backdrop-blur p-2 rounded-[2rem] w-fit mx-auto shadow-sm border border-white">
                <button id="tab-home" onclick="setView('home')">홈</button>
                <button id="tab-gallery" onclick="setView('gallery')">갤러리</button>
                <button id="tab-theme" onclick="setView('theme')">테마</button>
            </div>

            <div id="main-content" class="pb-16"></div>
        </main>
    </div>

</body>
</html>
