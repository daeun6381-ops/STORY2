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
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, collection, addDoc, onSnapshot, doc, setDoc, deleteDoc, query, orderBy } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {
            apiKey: "",
            authDomain: "story2-60345.firebaseapp.com",
            projectId: "story2-60345",
            storageBucket: "story2-60345.firebasestorage.app",
            messagingSenderId: "64911818684",
            appId: "1:64911818684:web:fc18e4a06917b8f38435b9"
        };

        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'couple-diary-html';
        const ANNIVERSARY_DATE = new Date('2025-05-28');
        const SECRET_CODE = "EVOL"; // 설정된 비밀번호

        let photos = [];
        let messages = [];
        let currentTheme = 'pink';
        let currentView = 'home';
        let isDataSubscribed = false;
        let deleteTarget = null;
        let currentUser = null;
        let isUploading = false; 

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
            document.getElementById('login-screen').style.display = 'none';
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
                ddayEl.className = `text-2xl md:text-3xl font-black ${theme.text} tracking-tighter uppercase`;
            }
            
            const heartIcons = document.querySelectorAll('.theme-heart');
            heartIcons.forEach(icon => {
                icon.setAttribute('class', `theme-heart ${theme.text} fill-current w-5 h-5`);
            });

            ['home', 'gallery', 'theme'].forEach(view => {
                const btn = document.getElementById(`tab-${view}`);
                if (btn) {
                    const label = view === 'home' ? '홈' : view === 'gallery' ? '갤러리' : '테마';
                    btn.innerText = label;
                    btn.className = (view === currentView) 
                        ? `px-8 py-3 rounded-2xl text-base font-black bg-white shadow-md ${theme.text}`
                        : `px-8 py-3 rounded-2xl text-base font-black text-gray-400 hover:${theme.text}`;
                }
            });

            renderContent();
            lucide.createIcons();
        }

        window.setView = (view) => {
            currentView = view;
            updateUI();
            window.scrollTo({ top: 0, behavior: 'smooth' });
        };

        window.changeTheme = async (themeId) => {
            currentTheme = themeId;
            updateUI();
            if (!currentUser) return;
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
            if (!deleteTarget || !currentUser) return;
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
                        <section class="space-y-6">
                            <div class="flex items-end justify-between px-4">
                                <h2 class="text-3xl font-black text-gray-800 tracking-tighter uppercase">STORY</h2>
                                <span class="text-xs font-black ${theme.text} opacity-80 italic tracking-tighter animate-pulse">옆으로 밀어서 추억 보기 ➔</span>
                            </div>
                            <div class="flex overflow-x-auto gap-8 pb-8 px-4 no-scrollbar snap-x">
                                ${photos.map(p => `
                                    <div ondblclick="openDeleteModal('${p.id}', 'photo')" class="snap-center shrink-0 w-[300px] h-[420px] md:w-[350px] md:h-[480px] relative bg-white rounded-[2.5rem] overflow-hidden shadow-xl border border-gray-100 cursor-pointer select-none group">
                                        <img src="${p.url}" class="w-full h-full object-cover pointer-events-none group-hover:scale-105 transition-transform duration-700">
                                        <div class="absolute top-6 left-6 ${theme.primary} text-white px-4 py-1.5 rounded-full text-xs font-black shadow-md">${calculateDDay(p.date)}</div>
                                        <div class="absolute inset-x-0 bottom-0 bg-gradient-to-t from-black/80 via-transparent p-8">
                                            <p class="text-white font-bold truncate mb-2 text-lg">${p.caption}</p>
                                            <p class="text-white/60 text-[10px] font-black tracking-widest uppercase">${p.date}</p>
                                        </div>
                                    </div>
                                `).join('') || `<div class="w-full py-32 text-center opacity-30 font-bold italic tracking-tighter text-xl text-gray-400">아직 올라온 추억이 없어요</div>`}
                            </div>
                        </section>

                        <div class="grid grid-cols-1 lg:grid-cols-2 gap-10 items-start px-4">
                            <section class="bg-white rounded-[3.5rem] p-10 shadow-xl border ${theme.border} flex flex-col">
                                <div class="flex items-center gap-5 mb-10">
                                    <div class="p-4 ${theme.light} rounded-[1.5rem]"><i data-lucide="camera" class="${theme.text} w-6 h-6"></i></div>
                                    <div>
                                        <h3 class="text-2xl font-black text-gray-800 tracking-tight">새로운 기록 남기기</h3>
                                        <p class="text-xs text-gray-400 font-black uppercase tracking-wider">Daily Upload</p>
                                    </div>
                                </div>
                                <div class="space-y-8">
                                    <div class="space-y-3">
                                        <label class="text-xs font-black text-gray-400 ml-4 uppercase">날짜 선택</label>
                                        <input type="date" id="upload-date" value="${new Date().toISOString().split('T')[0]}" class="w-full p-5 bg-gray-50 rounded-[1.5rem] border-2 border-transparent focus:bg-white focus:border-gray-200 outline-none font-bold text-gray-700 text-lg transition-all">
                                    </div>
                                    <div class="space-y-3">
                                        <label class="text-xs font-black text-gray-400 ml-4 uppercase">코멘트</label>
                                        <input type="text" id="upload-caption" placeholder="하루 정리하기" class="w-full p-5 bg-gray-50 rounded-[1.5rem] border-2 border-transparent focus:bg-white focus:border-gray-200 outline-none font-bold text-gray-700 text-lg transition-all">
                                    </div>
                                    <div class="pt-2">
                                        <label id="upload-label" class="cursor-pointer group">
                                            <div id="upload-btn" class="w-full py-6 ${theme.primary} text-white rounded-[2rem] font-black text-xl text-center shadow-lg group-active:scale-[0.98] transition-all">
                                                사진 업로드
                                            </div>
                                            <input type="file" id="file-input" class="hidden" accept="image/*">
                                        </label>
                                    </div>
                                </div>
                            </section>

                            <section class="bg-white rounded-[3.5rem] p-10 shadow-xl border ${theme.border} flex flex-col h-[650px]">
                                <div class="flex items-center gap-5 mb-8">
                                    <div class="p-4 ${theme.light} rounded-[1.5rem]"><i data-lucide="message-circle" class="${theme.text} w-6 h-6"></i></div>
                                    <div>
                                        <h3 class="text-2xl font-black text-gray-800 tracking-tight">메모 남기기</h3>
                                        <p class="text-xs text-gray-400 font-black uppercase tracking-wider">Private Messenger</p>
                                    </div>
                                </div>
                                
                                <div id="memo-list" class="flex-1 overflow-y-auto space-y-5 mb-8 pr-2 no-scrollbar bg-gray-50/50 rounded-[2.5rem] p-6">
                                    ${messages.map(m => {
                                        const isSelf = currentUser && m.authorId === currentUser.uid;
                                        return `
                                            <div ondblclick="openDeleteModal('${m.id}', 'message')" class="flex flex-col ${isSelf ? 'items-end' : 'items-start'} cursor-pointer group">
                                                <div class="${isSelf ? theme.memoSelf : theme.memoOther} px-6 py-4 rounded-[1.5rem] ${isSelf ? 'rounded-tr-none' : 'rounded-tl-none'} shadow-sm max-w-[85%] border border-black/5">
                                                    <p class="text-base font-bold leading-relaxed">${m.text}</p>
                                                </div>
                                                <span class="text-[10px] font-black text-gray-300 mt-2 px-2 uppercase tracking-tighter">
                                                    ${new Date(m.timestamp).toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'})}
                                                </span>
                                            </div>
                                        `;
                                    }).join('') || `<div class="h-full flex flex-col items-center justify-center opacity-20"><i data-lucide="edit-3" class="mb-4 w-10 h-10"></i><span class="font-bold italic text-lg">메시지를 남겨보세요</span></div>`}
                                </div>

                                <div class="flex gap-4">
                                    <input type="text" id="memo-input" placeholder="메시지 입력..." class="flex-1 p-5 bg-gray-50 rounded-[1.5rem] border-2 border-transparent focus:bg-white focus:border-gray-200 outline-none font-bold text-gray-700 text-lg transition-all">
                                    <button onclick="sendMemo()" class="${theme.primary} p-5 rounded-[1.5rem] text-white shadow-lg active:scale-90 transition-transform">
                                        <i data-lucide="send" size="24"></i>
                                    </button>
                                </div>
                            </section>
                        </div>
                    </div>
                `;
                
                const fileInput = document.getElementById('file-input');
                if(fileInput) fileInput.addEventListener('change', handleFileUpload);
                
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
                    <div class="grid grid-cols-2 md:grid-cols-3 xl:grid-cols-4 gap-6 animate-fade-in px-4">
                        ${photos.map(p => `
                            <div ondblclick="openDeleteModal('${p.id}', 'photo')" class="aspect-[3/4] relative bg-white rounded-[2rem] overflow-hidden shadow-lg group cursor-pointer select-none border border-gray-100">
                                <img src="${p.url}" class="w-full h-full object-cover transition-transform group-hover:scale-110 duration-1000 pointer-events-none">
                                <div class="absolute inset-0 bg-gradient-to-t from-black/80 via-transparent opacity-0 group-hover:opacity-100 transition-opacity p-6 flex flex-col justify-end">
                                    <p class="text-white text-lg font-bold truncate mb-1">${p.caption}</p>
                                    <p class="text-white/60 text-[10px] font-black uppercase tracking-widest">${p.date}</p>
                                </div>
                            </div>
                        `).join('')}
                    </div>
                `;
            } else if (currentView === 'theme') {
                container.innerHTML = `
                    <div class="max-w-4xl mx-auto space-y-10 animate-fade-in text-center py-12">
                        <h2 class="text-3xl font-black text-gray-800 tracking-tight uppercase">THEME SELECT</h2>
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-6 px-4">
                            ${Object.entries(THEMES).map(([id, t]) => `
                                <button onclick="changeTheme('${id}')" class="w-full p-8 bg-white rounded-[2.5rem] border-4 ${currentTheme === id ? t.border + ' ' + t.light : 'border-gray-50'} flex items-center justify-between transition-all hover:shadow-lg hover:-translate-y-1">
                                    <div class="flex items-center gap-6">
                                        <div class="w-12 h-12 ${t.primary} rounded-[1rem] shadow-inner transform rotate-6"></div>
                                        <span class="font-black text-gray-700 uppercase tracking-tighter text-xl">${id}</span>
                                    </div>
                                    ${currentTheme === id ? '<div class="bg-gray-800 text-white p-2 rounded-full"><i data-lucide="check" size="18"></i></div>' : ''}
                                </button>
                            `).join('')}
                        </div>
                    </div>
                `;
            }
            lucide.createIcons();
        }

        async function handleFileUpload(e) {
            const file = e.target.files[0];
            if (!file || isUploading || !currentUser) return;
            
            isUploading = true; 
            const btn = document.getElementById('upload-btn');
            const originalText = btn.innerText;
            btn.innerText = "저장 중...";
            btn.style.opacity = "0.5";

            const reader = new FileReader();
            reader.onload = async () => {
                const base64 = reader.result;
                const captionInput = document.getElementById('upload-caption');
                const dateInput = document.getElementById('upload-date');
                
                const caption = captionInput ? captionInput.value || "오늘의 조각" : "오늘의 조각";
                const date = dateInput ? dateInput.value : new Date().toISOString().split('T')[0];

                try {
                    await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'photos'), {
                        url: base64, 
                        caption: caption, 
                        date: date, 
                        timestamp: Date.now(), 
                        authorId: currentUser.uid
                    });
                    
                    if(captionInput) captionInput.value = "";
                    e.target.value = ""; 
                } catch (err) { 
                    console.error("Upload Error:", err); 
                } finally {
                    isUploading = false;
                    btn.innerText = originalText;
                    btn.style.opacity = "1";
                }
            };
            
            reader.onerror = () => {
                isUploading = false;
                btn.innerText = originalText;
                btn.style.opacity = "1";
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
                photos = snap.docs.map(d => ({id: d.id, ...d.data()})).sort((a,b) => {
                    if (b.date !== a.date) return new Date(b.date) - new Date(a.date);
                    return b.timestamp - a.timestamp;
                });
                updateUI();
            }, (error) => console.error("Photos sub error:", error));

            onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'messages'), (snap) => {
                messages = snap.docs.map(d => ({id: d.id, ...d.data()})).sort((a,b) => a.timestamp - b.timestamp);
                updateUI();
            }, (error) => console.error("Messages sub error:", error));

            onSnapshot(doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'appearance'), (snap) => {
                if (snap.exists()) {
                    currentTheme = snap.data().themeId || 'pink';
                    updateUI();
                }
            }, (error) => console.error("Theme sub error:", error));
        }

        // Firebase 인증 리스너
        onAuthStateChanged(auth, async (user) => {
            if (user) {
                currentUser = user;
                if (localStorage.getItem('couple_diary_auth') === 'true') {
                    enterApp();
                }
            }
        });

        // 비밀코드 체크 및 입장
        window.checkCode = async (e) => {
            if (e) e.preventDefault();
            
            const inputEl = document.getElementById('pass-input');
            const btnEl = document.getElementById('connect-btn');
            const errEl = document.getElementById('error-msg');
            
            const val = inputEl.value.trim().toUpperCase(); 
            
            if (val === SECRET_CODE) {
                // 1. UI 상태 변경
                btnEl.innerText = "Connecting...";
                btnEl.disabled = true;
                errEl.classList.add('hidden');

                try {
                    // 2. 인증 시도 (이미 인증되어 있지 않은 경우만)
                    if (!currentUser) {
                        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                            const cred = await signInWithCustomToken(auth, __initial_auth_token);
                            currentUser = cred.user;
                        } else {
                            const cred = await signInAnonymously(auth);
                            currentUser = cred.user;
                        }
                    }
                    
                    // 3. 인증 성공 시 플래그 저장 및 입장
                    localStorage.setItem('couple_diary_auth', 'true');
                    enterApp();
                } catch (err) {
                    console.error("Connection Error:", err);
                    btnEl.innerText = "Connect";
                    btnEl.disabled = false;
                    showError("서버와 연결할 수 없어요. 다시 시도해주세요.");
                }
            } else {
                showError("비밀코드가 일치하지 않아요.");
            }
        };

        function showError(msg) {
            const errEl = document.getElementById('error-msg');
            const inputEl = document.getElementById('pass-input');
            if(errEl) {
                errEl.innerText = msg;
                errEl.classList.remove('hidden');
                inputEl.classList.add('border-red-300');
                inputEl.value = ""; // 틀렸을 때 입력창 비우기
                inputEl.focus();
                setTimeout(() => inputEl.classList.remove('border-red-300'), 1000);
            }
        }

        window.onload = () => {
            // 브라우저 자동 완성 방지 및 초기화
            const passInput = document.getElementById('pass-input');
            if(passInput) {
                passInput.value = "";
                passInput.focus();
            }

            const loginDateEl = document.getElementById('login-date');
            if(loginDateEl) loginDateEl.innerText = `Since 2025.05.28 - Today ${getTodayFormatted()}`;
            lucide.createIcons();
            
            // 앱 로드 시 익명 인증 미리 준비
            (async () => {
                try {
                    if (!auth.currentUser) {
                         if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                            await signInWithCustomToken(auth, __initial_auth_token);
                        } else {
                            await signInAnonymously(auth);
                        }
                    }
                } catch(e) { console.warn("Background auth silent fail:", e); }
            })();
        };

    </script>
    <style>
        @import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css');
        .no-scrollbar::-webkit-scrollbar { display: none; }
        .no-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
        .animate-fade-in { animation: fadeIn 0.8s cubic-bezier(0.16, 1, 0.3, 1); }
        @keyframes fadeIn { from { opacity: 0; transform: translateY(30px); } to { opacity: 1; transform: translateY(0); } }
        body { font-family: 'Pretendard', sans-serif; -webkit-tap-highlight-color: transparent; }
        input[type="date"]::-webkit-calendar-picker-indicator { opacity: 0.3; width: 24px; height: 24px; cursor: pointer; }
        
        /* 자동완성 시 배경색 변하는 것 방지 */
        input:-webkit-autofill,
        input:-webkit-autofill:hover, 
        input:-webkit-autofill:focus {
            -webkit-box-shadow: 0 0 0px 1000px #F9FAFB inset;
            transition: background-color 5000s ease-in-out 0s;
        }
    </style>
</head>
<body class="bg-pink-50 min-h-screen">

    <!-- 삭제 확인 모달 -->
    <div id="delete-modal" class="fixed inset-0 bg-black/60 z-[200] hidden items-center justify-center p-8 backdrop-blur-md">
        <div class="bg-white rounded-[3rem] p-10 w-full max-sm text-center shadow-2xl animate-fade-in border-4 border-white">
            <div class="w-16 h-16 bg-red-50 rounded-[1.5rem] flex items-center justify-center mx-auto mb-6 transform -rotate-6">
                <i data-lucide="trash-2" class="text-red-400 w-8 h-8"></i>
            </div>
            <h3 class="text-2xl font-black text-gray-800 mb-3">기록을 지울까요?</h3>
            <p class="text-sm text-gray-400 font-bold mb-8 uppercase tracking-widest leading-relaxed">복구할 수 없습니다</p>
            <div class="flex gap-4">
                <button onclick="closeDeleteModal()" class="flex-1 py-4 bg-gray-100 text-gray-500 rounded-2xl font-black text-base">취소</button>
                <button onclick="confirmDelete()" class="flex-1 py-4 bg-red-500 text-white rounded-2xl font-black text-base shadow-lg shadow-red-200">삭제</button>
            </div>
        </div>
    </div>

    <!-- 로그인 화면 -->
    <div id="login-screen" class="fixed inset-0 bg-pink-50 z-[100] flex items-center justify-center p-8">
        <div class="bg-white p-12 md:p-16 rounded-[4rem] shadow-2xl w-full max-w-xl text-center border-8 border-white">
            <div class="w-24 h-24 bg-pink-50 rounded-[2rem] flex items-center justify-center mx-auto mb-8 shadow-inner transform rotate-12">
                <i data-lucide="heart" class="text-pink-400 w-12 h-12 fill-current"></i>
            </div>
            <h1 class="text-4xl font-black mb-3 text-gray-800 uppercase italic tracking-tighter">INPUT CODE</h1>
            <p id="login-date" class="text-gray-300 mb-12 text-xs font-black tracking-[0.3em] uppercase"></p>
            
            <form onsubmit="checkCode(event)" class="space-y-6" autocomplete="off">
                <input type="password" 
                       id="pass-input" 
                       placeholder="CODE" 
                       autocomplete="new-password"
                       class="w-full px-8 py-6 bg-gray-50 border-4 border-transparent rounded-[2.5rem] focus:bg-white focus:border-pink-200 outline-none text-center font-black tracking-[0.5em] text-3xl transition-all">
                <p id="error-msg" class="text-red-400 text-xs font-black hidden uppercase">코드가 맞지 않아요</p>
                <button type="submit" 
                        id="connect-btn" 
                        class="w-full bg-pink-500 hover:bg-pink-600 text-white font-black py-6 rounded-[2.5rem] shadow-xl transition-all active:scale-95 text-xl uppercase tracking-widest">
                    Connect
                </button>
            </form>
        </div>
    </div>

    <!-- 앱 메인 화면 -->
    <div id="app-screen" class="hidden">
        <header class="bg-white/90 backdrop-blur-3xl border-b sticky top-0 z-50 px-6 py-4 transition-all duration-500 shadow-sm">
            <div class="max-w-7xl mx-auto flex items-center justify-between">
                <div class="flex items-center gap-3">
                    <div class="p-2 bg-gray-50 rounded-xl shadow-inner">
                        <i data-lucide="heart" class="theme-heart"></i>
                    </div>
                    <h1 id="header-dday" class="font-black leading-none text-pink-500">D+0</h1>
                </div>

                <div class="flex items-center gap-3">
                    <button onclick="localStorage.removeItem('couple_diary_auth'); location.reload();" class="p-2 text-gray-300 hover:text-gray-600 transition-colors bg-gray-50 rounded-xl shadow-inner">
                        <i data-lucide="log-out" size="20"></i>
                    </button>
                </div>
            </div>
        </header>

        <main class="max-w-[1400px] mx-auto p-6 md:p-10 space-y-8">
            <div class="flex bg-white/70 backdrop-blur-2xl p-2 rounded-[2.5rem] w-fit mx-auto shadow-xl border-4 border-white mb-4">
                <button id="tab-home" onclick="setView('home')">홈</button>
                <button id="tab-gallery" onclick="setView('gallery')">갤러리</button>
                <button id="tab-theme" onclick="setView('theme')">테마</button>
            </div>

            <div id="main-content" class="pb-24 max-w-5xl mx-auto"></div>
        </main>
    </div>

</body>
</html>
