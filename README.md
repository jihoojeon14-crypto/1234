import React, { useState, useEffect, useRef } from 'react';
import { 
  Home, 
  Calendar, 
  Users, 
  MessageSquare, 
  Plus, 
  Trash2, 
  Edit2, 
  LogOut, 
  Send, 
  ChevronRight,
  ShieldCheck,
  Bell,
  Sparkles,
  RefreshCw,
  Zap,
  Lock,
  Unlock,
  KeyRound,
  Eye,
  EyeOff,
  Megaphone,
  Watch,
  Link as LinkIcon
} from 'lucide-react';

import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot } from 'firebase/firestore';

const ADMIN_ID = 'jay';
const ADMIN_PW = 'wjswlgn2024';

const INITIAL_SCHEDULES = [
  { id: 1, title: "어머니 생신 파티 🎂", date: "2026-05-20", description: "저녁 7시 한정식집 예약 완료", category: "행사" },
  { id: 2, title: "거실 에어컨 필터 청소 ❄️", date: "2026-06-01", description: "다 같이 오전 11시에 시작하기", category: "가사" }
];

const INITIAL_ALLOWED_USERS = [
  { id: 'jay', pin: '1234', canChangePin: true, isAdmin: true },
  { id: 'mom', pin: '1111', canChangePin: true, isAdmin: false },
  { id: 'dad', pin: '2222', canChangePin: false, isAdmin: false },
  { id: 'sister', pin: '3333', canChangePin: false, isAdmin: false }
];

const firebaseConfig = typeof __firebase_config !== 'undefined'
  ? JSON.parse(__firebase_config)
  : {
      apiKey: "demo-api-key",
      authDomain: "demo-auth-domain",
      projectId: "demo-project-id",
      storageBucket: "demo-storage-bucket",
      messagingSenderId: "demo-sender-id",
      appId: "demo-app-id"
    };

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

const FamilySite = () => {
  const [user, setUser] = useState(null);
  const [loginId, setLoginId] = useState('');
  const [loginPw, setLoginPw] = useState('');
  const [loginMessage, setLoginMessage] = useState({ type: '', text: '' });

  // 시스템 설정 및 워치모드 상태
  const [adminPassword, setAdminPassword] = useState(ADMIN_PW);
  const [siteName, setSiteName] = useState('우리 가족 아지트');
  const [dashboardLinks, setDashboardLinks] = useState([
    { id: 1, name: '우리가족 사이트 모음', url: 'https://sites.google.com/view/family2626/%ED%99%88' }
  ]);
  const [isWatchMode, setIsWatchMode] = useState(false);
  const [systemSiteName, setSystemSiteName] = useState('우리 가족 아지트');
  const [currentAdminPw, setCurrentAdminPw] = useState('');
  const [newAdminPw, setNewAdminPw] = useState('');
  const [confirmNewAdminPw, setConfirmNewAdminPw] = useState('');
  const [newLinkName, setNewLinkName] = useState('');
  const [newLinkUrl, setNewLinkUrl] = useState('');

  const [currentView, setCurrentView] = useState('dashboard');
  const [schedules, setSchedules] = useState(INITIAL_SCHEDULES);
  
  // 객체 형식으로 확장된 허용 유저 관리
  const [allowedUsers, setAllowedUsers] = useState(INITIAL_ALLOWED_USERS);
  
  // PIN 보안 관련 상태
  const [isUnlocked, setIsUnlocked] = useState(false); // 잠금 해제 여부
  const [isPinModalOpen, setIsPinModalOpen] = useState(false);
  const [pinInput, setPinInput] = useState('');
  const [pinError, setPinError] = useState('');
  const [pendingAction, setPendingAction] = useState(null); // 검증 완료 후 동작

  // PIN 변경 폼 상태
  const [currentPinInput, setCurrentPinInput] = useState('');
  const [newPinInput, setNewPinInput] = useState('');
  const [confirmNewPinInput, setConfirmNewPinInput] = useState('');
  const [pinChangeMessage, setPinChangeMessage] = useState({ type: '', text: '' });

  // 관리자 전용 개별 회원 PIN 임시 변경 및 보기 모드 상태
  const [visiblePins, setVisiblePins] = useState({}); // { [userId]: boolean }
  const [adminEditingPins, setAdminEditingPins] = useState({}); // { [userId]: string }

  // 신규 가입 폼 상태
  const [newUserId, setNewUserId] = useState('');
  const [newUserPin, setNewUserPin] = useState('');
  const [newUserChangePermission, setNewUserChangePermission] = useState(false);

  // 공지사항 상태
  const [notices, setNotices] = useState([
    { id: 1, title: '가족 아지트 이용 안내', content: '서로 존중하며 따뜻한 대화를 나누는 공간입니다.\n일정 등록과 확인을 자유롭게 해주세요!', date: '2026-05-17', author: 'Admin' }
  ]);
  const [isNoticeModalOpen, setIsNoticeModalOpen] = useState(false);
  const [noticeFormData, setNoticeFormData] = useState({ title: '', content: '' });
  const [editingNotice, setEditingNotice] = useState(null);

  // 알림 상태
  const [notifications, setNotifications] = useState([]);
  const [isNotiDropdownOpen, setIsNotiDropdownOpen] = useState(false);
  const [sendingNotiTo, setSendingNotiTo] = useState(null);
  const [notiMessage, setNotiMessage] = useState('');

  // 브라우저 내장 alert, confirm 차단 문제를 방지하기 위한 인앱 모달 상태
  const [modalAlert, setModalAlert] = useState({ isOpen: false, type: 'alert', title: '', message: '', onConfirm: null });

  // 데이터베이스 로딩 상태
  const [fbUser, setFbUser] = useState(null);
  const [isDbLoading, setIsDbLoading] = useState(true);

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Firebase Auth error:", err);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (firebaseUser) => {
      setFbUser(firebaseUser);
    });
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!fbUser) return;

    const settingsDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'main');
    const allowedUsersDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'allowedUsers', 'main');
    const schedulesDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'schedules', 'main');
    const noticesDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'notices', 'main');
    const notificationsDocRef = doc(db, 'artifacts', appId, 'public', 'data', 'notifications', 'main');

    // 1. 설정 동기화
    const unsubSettings = onSnapshot(settingsDocRef, (snap) => {
      if (snap.exists()) {
        const data = snap.data();
        if (data.siteName) {
          setSiteName(data.siteName);
          setSystemSiteName(data.siteName);
        }
        if (data.dashboardLinks) setDashboardLinks(data.dashboardLinks);
        if (data.adminPassword) setAdminPassword(data.adminPassword);
      } else {
        setDoc(settingsDocRef, {
          siteName: '우리 가족 아지트',
          dashboardLinks: [
            { id: 1, name: '우리가족 사이트 모음', url: 'https://sites.google.com/view/family2626/%ED%99%88' }
          ],
          adminPassword: ADMIN_PW
        });
      }
    }, (err) => console.error("Settings db error:", err));

    // 2. 가입 유저 동기화
    const unsubAllowedUsers = onSnapshot(allowedUsersDocRef, (snap) => {
      if (snap.exists()) {
        const data = snap.data();
        if (data.users) setAllowedUsers(data.users);
      } else {
        setDoc(allowedUsersDocRef, { users: INITIAL_ALLOWED_USERS });
      }
    }, (err) => console.error("Allowed users db error:", err));

    // 3. 일정 동기화
    const unsubSchedules = onSnapshot(schedulesDocRef, (snap) => {
      if (snap.exists()) {
        const data = snap.data();
        if (data.schedules) setSchedules(data.schedules);
      } else {
        setDoc(schedulesDocRef, { schedules: INITIAL_SCHEDULES });
      }
    }, (err) => console.error("Schedules db error:", err));

    // 4. 공지 동기화
    const unsubNotices = onSnapshot(noticesDocRef, (snap) => {
      if (snap.exists()) {
        const data = snap.data();
        if (data.notices) setNotices(data.notices);
      } else {
        const defaultNotices = [
          { id: 1, title: '가족 아지트 이용 안내', content: '서로 존중하며 따뜻한 대화를 나누는 공간입니다.\n일정 등록과 확인을 자유롭게 해주세요!', date: '2026-05-17', author: 'Admin' }
        ];
        setDoc(noticesDocRef, { notices: defaultNotices });
      }
    }, (err) => console.error("Notices db error:", err));

    // 5. 알림 동기화
    const unsubNotifications = onSnapshot(notificationsDocRef, (snap) => {
      if (snap.exists()) {
        const data = snap.data();
        if (data.notifications) setNotifications(data.notifications);
      } else {
        setDoc(notificationsDocRef, { notifications: [] });
      }
    }, (err) => console.error("Notifications db error:", err));

    setIsDbLoading(false);

    return () => {
      unsubSettings();
      unsubAllowedUsers();
      unsubSchedules();
      unsubNotices();
      unsubNotifications();
    };
  }, [fbUser]);

  const saveSchedulesToDb = async (newSchedules) => {
    if (!fbUser) return;
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'schedules', 'main'), { schedules: newSchedules });
    } catch (err) {
      console.error("Schedules save error:", err);
    }
  };

  const saveAllowedUsersToDb = async (newUsers) => {
    if (!fbUser) return;
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'allowedUsers', 'main'), { users: newUsers });
    } catch (err) {
      console.error("Users save error:", err);
    }
  };

  const saveNoticesToDb = async (newNotices) => {
    if (!fbUser) return;
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'notices', 'main'), { notices: newNotices });
    } catch (err) {
      console.error("Notices save error:", err);
    }
  };

  const saveNotificationsToDb = async (newNotifications) => {
    if (!fbUser) return;
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'notifications', 'main'), { notifications: newNotifications });
    } catch (err) {
      console.error("Notifications save error:", err);
    }
  };

  const saveSettingsToDb = async (newSiteName, newLinks, newPassword) => {
    if (!fbUser) return;
    try {
      await setDoc(doc(db, 'artifacts', appId, 'public', 'data', 'settings', 'main'), {
        siteName: newSiteName,
        dashboardLinks: newLinks,
        adminPassword: newPassword
      });
    } catch (err) {
      console.error("Settings save error:", err);
    }
  };

  const triggerAlert = (title, message) => {
    setModalAlert({ isOpen: true, type: 'alert', title, message, onConfirm: null });
  };

  const triggerConfirm = (title, message, onConfirm) => {
    setModalAlert({ isOpen: true, type: 'confirm', title, message, onConfirm });
  };

  // 일정 모달
  const [isModalOpen, setIsModalOpen] = useState(false);
  const [editingSchedule, setEditingSchedule] = useState(null);
  const [formData, setFormData] = useState({ title: '', date: '', description: '', category: '행사' });

  // AI 챗봇
  const [isChatOpen, setIsChatOpen] = useState(false);
  const [chatMessages, setChatMessages] = useState([
    { role: 'ai', text: '안녕하세요! 우리 가족 전용 Gemini AI 비서입니다. 개인정보 잠금이 해제되어 무엇이든 물어보실 수 있습니다. 일정 확인이나 페이지 이동을 요청해 보세요!' }
  ]);
  const [chatInput, setChatInput] = useState('');
  const [isAiTyping, setIsAiTyping] = useState(false);
  const [briefing, setBriefing] = useState("");
  const [isBriefingLoading, setIsBriefingLoading] = useState(false);
  const chatEndRef = useRef(null);

  // 알림창 외부 클릭 감지용 Ref
  const notiRef = useRef(null);

  // 챗봇 드래그 관련 상태
  const [chatOffset, setChatOffset] = useState({ x: 0, y: 0 });
  const [isDraggingChat, setIsDraggingChat] = useState(false);
  const dragStartPos = useRef({ startX: 0, startY: 0, initialOffsetX: 0, initialOffsetY: 0, isDragging: false });

  // 워치 토글 버튼 드래그 관련 상태
  const [toggleOffset, setToggleOffset] = useState({ x: 0, y: 0 });
  const toggleDragStartPos = useRef({ startX: 0, startY: 0, initialOffsetX: 0, initialOffsetY: 0, isDragging: false });

  useEffect(() => {
    chatEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [chatMessages]);

  const callGemini = async (prompt, systemInstruction = "") => {
    const apiKey = ""; // Canvas 환경에서 자동 주입됨
    const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent?key=${apiKey}`;

    const payload = {
      contents: [{ parts: [{ text: prompt }] }],
      systemInstruction: {
        parts: [{ text: systemInstruction }]
      }
    };

    try {
      const response = await fetch(apiUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });
      const result = await response.json();
      return result.candidates?.[0]?.content?.parts?.[0]?.text || "오류가 발생했습니다.";
    } catch (error) {
      console.error("Gemini API Error:", error);
      return "죄송합니다. AI 응답을 가져오는 중 오류가 발생했습니다.";
    }
  };

  const handleLogin = (e) => {
    e.preventDefault();
    const id = loginId.trim();
    
    // 허용된 계정 목록 매칭
    const foundUser = allowedUsers.find(u => u.id === id);
    if (!foundUser) {
      setLoginMessage({ type: 'error', text: '우리 가족 멤버로 등록되지 않은 아이디입니다.' });
      return;
    }

    if (id === ADMIN_ID) {
      if (loginPw !== adminPassword) {
        setLoginMessage({ type: 'error', text: '관리자 비밀번호가 틀렸습니다.' });
        return;
      }
      setUser({ id, isAdmin: true });
    } else {
      setUser({ id, isAdmin: false });
    }
    
    setIsUnlocked(false); // 로그인 후 기본 잠금 처리
    setLoginMessage({ type: 'success', text: '로그인 성공!' });
  };

  const handleLogout = () => {
    setUser(null);
    setLoginId('');
    setLoginPw('');
    setLoginMessage({ type: '', text: '' });
    setIsUnlocked(false);
    setCurrentView('dashboard');
  };

  // 현재 로그인한 유저에 따른 보안 PIN 조회
  const getCurrentUserPin = () => {
    if (!user) return '1234';
    const found = allowedUsers.find(u => u.id === user.id);
    return found ? found.pin : '1234';
  };

  const secureAccess = (action, callback) => {
    if (isUnlocked) {
      callback();
    } else {
      setPendingAction(() => () => {
        setIsUnlocked(true);
        callback();
      });
      setPinInput('');
      setPinError('');
      setIsPinModalOpen(true);
    }
  };

  const handlePinSubmit = (e) => {
    if (e) e.preventDefault();
    const targetPin = getCurrentUserPin();
    
    if (pinInput === targetPin) {
      setIsPinModalOpen(false);
      setIsUnlocked(true);
      setPinError('');
      setPinInput('');
      if (pendingAction) {
        pendingAction();
        setPendingAction(null);
      }
    } else {
      setPinError('PIN 비밀번호가 올바르지 않습니다. 다시 시도해주세요.');
      setPinInput('');
    }
  };

  const handlePinInputChange = (num) => {
    if (pinInput.length < 4) {
      const newVal = pinInput + num;
      setPinInput(newVal);
      const targetPin = getCurrentUserPin();

      if (newVal === targetPin) {
        setTimeout(() => {
          setIsPinModalOpen(false);
          setIsUnlocked(true);
          setPinError('');
          setPinInput('');
          if (pendingAction) {
            pendingAction();
            setPendingAction(null);
          }
        }, 300);
      } else if (newVal.length === 4) {
        setTimeout(() => {
          setPinError('비밀번호가 올바르지 않습니다. 다시 시도해주세요.');
          setPinInput('');
        }, 300);
      }
    }
  };

  const handlePinBackspace = () => {
    setPinInput(pinInput.slice(0, -1));
  };

  const handleMyPinChangeSubmit = (e) => {
    e.preventDefault();
    const myAccount = allowedUsers.find(u => u.id === user.id);

    if (!myAccount || !myAccount.canChangePin) {
      setPinChangeMessage({ type: 'error', text: 'PIN 변경 권한이 없습니다. 관리자에게 문의하세요.' });
      return;
    }
    if (currentPinInput !== myAccount.pin) {
      setPinChangeMessage({ type: 'error', text: '현재 비밀번호가 틀렸습니다.' });
      return;
    }
    if (!/^\d{4}$/.test(newPinInput)) {
      setPinChangeMessage({ type: 'error', text: '새 비밀번호는 숫자 4자리여야 합니다.' });
      return;
    }
    if (newPinInput !== confirmNewPinInput) {
      setPinChangeMessage({ type: 'error', text: '새 비밀번호 확인이 일치하지 않습니다.' });
      return;
    }

    const updated = allowedUsers.map(u => u.id === user.id ? { ...u, pin: newPinInput } : u);
    setAllowedUsers(updated);
    saveAllowedUsersToDb(updated);

    setPinChangeMessage({ type: 'success', text: '성공적으로 PIN 비밀번호를 변경했습니다!' });
    setCurrentPinInput('');
    setNewPinInput('');
    setConfirmNewPinInput('');
    setTimeout(() => setPinChangeMessage({ type: '', text: '' }), 4000);
  };

  const handleToggleChangePermission = (userId) => {
    if (userId === ADMIN_ID) return; 
    const updated = allowedUsers.map(u => 
      u.id === userId ? { ...u, canChangePin: !u.canChangePin } : u
    );
    setAllowedUsers(updated);
    saveAllowedUsersToDb(updated);
  };

  const handleToggleViewPin = (userId) => {
    setVisiblePins(prev => ({ ...prev, [userId]: !prev[userId] }));
  };

  const handleAdminPinValueChange = (userId, value) => {
    const cleaned = value.replace(/\D/g, '').substring(0, 4);
    setAdminEditingPins(prev => ({ ...prev, [userId]: cleaned }));
  };

  const handleSaveUserPinByAdmin = (userId) => {
    const newPin = adminEditingPins[userId];
    if (!newPin || !/^\d{4}$/.test(newPin)) {
      triggerAlert("입력 오류", "PIN 번호는 숫자 4자리 형식으로 작성해야 합니다.");
      return;
    }
    const updated = allowedUsers.map(u => u.id === userId ? { ...u, pin: newPin } : u);
    setAllowedUsers(updated);
    saveAllowedUsersToDb(updated);

    setAdminEditingPins(prev => {
      const copy = { ...prev };
      delete copy[userId];
      return copy;
    });
    triggerAlert("변경 완료", `${userId}님의 보안 PIN이 성공적으로 변경되었습니다.`);
  };

  const handleSiteNameChange = (e) => {
    e.preventDefault();
    if (!systemSiteName.trim()) return triggerAlert('오류', '사이트 이름을 입력해주세요.');
    setSiteName(systemSiteName);
    saveSettingsToDb(systemSiteName, dashboardLinks, adminPassword);
    triggerAlert('변경 완료', '사이트 이름이 성공적으로 변경되었습니다.');
  };

  const handleAdminPwChange = (e) => {
    e.preventDefault();
    if (currentAdminPw !== adminPassword) return triggerAlert('오류', '현재 비밀번호가 일치하지 않습니다.');
    if (newAdminPw !== confirmNewAdminPw) return triggerAlert('오류', '새 비밀번호 확인이 일치하지 않습니다.');
    if (newAdminPw.length < 4) return triggerAlert('오류', '새 비밀번호는 4자리 이상이어야 합니다.');
    
    setAdminPassword(newAdminPw);
    saveSettingsToDb(siteName, dashboardLinks, newAdminPw);
    setCurrentAdminPw('');
    setNewAdminPw('');
    setConfirmNewAdminPw('');
    triggerAlert('변경 완료', '관리자 로그인 비밀번호가 성공적으로 변경되었습니다.');
  };

  const handleAddLink = (e) => {
    e.preventDefault();
    if (!newLinkName.trim() || !newLinkUrl.trim()) return triggerAlert('오류', '링크 이름과 URL을 모두 입력해주세요.');
    const updated = [...dashboardLinks, { id: Date.now(), name: newLinkName, url: newLinkUrl }];
    setDashboardLinks(updated);
    saveSettingsToDb(siteName, updated, adminPassword);
    setNewLinkName('');
    setNewLinkUrl('');
  };

  const handleDeleteLink = (id) => {
    const updated = dashboardLinks.filter(l => l.id !== id);
    setDashboardLinks(updated);
    saveSettingsToDb(siteName, updated, adminPassword);
  };

  const handleAddUser = (e) => {
    e.preventDefault();
    const idClean = newUserId.trim().toLowerCase();
    const pinClean = newUserPin.trim();

    if (!idClean || !pinClean) {
      triggerAlert('입력 오류', '아이디와 기본 PIN을 입력해주세요.');
      return;
    }
    if (!/^\d{4}$/.test(pinClean)) {
      triggerAlert('입력 오류', '기본 PIN은 숫자 4자리여야 합니다.');
      return;
    }
    if (allowedUsers.some(u => u.id === idClean)) {
      triggerAlert('중복 오류', '이미 존재하는 계정 아이디입니다.');
      return;
    }

    const updated = [
      ...allowedUsers, 
      { id: idClean, pin: pinClean, canChangePin: newUserChangePermission, isAdmin: false }
    ];
    setAllowedUsers(updated);
    saveAllowedUsersToDb(updated);

    setNewUserId('');
    setNewUserPin('');
    setNewUserChangePermission(false);
    triggerAlert('추가 완료', `${idClean}님이 가족 화이트리스트 계정에 정상적으로 등록되었습니다.`);
  };

  const handleDeleteUser = (userId) => {
    if (userId === ADMIN_ID) return;
    triggerConfirm(
      "가족 계정 삭제",
      `정말로 '${userId}' 계정을 아지트 화이트리스트에서 차단 및 삭제하시겠습니까?`,
      () => {
        const updated = allowedUsers.filter(u => u.id !== userId);
        setAllowedUsers(updated);
        saveAllowedUsersToDb(updated);
        triggerAlert("삭제 완료", `${userId} 계정이 정상적으로 삭제되었습니다.`);
      }
    );
  };

  const openAddModal = () => {
    setEditingSchedule(null);
    setFormData({ title: '', date: '', description: '', category: '행사' });
    setIsModalOpen(true);
  };

  const openEditModal = (s) => {
    setEditingSchedule(s);
    setFormData({ title: s.title, date: s.date, description: s.description, category: s.category });
    setIsModalOpen(true);
  };

  const saveSchedule = (e) => {
    e.preventDefault();
    if (!user.isAdmin) return;
    const updated = editingSchedule
      ? schedules.map(s => s.id === editingSchedule.id ? { ...formData, id: s.id } : s)
      : [...schedules, { ...formData, id: Date.now() }];
    
    setSchedules(updated);
    saveSchedulesToDb(updated);
    setIsModalOpen(false);
  };

  const suggestDetails = async () => {
    if (!formData.title) return;
    setIsAiTyping(true);
    const prompt = `'${formData.title}'라는 제목의 가족 일정에 어울리는 상세 설명을 한 문장으로 추천해줘.`;
    const suggestion = await callGemini(prompt, "너는 가족 비서야. 아주 친절하고 따뜻하게 한 문장으로 대답해.");
    setFormData({ ...formData, description: suggestion.replace(/"/g, '') });
    setIsAiTyping(false);
  };

  const generateBriefing = async () => {
    setIsBriefingLoading(true);
    const scheduleContext = schedules.map(s => `${s.date}: ${s.title} (${s.description})`).join(', ');
    const prompt = `현재 가족 일정 목록은 다음과 같아: [${scheduleContext}]. 
    이 일정들을 바탕으로 오늘 날짜(2026년 5월 중순 가정) 기준의 주간 브리핑을 해줘. 
    1. 요약된 일정 2. 주의사항 3. 가족 응원 메시지 순서로 아주 친절하게 말하되, **각 항목당 1문장씩만 작성해서 전체 길이를 매우 짧고 간결하게** 요약해줘.`;
    
    const result = await callGemini(prompt, "너는 따뜻한 가족 상담사이자 비서야. 한국어로 답변해줘.");
    setBriefing(result);
    setIsBriefingLoading(false);
  };

  const saveNotice = (e) => {
    e.preventDefault();
    if (!user.isAdmin) return;
    const today = new Date().toISOString().split('T')[0];
    
    const updated = editingNotice
      ? notices.map(n => n.id === editingNotice.id ? { ...n, title: noticeFormData.title, content: noticeFormData.content } : n)
      : [{ id: Date.now(), title: noticeFormData.title, content: noticeFormData.content, date: today, author: 'Admin' }, ...notices];

    setNotices(updated);
    saveNoticesToDb(updated);
    setIsNoticeModalOpen(false);
    setEditingNotice(null);
    setNoticeFormData({ title: '', content: '' });
  };

  const markNotificationsAsRead = () => {
    if (!user) return;
    const updated = notifications.map(n => n.to === user.id ? { ...n, isRead: true } : n);
    setNotifications(updated);
    saveNotificationsToDb(updated);
  };

  const askAI = async (input) => {
    setChatMessages(prev => [...prev, { role: 'user', text: input }]);
    setIsAiTyping(true);

    const scheduleContext = schedules.map(s => `${s.date}: ${s.title}`).join('\n');
    const userListContext = allowedUsers.map(u => `- ${u.id} (PIN 변경권한: ${u.canChangePin ? '있음' : '없음'})`).join('\n');
    
    const systemPrompt = `너는 '우리 가족 아지트'의 AI 비서야.
    - 가족 멤버 현황:
    ${userListContext}
    - 현재 일정:
    ${scheduleContext}
    
    [기능 실행 규칙]
    사용자가 특정 페이지로 가고 싶어 하면 답변 끝에 반드시 [GOTO:페이지이름]을 붙여줘. 
    - 대시보드/홈: [GOTO:dashboard]
    - 일정 관리: [GOTO:schedule]
    - 멤버 및 보안 관리: [GOTO:security]
    
    예: "보안 관리 페이지로 이동해 드릴게요! [GOTO:security]"
    항상 친절하고 다정하게 존댓말로 대답해줘.`;

    const aiResponse = await callGemini(input, systemPrompt);
    
    if (aiResponse.includes('[GOTO:')) {
      const match = aiResponse.match(/\[GOTO:(\w+)\]/);
      if (match && match[1]) {
        setTimeout(() => {
          if (match[1] === 'schedule') {
            secureAccess('schedule_page', () => setCurrentView('schedule'));
          } else {
            setCurrentView(match[1]);
          }
        }, 1000);
      }
    }

    setChatMessages(prev => [...prev, { role: 'ai', text: aiResponse.replace(/\[GOTO:\w+\]/g, '') }]);
    setIsAiTyping(false);
  };

  const handleChatPointerDown = (e) => {
    dragStartPos.current = {
      startX: e.clientX,
      startY: e.clientY,
      initialOffsetX: chatOffset.x,
      initialOffsetY: chatOffset.y,
      isDragging: false
    };
    
    const handlePointerMove = (moveEvent) => {
      const dx = moveEvent.clientX - dragStartPos.current.startX;
      const dy = moveEvent.clientY - dragStartPos.current.startY;
      
      if (Math.abs(dx) > 5 || Math.abs(dy) > 5) {
        dragStartPos.current.isDragging = true;
        setIsDraggingChat(true);
        setChatOffset({
          x: dragStartPos.current.initialOffsetX + dx,
          y: dragStartPos.current.initialOffsetY + dy
        });
      }
    };

    const handlePointerUp = () => {
      window.removeEventListener('pointermove', handlePointerMove);
      window.removeEventListener('pointerup', handlePointerUp);
      setTimeout(() => setIsDraggingChat(false), 0);
    };

    window.addEventListener('pointermove', handlePointerMove);
    window.addEventListener('pointerup', handlePointerUp);
  };

  const handleTogglePointerDown = (e) => {
    toggleDragStartPos.current = {
      startX: e.clientX,
      startY: e.clientY,
      initialOffsetX: toggleOffset.x,
      initialOffsetY: toggleOffset.y,
      isDragging: false
    };
    
    const handlePointerMove = (moveEvent) => {
      const dx = moveEvent.clientX - toggleDragStartPos.current.startX;
      const dy = moveEvent.clientY - toggleDragStartPos.current.startY;
      
      if (Math.abs(dx) > 5 || Math.abs(dy) > 5) {
        toggleDragStartPos.current.isDragging = true;
        setToggleOffset({
          x: toggleDragStartPos.current.initialOffsetX + dx,
          y: toggleDragStartPos.current.initialOffsetY + dy
        });
      }
    };

    const handlePointerUp = () => {
      window.removeEventListener('pointermove', handlePointerMove);
      window.removeEventListener('pointerup', handlePointerUp);
    };

    window.addEventListener('pointermove', handlePointerMove);
    window.addEventListener('pointerup', handlePointerUp);
  };

  // 알림창 바깥 클릭 시 닫히도록 감지
  useEffect(() => {
    const handleOutsideClick = (e) => {
      if (notiRef.current && !notiRef.current.contains(e.target)) {
        setIsNotiDropdownOpen(false);
      }
    };
    document.addEventListener('mousedown', handleOutsideClick);
    return () => document.removeEventListener('mousedown', handleOutsideClick);
  }, []);

  if (isDbLoading) {
    return (
      <div className="min-h-screen bg-slate-900 flex flex-col items-center justify-center text-white font-sans">
        <div className="w-14 h-14 border-4 border-indigo-500 border-t-transparent rounded-full animate-spin mb-4"></div>
        <p className="text-sm font-semibold tracking-wider animate-pulse">우리 가족 아지트 데이터 로딩 중...</p>
      </div>
    );
  }

  if (isWatchMode) {
    return (
      <div className="min-h-screen bg-slate-900 flex items-center justify-center relative select-none font-sans overflow-hidden">
        <button 
          onPointerDown={handleTogglePointerDown}
          onClick={() => {
            if (toggleDragStartPos.current.isDragging) return;
            setIsWatchMode(false);
          }}
          style={{ transform: `translate(${toggleOffset.x}px, ${toggleOffset.y}px)`, touchAction: 'none' }}
          className="absolute bottom-6 left-6 flex items-center gap-2 text-xs font-bold px-4 py-2 rounded-full bg-slate-800 text-slate-300 hover:bg-slate-700 transition-colors z-[100] shadow-lg"
        >
          <Sparkles size={14} /> 폰/PC 화면으로
        </button>

        {/* 갤럭시 워치 6 베젤 디자인 (300x300, 원형) */}
        <div className="w-[300px] h-[300px] bg-black border-[14px] border-slate-800 rounded-full shadow-[0_0_30px_rgba(0,0,0,0.8)] relative overflow-hidden flex flex-col items-center">
          
          {!user ? (
            <div className="flex flex-col items-center justify-center h-full w-full p-6 text-white pt-8">
              <div className="text-[13px] font-bold mb-4 text-center leading-tight">
                <span className="text-indigo-400">우리 가족 아지트</span>
              </div>
              <form onSubmit={handleLogin} className="w-full space-y-2 flex flex-col items-center">
                <input 
                  type="text" 
                  value={loginId}
                  onChange={(e) => setLoginId(e.target.value)}
                  placeholder="아이디"
                  className="w-[85%] text-center px-3 py-1.5 text-[11px] bg-slate-800 text-white rounded-full border-none focus:ring-1 focus:ring-indigo-500 outline-none"
                  required
                />
                {loginId === ADMIN_ID && (
                  <input 
                    type="password" 
                    value={loginPw}
                    onChange={(e) => setLoginPw(e.target.value)}
                    placeholder="비밀번호"
                    className="w-[85%] text-center px-3 py-1.5 text-[11px] bg-slate-800 text-white rounded-full border-none focus:ring-1 focus:ring-indigo-500 outline-none"
                    required
                  />
                )}
                {loginMessage.text && (
                  <div className={`text-[9px] ${loginMessage.type === 'error' ? 'text-red-400' : 'text-green-400'}`}>
                    {loginMessage.text}
                  </div>
                )}
                <button className="w-[85%] bg-indigo-600 text-white text-[11px] py-2 rounded-full font-bold mt-1">
                  로그인
                </button>
              </form>
            </div>
          ) : (
            <div className="w-full h-full text-white flex flex-col relative">
              {/* 상단 상태표시줄 (시계) */}
              <div className="absolute top-2 left-0 right-0 flex justify-center items-center text-[10px] font-medium text-slate-400 z-10 bg-black/50 backdrop-blur-sm py-0.5">
                 {new Date().toLocaleTimeString('ko-KR', { hour: '2-digit', minute: '2-digit' })}
              </div>

              {/* 워치 PIN 모달 */}
              {isPinModalOpen && (
                <div className="absolute inset-0 bg-black/95 z-50 flex flex-col items-center justify-center pt-4">
                   <div className="text-[10px] text-amber-400 mb-2 flex items-center gap-1">
                     <Lock size={10} /> PIN 입력
                   </div>
                   <div className="flex gap-2 mb-3">
                      {[...Array(4)].map((_, i) => (
                        <div key={i} className={`w-2 h-2 rounded-full ${i < pinInput.length ? 'bg-indigo-500' : 'bg-slate-700'}`} />
                      ))}
                   </div>
                   <div className="grid grid-cols-3 gap-1 w-[160px]">
                      {[1,2,3,4,5,6,7,8,9].map(num => (
                        <button key={num} type="button" onClick={() => handlePinInputChange(num.toString())} className="bg-slate-800/80 text-white h-9 w-full rounded-full text-[13px] font-bold active:bg-slate-600">{num}</button>
                      ))}
                      <button type="button" onClick={() => {setPinInput(''); setIsPinModalOpen(false); setPendingAction(null);}} className="bg-rose-900/40 text-rose-400 h-9 w-full rounded-full text-[9px] font-bold active:bg-rose-900">취소</button>
                      <button type="button" onClick={() => handlePinInputChange('0')} className="bg-slate-800/80 text-white h-9 w-full rounded-full text-[13px] font-bold active:bg-slate-600">0</button>
                      <button type="button" onClick={handlePinBackspace} className="bg-slate-800/80 text-slate-300 h-9 w-full rounded-full text-[9px] font-bold active:bg-slate-600">지움</button>
                   </div>
                </div>
              )}

              {/* 콘텐츠 영역 */}
              <div className="flex-1 overflow-y-auto w-full pt-8 pb-10 px-6 flex flex-col items-center" style={{ scrollbarWidth: 'none', msOverflowStyle: 'none' }}>
                <style>{`::-webkit-scrollbar { display: none; }`}</style>
                
                {currentView === 'dashboard' && (
                   <div className="flex flex-col items-center w-full animate-in fade-in duration-300">
                      <div className="w-10 h-10 bg-indigo-600 rounded-full flex items-center justify-center mb-1 shadow-lg shadow-indigo-900/50">
                        <span className="text-sm font-bold">{user.isAdmin ? 'AD' : user.id.substring(0, 2).toUpperCase()}</span>
                      </div>
                      <h2 className="text-[11px] font-bold text-slate-300 mb-2">반가워요!</h2>
                      
                      <div className="grid grid-cols-3 gap-2 w-full mt-1">
                        <button onClick={() => setCurrentView('schedule')} className="bg-slate-800 hover:bg-slate-700 p-2.5 rounded-2xl flex flex-col items-center gap-1 transition-colors">
                          <Calendar size={16} className="text-indigo-400"/>
                          <span className="text-[8px] font-bold">일정</span>
                        </button>
                        <button onClick={() => setCurrentView('notice')} className="bg-slate-800 hover:bg-slate-700 p-2.5 rounded-2xl flex flex-col items-center gap-1 transition-colors">
                          <Megaphone size={16} className="text-blue-400"/>
                          <span className="text-[8px] font-bold">공지</span>
                        </button>
                        <button onClick={() => secureAccess('watch_chat', () => setCurrentView('chat'))} className="bg-slate-800 hover:bg-slate-700 p-2.5 rounded-2xl flex flex-col items-center gap-1 transition-colors">
                          <MessageSquare size={16} className="text-green-400"/>
                          <span className="text-[8px] font-bold">AI챗봇</span>
                        </button>
                        <button onClick={() => secureAccess('watch_security', () => setCurrentView('security'))} className="bg-slate-800 hover:bg-slate-700 p-2.5 rounded-2xl flex flex-col items-center gap-1 transition-colors">
                          <KeyRound size={16} className="text-amber-400"/>
                          <span className="text-[8px] font-bold">설정</span>
                        </button>
                        <button onClick={() => {
                           if(isUnlocked) setIsUnlocked(false);
                           else secureAccess('watch_unlock', () => {});
                        }} className="bg-slate-800 hover:bg-slate-700 p-2.5 rounded-2xl flex flex-col items-center gap-1 transition-colors">
                          {isUnlocked ? <Unlock size={16} className="text-emerald-400"/> : <Lock size={16} className="text-rose-400"/>}
                          <span className="text-[8px] font-bold">{isUnlocked ? '보안해제' : '보안잠금'}</span>
                        </button>
                        <button onClick={handleLogout} className="bg-slate-800 hover:bg-slate-700 p-2.5 rounded-2xl flex flex-col items-center gap-1 transition-colors">
                          <LogOut size={16} className="text-rose-400"/>
                          <span className="text-[8px] font-bold">종료</span>
                        </button>
                      </div>
                   </div>
                )}

                {currentView === 'schedule' && (
                   <div className="flex flex-col items-center w-full animate-in slide-in-from-right-4 duration-300">
                      <div className="flex items-center gap-2 mb-3 text-indigo-400">
                        <Calendar size={12} /> <span className="text-[11px] font-bold">일정</span>
                      </div>
                      <div className="w-full space-y-2">
                         {!isUnlocked ? (
                            <button onClick={() => secureAccess('watch_schedule', () => {})} className="w-full bg-slate-800/50 rounded-2xl py-6 flex flex-col items-center justify-center border border-slate-700">
                              <Lock size={18} className="mb-2 text-amber-500" />
                              <span className="text-[10px] text-slate-400">눌러서 잠금해제</span>
                            </button>
                         ) : (
                            schedules.slice(0,3).map(s => (
                              <div key={s.id} className="bg-slate-800 p-3 rounded-2xl w-full text-center">
                                <div className="text-[9px] font-bold text-indigo-400 mb-0.5">{s.date.substring(5)}</div>
                                <div className="text-[11px] font-bold truncate">{s.title}</div>
                              </div>
                            ))
                         )}
                         {isUnlocked && schedules.length === 0 && (
                           <div className="text-[10px] text-slate-500 text-center py-4">일정이 없습니다.</div>
                         )}
                      </div>
                      <button onClick={() => setCurrentView('dashboard')} className="mt-4 bg-slate-700 hover:bg-slate-600 px-4 py-1.5 rounded-full text-[10px] font-bold transition-colors">뒤로</button>
                   </div>
                )}

                {currentView === 'notice' && (
                   <div className="flex flex-col items-center w-full animate-in slide-in-from-right-4 duration-300">
                      <div className="flex items-center gap-2 mb-3 text-blue-400">
                        <Megaphone size={12} /> <span className="text-[11px] font-bold">공지사항</span>
                      </div>
                      <div className="w-full space-y-2">
                         {notices.slice(0,3).map(n => (
                           <div key={n.id} className="bg-slate-800 p-3 rounded-2xl w-full text-center">
                             <div className="text-[9px] text-slate-400 mb-0.5">{n.date.substring(5)}</div>
                             <div className="text-[11px] font-bold truncate leading-tight line-clamp-2">{n.title}</div>
                           </div>
                         ))}
                         {notices.length === 0 && (
                           <div className="text-[10px] text-slate-500 text-center py-4">공지가 없습니다.</div>
                         )}
                      </div>
                      <button onClick={() => setCurrentView('dashboard')} className="mt-4 bg-slate-700 hover:bg-slate-600 px-4 py-1.5 rounded-full text-[10px] font-bold transition-colors">뒤로</button>
                   </div>
                )}
                
                {currentView === 'security' && (
                   <div className="flex flex-col w-full h-full animate-in slide-in-from-right-4 duration-300">
                      <div className="flex justify-between items-center mb-2 shrink-0">
                         <div className="flex items-center gap-1 text-amber-500">
                           <KeyRound size={12} /> <span className="text-[11px] font-bold">보안/관리</span>
                         </div>
                         <button onClick={() => setCurrentView('dashboard')} className="text-[9px] bg-slate-700 hover:bg-slate-600 px-2 py-1 rounded transition-colors">뒤로</button>
                      </div>

                      {!isUnlocked ? (
                         <div className="flex-1 flex flex-col items-center justify-center">
                            <button onClick={() => secureAccess('watch_security', () => {})} className="bg-slate-800/50 rounded-2xl py-4 px-6 flex flex-col items-center justify-center border border-slate-700 w-full">
                              <Lock size={18} className="mb-2 text-amber-500" />
                              <span className="text-[10px] text-slate-400">잠금해제 필요</span>
                            </button>
                         </div>
                      ) : (
                         <div className="flex-1 overflow-y-auto space-y-3 pb-8" style={{ scrollbarWidth: 'none' }}>
                            {/* 내 PIN 변경 */}
                            {allowedUsers.find(u => u.id === user.id)?.canChangePin && (
                              <div className="bg-slate-800 p-3 rounded-2xl">
                                 <h4 className="text-[10px] font-bold text-white mb-2">내 PIN 변경</h4>
                                 <form onSubmit={handleMyPinChangeSubmit} className="space-y-1.5">
                                    <input type="password" value={currentPinInput} onChange={e=>setCurrentPinInput(e.target.value.replace(/\D/g,''))} placeholder="현재 PIN (4자리)" className="w-full bg-slate-900 text-white text-[10px] px-2 py-1.5 rounded border border-slate-700 outline-none" maxLength={4}/>
                                    <div className="flex gap-1.5">
                                      <input type="password" value={newPinInput} onChange={e=>setNewPinInput(e.target.value.replace(/\D/g,''))} placeholder="새 PIN" className="w-full bg-slate-900 text-white text-[10px] px-2 py-1.5 rounded border border-slate-700 outline-none" maxLength={4}/>
                                      <input type="password" value={confirmNewPinInput} onChange={e=>setConfirmNewPinInput(e.target.value.replace(/\D/g,''))} placeholder="확인" className="w-full bg-slate-900 text-white text-[10px] px-2 py-1.5 rounded border border-slate-700 outline-none" maxLength={4}/>
                                    </div>
                                    <button type="submit" className="w-full bg-indigo-600 text-white text-[9px] py-1.5 rounded font-bold mt-1">변경 적용</button>
                                    {pinChangeMessage.text && <div className="text-[8px] text-center text-amber-400 mt-1">{pinChangeMessage.text}</div>}
                                 </form>
                              </div>
                            )}

                            {/* 관리자 전용 메뉴 */}
                            {user.isAdmin && (
                              <>
                                <div className="bg-slate-800 p-3 rounded-2xl">
                                   <h4 className="text-[10px] font-bold text-white mb-2">사이트 이름 변경</h4>
                                   <form onSubmit={handleSiteNameChange} className="flex gap-1.5">
                                      <input type="text" value={systemSiteName} onChange={e=>setSystemSiteName(e.target.value)} className="flex-1 bg-slate-900 text-white text-[10px] px-2 py-1.5 rounded border border-slate-700 outline-none"/>
                                      <button type="submit" className="bg-slate-700 hover:bg-slate-600 text-white text-[9px] px-2.5 rounded font-bold">저장</button>
                                   </form>
                                </div>

                                <div className="bg-slate-800 p-3 rounded-2xl">
                                   <h4 className="text-[10px] font-bold text-white mb-2 flex items-center justify-between">
                                     멤버 PIN 조회
                                   </h4>
                                   <div className="space-y-1">
                                      {allowedUsers.map(u => (
                                        <div key={u.id} className="flex justify-between items-center bg-slate-900 px-2 py-1.5 rounded">
                                           <span className="text-[9px] text-slate-300 font-medium">{u.id}</span>
                                           <div className="flex items-center gap-2">
                                             <span className="text-[10px] font-mono font-bold text-indigo-300 w-6 text-center">
                                               {visiblePins[u.id] ? u.pin : '••••'}
                                             </span>
                                             <button onClick={() => handleToggleViewPin(u.id)} className="text-slate-400 hover:text-white p-1">
                                               {visiblePins[u.id] ? <EyeOff size={10}/> : <Eye size={10}/>}
                                              </button>
                                           </div>
                                        </div>
                                      ))}
                                   </div>
                                </div>
                              </>
                            )}
                         </div>
                      )}
                   </div>
                )}

                {currentView === 'chat' && (
                   <div className="flex flex-col w-full h-full animate-in slide-in-from-right-4 duration-300">
                      <div className="flex justify-between items-center mb-2 shrink-0">
                         <div className="flex items-center gap-1 text-green-400">
                           <Zap size={12} /> <span className="text-[11px] font-bold">AI 비서</span>
                         </div>
                         <button onClick={() => setCurrentView('dashboard')} className="text-[9px] bg-slate-700 hover:bg-slate-600 px-2 py-1 rounded transition-colors">뒤로</button>
                      </div>
                      <div className="flex-1 overflow-y-auto space-y-2 mb-2 pr-1 pb-4" style={{ scrollbarWidth: 'none' }}>
                         {chatMessages.map((msg, i) => (
                           <div key={i} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
                             <div className={`max-w-[90%] px-2.5 py-2 rounded-xl text-[9px] leading-relaxed ${msg.role === 'user' ? 'bg-indigo-600 text-white rounded-tr-none' : 'bg-slate-700 text-slate-200 rounded-tl-none'}`}>
                               {msg.text}
                             </div>
                           </div>
                         ))}
                         {isAiTyping && <div className="text-[9px] text-slate-400 ml-1">AI 입력중...</div>}
                         <div ref={chatEndRef}></div>
                      </div>
                      <form onSubmit={(e) => { e.preventDefault(); if(chatInput) { askAI(chatInput); setChatInput(''); } }} className="flex gap-1.5 shrink-0 mt-auto pb-4 pt-1 bg-black">
                        <input type="text" value={chatInput} onChange={e=>setChatInput(e.target.value)} className="flex-1 bg-slate-800 text-white text-[10px] px-3 py-2 rounded-full outline-none border border-slate-700" placeholder="질문하기..."/>
                        <button type="submit" disabled={isAiTyping} className="bg-indigo-600 hover:bg-indigo-700 text-white w-8 h-8 flex items-center justify-center rounded-full disabled:opacity-50 shrink-0">
                          <Send size={12}/>
                        </button>
                      </form>
                   </div>
                )}
              </div>
            </div>
          )}
        </div>
      </div>
    );
  }

  if (!user) {
    return (
      <div className="min-h-screen flex items-center justify-center p-4 transition-all duration-500 bg-slate-50 overflow-hidden">
        {/* 워치 모드 토글 버튼 */}
        <button 
          onPointerDown={handleTogglePointerDown}
          onClick={() => {
            if (toggleDragStartPos.current.isDragging) return;
            setIsWatchMode(true);
          }}
          style={{ transform: `translate(${toggleOffset.x}px, ${toggleOffset.y}px)`, touchAction: 'none' }}
          className="absolute bottom-6 left-6 flex items-center gap-2 text-xs font-bold px-4 py-2 rounded-full bg-white shadow-lg border border-slate-200 text-slate-700 hover:bg-slate-50 transition-colors z-50 backdrop-blur-sm"
        >
          <Watch size={14} /> 워치 화면으로
        </button>

        <div className="bg-white p-8 rounded-3xl shadow-xl w-full max-w-md border border-slate-100">
          <div className="text-center mb-8">
            <div className="bg-indigo-600 w-16 h-16 rounded-2xl mb-4 flex items-center justify-center mx-auto shadow-lg shadow-indigo-200">
              <Home className="text-white w-8 h-8" />
            </div>
            <h1 className="text-2xl font-bold text-slate-800">{siteName}</h1>
            <p className="text-slate-500 text-sm mt-2 font-medium">Gemini AI가 관리하는 가족 공간</p>
          </div>

          <form onSubmit={handleLogin} className="space-y-4">
            <div>
              <label className="block text-sm font-semibold text-slate-700 mb-1.5">아이디</label>
              <input 
                type="text" 
                value={loginId}
                onChange={(e) => setLoginId(e.target.value)}
                className="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition-all"
                placeholder="아이디"
                required
              />
            </div>
            {loginId === ADMIN_ID && (
              <div>
                <label className="block text-sm font-semibold text-slate-700 mb-1.5">비밀번호</label>
                <input 
                  type="password" 
                  value={loginPw}
                  onChange={(e) => setLoginPw(e.target.value)}
                  className="w-full px-4 py-3 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition-all"
                  placeholder="비밀번호"
                  required
                />
              </div>
            )}
            
            {loginMessage.text && (
              <div className={`text-xs p-3 rounded-xl flex items-center justify-center text-center gap-1 font-medium ${loginMessage.type === 'error' ? 'bg-red-50 text-red-600' : 'bg-green-50 text-green-600'}`}>
                {loginMessage.text}
              </div>
            )}

            <button className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3.5 rounded-xl transition-all flex items-center justify-center gap-1 shadow-md shadow-indigo-900/50">
              입장 <ChevronRight size={18} />
            </button>
          </form>
        </div>
      </div>
    );
  }

  // 현재 로그인한 사용자의 실시간 정보 가져오기
  const myLiveAccount = allowedUsers.find(u => u.id === user.id) || { canChangePin: false };
  const displayName = user.isAdmin ? 'Admin' : user.id;

  return (
    <div className="min-h-screen bg-slate-50 flex flex-col md:flex-row pb-16 md:pb-0 relative overflow-hidden">
      {/* 폰/PC 모드용 워치 토글 버튼 */}
      <button 
        onPointerDown={handleTogglePointerDown}
        onClick={() => {
          if (toggleDragStartPos.current.isDragging) return;
          setIsWatchMode(true);
        }}
        style={{ transform: `translate(${toggleOffset.x}px, ${toggleOffset.y}px)`, touchAction: 'none' }}
        className="hidden md:flex absolute bottom-6 left-6 items-center gap-2 text-xs font-bold px-4 py-2 rounded-full bg-white shadow-lg border border-slate-200 text-slate-700 hover:bg-slate-50 transition-colors z-[100]"
      >
        <Watch size={14} /> 워치 화면으로
      </button>

      {/* Desktop Sidebar */}
      <aside className={`w-full md:w-64 bg-white border-r border-slate-200 flex-col z-20 hidden md:flex`}>
        <div className="p-6 border-b border-slate-100 flex items-center gap-3">
          <div className="w-10 h-10 bg-indigo-100 rounded-xl flex items-center justify-center">
            <Sparkles className="text-indigo-600 w-6 h-6" />
          </div>
          <h2 className="font-bold text-slate-800 leading-tight">{siteName}</h2>
        </div>

        <nav className="flex-1 p-4 space-y-2">
          <button 
            onClick={() => setCurrentView('dashboard')} 
            className={`w-full flex items-center gap-3 px-4 py-3 rounded-xl transition-all ${currentView === 'dashboard' ? 'bg-indigo-50 text-indigo-700 font-bold' : 'text-slate-500 hover:bg-slate-50'}`}
          >
            <Home size={20} /> 대시보드
          </button>
          
          <button 
            onClick={() => setCurrentView('notice')} 
            className={`w-full flex items-center gap-3 px-4 py-3 rounded-xl transition-all ${currentView === 'notice' ? 'bg-indigo-50 text-indigo-700 font-bold' : 'text-slate-500 hover:bg-slate-50'}`}
          >
            <Megaphone size={20} /> 공지사항
          </button>
          
          <button 
            onClick={() => secureAccess('schedule_nav', () => setCurrentView('schedule'))} 
            className={`w-full flex items-center justify-between px-4 py-3 rounded-xl transition-all ${currentView === 'schedule' ? 'bg-indigo-50 text-indigo-700 font-bold' : 'text-slate-500 hover:bg-slate-50'}`}
          >
            <span className="flex items-center gap-3">
              <Calendar size={20} /> 일정 관리
            </span>
            {!isUnlocked && <Lock size={14} className="text-amber-500" />}
          </button>
          
          <button 
            onClick={() => secureAccess('security_nav', () => setCurrentView('security'))} 
            className={`w-full flex items-center justify-between px-4 py-3 rounded-xl transition-all ${currentView === 'security' ? 'bg-indigo-50 text-indigo-700 font-bold' : 'text-slate-500 hover:bg-slate-50'}`}
          >
            <span className="flex items-center gap-3">
              <KeyRound size={20} /> {user.isAdmin ? '가족 & 보안 관리' : '내 보안 설정'}
            </span>
            {!isUnlocked && <Lock size={14} className="text-amber-500" />}
          </button>
        </nav>

        <div className="p-4 border-t border-slate-100">
          <div className="flex items-center gap-3 p-3 bg-slate-50 rounded-2xl mb-3">
            <div className="w-8 h-8 rounded-full bg-indigo-600 flex items-center justify-center font-bold text-white text-xs">
              {displayName.substring(0, 2).toUpperCase()}
            </div>
            <div className="flex-1 overflow-hidden">
              <p className="text-xs font-bold text-slate-800 truncate">{displayName}</p>
              <p className="text-[10px] text-slate-400">{user.isAdmin ? '총괄 관리자' : '가족 멤버'}</p>
            </div>
          </div>
          <button onClick={handleLogout} className="w-full flex items-center justify-center gap-2 text-xs font-bold text-slate-500 hover:text-red-500 transition-colors p-2">
            <LogOut size={14} /> 로그아웃
          </button>
        </div>
      </aside>

      {/* Mobile Bottom Nav */}
      <nav className="md:hidden fixed bottom-0 left-0 right-0 bg-white border-t border-slate-200 flex justify-around items-center p-2 pb-safe z-50 shadow-[0_-5px_15px_-10px_rgba(0,0,0,0.1)]">
        <button onClick={() => setCurrentView('dashboard')} className={`flex flex-col items-center gap-1 p-2 w-16 rounded-xl transition-colors ${currentView === 'dashboard' ? 'text-indigo-600 bg-indigo-50' : 'text-slate-400 hover:bg-slate-50'}`}>
          <Home size={20} />
          <span className="text-[10px] font-bold">홈</span>
        </button>
        <button onClick={() => setCurrentView('notice')} className={`flex flex-col items-center gap-1 p-2 w-16 rounded-xl transition-colors ${currentView === 'notice' ? 'text-indigo-600 bg-indigo-50' : 'text-slate-400 hover:bg-slate-50'}`}>
          <Megaphone size={20} />
          <span className="text-[10px] font-bold">공지</span>
        </button>
        <button onClick={() => secureAccess('schedule_nav', () => setCurrentView('schedule'))} className={`flex flex-col items-center gap-1 p-2 w-16 rounded-xl transition-colors relative ${currentView === 'schedule' ? 'text-indigo-600 bg-indigo-50' : 'text-slate-400 hover:bg-slate-50'}`}>
          <Calendar size={20} />
          <span className="text-[10px] font-bold">일정</span>
          {!isUnlocked && <Lock size={10} className="absolute top-1.5 right-3 text-amber-500" />}
        </button>
        <button onClick={() => secureAccess('security_nav', () => setCurrentView('security'))} className={`flex flex-col items-center gap-1 p-2 w-16 rounded-xl transition-colors relative ${currentView === 'security' ? 'text-indigo-600 bg-indigo-50' : 'text-slate-400 hover:bg-slate-50'}`}>
          <KeyRound size={20} />
          <span className="text-[10px] font-bold">보안</span>
          {!isUnlocked && <Lock size={10} className="absolute top-1.5 right-3 text-amber-500" />}
        </button>
        <button onClick={handleLogout} className="flex flex-col items-center gap-1 p-2 w-16 rounded-xl text-rose-400 hover:bg-rose-50 transition-colors">
          <LogOut size={20} />
          <span className="text-[10px] font-bold">종료</span>
        </button>
      </nav>

      <main className="flex-1 flex flex-col min-w-0 pb-16 md:pb-0">
        {/* Header with locking status */}
        <header className="h-16 bg-white border-b border-slate-200 px-6 md:px-8 flex items-center justify-between sticky top-0 z-10">
          <h3 className="font-bold text-slate-700 uppercase tracking-wide text-xs">
            {currentView === 'security' ? 'Security & Accounts' : currentView}
          </h3>
          
          <div className="flex items-center gap-4">
            <button 
              onClick={() => {
                if (isUnlocked) {
                  setIsUnlocked(false);
                } else {
                  secureAccess('header_unlock', () => {});
                }
              }}
              className={`flex items-center gap-1.5 px-3 py-1.5 rounded-full text-xs font-bold border transition-all ${
                isUnlocked 
                  ? 'bg-green-50 text-green-700 border-green-200 hover:bg-green-100' 
                  : 'bg-amber-50 text-amber-700 border-amber-200 hover:bg-amber-100'
              }`}
            >
              {isUnlocked ? (
                <>
                  <Unlock size={14} className="text-green-600 animate-pulse" />
                  <span>개인정보 해제됨</span>
                </>
              ) : (
                <>
                  <Lock size={14} className="text-amber-600" />
                  <span>개인정보 잠겨있음</span>
                </>
              )}
            </button>

            <div className="relative" ref={notiRef}>
              <button onClick={() => { setIsNotiDropdownOpen(!isNotiDropdownOpen); markNotificationsAsRead(); }} className="p-2 text-slate-400 hover:text-indigo-600 relative">
                <Bell size={20} />
                {notifications.some(n => n.to === user.id && !n.isRead) && (
                  <span className="absolute top-1.5 right-1.5 w-2 h-2 bg-red-500 rounded-full border-2 border-white"></span>
                )}
              </button>
              
              {isNotiDropdownOpen && (
                <div className="absolute right-0 mt-2 w-72 bg-white rounded-2xl shadow-xl border border-slate-100 overflow-hidden z-50">
                  <div className="p-3 border-b border-slate-100 bg-slate-50 font-bold text-sm text-slate-800">
                    내 알림
                  </div>
                  <div className="max-h-64 overflow-y-auto">
                    {notifications.filter(n => n.to === user.id).length === 0 ? (
                      <div className="p-4 text-center text-xs text-slate-400">새로운 알림이 없습니다.</div>
                    ) : (
                      notifications.filter(n => n.to === user.id).map(noti => (
                        <div key={noti.id} className="p-3 border-b border-slate-50 hover:bg-slate-50 flex flex-col gap-1">
                          <div className="flex items-center justify-between">
                            <span className="text-[10px] font-bold text-indigo-600">Admin</span>
                            <span className="text-[10px] text-slate-400">{noti.date}</span>
                          </div>
                          <p className="text-xs text-slate-700">{noti.message}</p>
                        </div>
                      ))
                    )}
                  </div>
                </div>
              )}
            </div>
          </div>
        </header>

        <div className="p-8 overflow-y-auto flex-1">
          {currentView === 'dashboard' && (
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
              <div className="lg:col-span-2 space-y-6">
                <div className="bg-gradient-to-br from-indigo-600 to-violet-700 p-8 rounded-3xl text-white shadow-xl shadow-indigo-100 relative overflow-hidden">
                  <div className="relative z-10">
                    <h2 className="text-xl md:text-2xl font-bold mb-2">오늘도 행복한 하루 되세요, {displayName}님! 🍀</h2>
                    <p className="opacity-80 text-sm">가족 간의 안전하고 소중한 약속들이 지켜지는 공간입니다.</p>
                  </div>
                  <Sparkles className="absolute -right-4 -bottom-4 w-32 h-32 opacity-10 rotate-12" />
                </div>
                
                {/* 동적 링크 목록 렌더링 */}
                <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
                  {dashboardLinks.map(link => (
                    <a 
                      key={link.id}
                      href={link.url} 
                      target="_blank" 
                      rel="noopener noreferrer"
                      className="bg-white p-4 md:p-5 rounded-2xl md:rounded-3xl border border-slate-200 shadow-sm flex items-center justify-between hover:bg-slate-50 transition-all group cursor-pointer block hover:border-indigo-200 hover:shadow-md"
                    >
                      <div className="flex items-center gap-4 truncate pr-4">
                        <div className="w-10 h-10 md:w-12 md:h-12 bg-indigo-50 rounded-xl flex items-center justify-center shrink-0 group-hover:scale-110 transition-transform">
                          <LinkIcon size={18} className="text-indigo-600 md:hidden" />
                          <span className="text-xl hidden md:block">🔗</span>
                        </div>
                        <div className="truncate">
                          <h4 className="font-bold text-slate-800 text-sm md:text-lg truncate group-hover:text-indigo-600 transition-colors">{link.name}</h4>
                          <p className="text-[10px] md:text-xs text-slate-500 mt-0.5 truncate">{link.url}</p>
                        </div>
                      </div>
                      <div className="w-8 h-8 md:w-10 md:h-10 shrink-0 rounded-full bg-slate-50 flex items-center justify-center group-hover:bg-indigo-600 group-hover:text-white transition-colors text-slate-400 shadow-sm">
                        <ChevronRight size={18} />
                      </div>
                    </a>
                  ))}
                </div>

                {/* AI 주간 브리핑 카드 */}
                <div className="bg-white p-6 rounded-3xl border border-indigo-100 shadow-sm relative overflow-hidden min-h-[180px]">
                  <div className="flex items-center justify-between mb-4">
                    <h4 className="font-bold text-slate-800 flex items-center gap-2">
                      <Zap size={18} className="text-indigo-600" /> AI 주간 브리핑
                    </h4>
                    {isUnlocked && (
                      <button 
                        onClick={generateBriefing} 
                        disabled={isBriefingLoading}
                        className="p-2 hover:bg-slate-50 rounded-full transition-colors disabled:opacity-50"
                      >
                        <RefreshCw size={16} className={isBriefingLoading ? "animate-spin" : ""} />
                      </button>
                    )}
                  </div>

                  {isUnlocked ? (
                    <div className="text-sm text-slate-600 leading-relaxed bg-slate-50 p-4 rounded-2xl min-h-[100px] whitespace-pre-wrap">
                      {isBriefingLoading ? (
                        <div className="flex items-center gap-2 animate-pulse">
                          <div className="w-2 h-2 bg-indigo-400 rounded-full"></div>AI가 일정을 분석하는 중...
                        </div>
                      ) : briefing || "새로고침 아이콘을 눌러 이번 주 브리핑을 받아보세요!"}
                    </div>
                  ) : (
                    <div className="absolute inset-0 bg-white/75 backdrop-blur-md flex flex-col items-center justify-center p-4 text-center z-10">
                      <Lock size={28} className="text-amber-500 mb-2" />
                      <p className="text-sm font-bold text-slate-800">보안을 위해 AI 브리핑이 잠겨있습니다.</p>
                      <p className="text-xs text-slate-400 mt-1 mb-3">개인정보 보호용 계정 고유 PIN 번호를 입력해야 확인할 수 있습니다.</p>
                      <button 
                        onClick={() => secureAccess('view_briefing', () => {})} 
                        className="bg-indigo-600 hover:bg-indigo-700 text-white text-xs font-bold px-4 py-2 rounded-xl transition-all shadow-md"
                      >
                        PIN 번호 입력 및 잠금해제
                      </button>
                    </div>
                  )}
                </div>

                {/* 다가오는 일정 목록 */}
                <div className="bg-white p-6 rounded-3xl border border-slate-200 relative overflow-hidden min-h-[200px]">
                  <div className="flex items-center justify-between mb-6">
                    <h4 className="font-bold text-slate-800">🗓️ 다가오는 주요 일정</h4>
                  </div>

                  {isUnlocked ? (
                    <div className="space-y-4">
                      {schedules.slice(0, 3).map(s => (
                        <div key={s.id} className="flex items-center gap-4 p-4 hover:bg-slate-50 rounded-2xl border border-transparent hover:border-slate-100 transition-all">
                          <div className="w-12 h-12 rounded-xl bg-slate-100 flex flex-col items-center justify-center shrink-0">
                            <span className="text-sm font-black text-slate-700">{s.date.split('-')[2]}일</span>
                          </div>
                          <div className="flex-1">
                            <h5 className="font-bold text-slate-800 text-sm">{s.title}</h5>
                            <p className="text-xs text-slate-500 truncate">{s.description}</p>
                          </div>
                          <span className={`text-[10px] font-bold px-2 py-1 rounded-lg ${s.category === '가사' ? 'bg-emerald-50 text-emerald-600' : 'bg-indigo-50 text-indigo-600'}`}>
                            {s.category}
                          </span>
                        </div>
                      ))}
                    </div>
                  ) : (
                    <div className="absolute inset-0 bg-white/75 backdrop-blur-md flex flex-col items-center justify-center p-4 text-center z-10">
                      <Lock size={28} className="text-amber-500 mb-2" />
                      <p className="text-sm font-bold text-slate-800">가족 일정이 잠겨있습니다.</p>
                      <p className="text-xs text-slate-400 mt-1 mb-3">상세 일정 정보 보호를 위해 PIN 입력이 필요합니다.</p>
                      <button 
                        onClick={() => secureAccess('view_schedules', () => {})} 
                        className="bg-indigo-600 hover:bg-indigo-700 text-white text-xs font-bold px-4 py-2 rounded-xl transition-all shadow-md"
                      >
                        일정 확인하기
                      </button>
                    </div>
                  )}
                </div>
              </div>

              {/* 가족 멤버 가이드 */}
              <div className="space-y-6">
                <div className="bg-white p-6 rounded-3xl border border-slate-200">
                  <h4 className="font-bold text-slate-800 mb-4">멤버 현황 및 상태</h4>
                  <div className="space-y-3">
                    {allowedUsers.map(u => (
                      <div key={u.id} className="flex items-center gap-3">
                        <div className="w-8 h-8 rounded-full bg-indigo-50 flex items-center justify-center text-[10px] font-bold text-indigo-600">
                          {u.isAdmin ? 'AD' : u.id.substring(0, 2).toUpperCase()}
                        </div>
                        <div className="flex-1 min-w-0">
                          <p className="text-xs font-bold text-slate-700 flex items-center gap-1.5">
                            {u.isAdmin ? 'Admin' : u.id}
                            {u.isAdmin && <ShieldCheck size={14} className="text-indigo-600" />}
                          </p>
                          <p className="text-[10px] text-slate-400">
                            PIN 변경권한: {u.canChangePin ? '허용됨' : '차단됨'}
                          </p>
                        </div>
                      </div>
                    ))}
                  </div>
                </div>
              </div>
            </div>
          )}

          {currentView === 'notice' && (
            <div className="bg-white rounded-3xl border border-slate-200 overflow-hidden shadow-sm">
              <div className="p-6 border-b border-slate-100 flex items-center justify-between bg-slate-50/50">
                <div className="flex items-center gap-3">
                  <div className="p-2.5 bg-blue-50 rounded-xl text-blue-600">
                    <Megaphone size={20} />
                  </div>
                  <div>
                    <h4 className="font-bold text-slate-800 text-lg">가족 공지사항</h4>
                    <p className="text-xs text-slate-400">관리자가 등록한 주요 소식을 확인하세요.</p>
                  </div>
                </div>
                {user.isAdmin && (
                  <button onClick={() => { setEditingNotice(null); setNoticeFormData({title:'', content:''}); setIsNoticeModalOpen(true); }} className="bg-slate-800 text-white text-sm font-bold px-4 py-2 rounded-xl flex items-center gap-2">
                    <Plus size={16} /> 새 공지
                  </button>
                )}
              </div>
              <div className="divide-y divide-slate-100">
                {notices.map(n => (
                  <div key={n.id} className="p-6 hover:bg-slate-50/30 transition-colors">
                    <div className="flex items-center gap-3 mb-2">
                      <span className="px-2 py-1 bg-red-50 text-red-600 text-[10px] font-bold rounded-lg border border-red-100">{n.author}</span>
                      <span className="text-xs text-slate-400 font-medium">{n.date}</span>
                    </div>
                    <h5 className="font-bold text-slate-800 mb-2">{n.title}</h5>
                    <p className="text-sm text-slate-600 whitespace-pre-wrap leading-relaxed">{n.content}</p>
                    {user.isAdmin && (
                      <div className="mt-4 flex gap-2">
                        <button onClick={() => { setEditingNotice(n); setNoticeFormData({title:n.title, content:n.content}); setIsNoticeModalOpen(true); }} className="text-xs px-3 py-1.5 bg-slate-100 hover:bg-slate-200 text-slate-600 font-bold rounded-lg transition-colors">수정</button>
                        <button onClick={() => {
                          const updated = notices.filter(x => x.id !== n.id);
                          setNotices(updated);
                          saveNoticesToDb(updated);
                        }} className="text-xs px-3 py-1.5 bg-red-50 hover:bg-red-100 text-red-600 font-bold rounded-lg transition-colors">삭제</button>
                      </div>
                    )}
                  </div>
                ))}
                {notices.length === 0 && (
                  <div className="p-8 text-center text-slate-400 text-sm">등록된 공지사항이 없습니다.</div>
                )}
              </div>
            </div>
          )}

          {currentView === 'schedule' && isUnlocked && (
            <div className="bg-white rounded-3xl border border-slate-200 overflow-hidden">
              <div className="p-6 border-b border-slate-100 flex items-center justify-between bg-slate-50/50">
                <h4 className="font-bold text-slate-800 text-lg">일정 리스트</h4>
                {user.isAdmin && (
                  <button onClick={openAddModal} className="bg-indigo-600 text-white text-sm font-bold px-4 py-2 rounded-xl flex items-center gap-2 shadow-lg shadow-indigo-100">
                    <Plus size={18} /> 일정 추가
                  </button>
                )}
              </div>
              <div className="divide-y divide-slate-100">
                {schedules.map(s => (
                  <div key={s.id} className="p-6 flex flex-col sm:flex-row sm:items-center gap-4 hover:bg-slate-50/50 transition-colors">
                    <div className="flex-1">
                      <div className="flex items-center gap-3 mb-1">
                        <span className="text-xs font-bold text-indigo-600">{s.date}</span>
                        <h5 className="font-bold text-slate-800">{s.title}</h5>
                      </div>
                      <p className="text-sm text-slate-500">{s.description}</p>
                    </div>
                    {user.isAdmin && (
                      <div className="flex items-center gap-1">
                        <button onClick={() => openEditModal(s)} className="p-2 text-slate-400 hover:text-indigo-600 hover:bg-white rounded-lg"><Edit2 size={16} /></button>
                        <button onClick={() => {
                          const updated = schedules.filter(x => x.id !== s.id);
                          setSchedules(updated);
                          saveSchedulesToDb(updated);
                        }} className="p-2 text-slate-400 hover:text-red-500 hover:bg-white rounded-lg"><Trash2 size={16} /></button>
                      </div>
                    )}
                  </div>
                ))}
              </div>
            </div>
          )}

          {currentView === 'security' && (
            <div className="space-y-6 max-w-4xl">
              <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                
                {/* 시스템 및 관리자 설정 (Admin 전용) */}
                {user.isAdmin && (
                  <div className="bg-white rounded-3xl border border-slate-200 shadow-sm p-6 lg:col-span-2">
                    <div className="flex items-center gap-3 border-b border-slate-100 pb-4 mb-4">
                      <div className="p-2.5 bg-slate-800 rounded-xl text-white">
                        <Sparkles size={20} />
                      </div>
                      <div>
                        <h4 className="font-bold text-slate-800">시스템 설정</h4>
                        <p className="text-xs text-slate-400">사이트 이름 및 관리자 로그인 비밀번호를 변경합니다.</p>
                      </div>
                    </div>

                    <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
                      {/* 사이트 이름 설정 */}
                      <form onSubmit={handleSiteNameChange} className="space-y-4">
                        <div>
                          <label className="block text-xs font-bold text-slate-500 mb-1">사이트 이름</label>
                          <div className="flex gap-2">
                            <input 
                              type="text" 
                              value={systemSiteName}
                              onChange={(e) => setSystemSiteName(e.target.value)}
                              placeholder="예: 우리 가족 아지트"
                              className="flex-1 px-4 py-2 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition"
                              required
                            />
                            <button type="submit" className="bg-slate-800 hover:bg-slate-900 text-white text-xs font-bold px-4 rounded-xl shadow-md transition shrink-0">
                              변경
                            </button>
                          </div>
                        </div>
                      </form>

                      {/* 관리자 비밀번호 설정 */}
                      <form onSubmit={handleAdminPwChange} className="space-y-4">
                        <div>
                          <label className="block text-xs font-bold text-slate-500 mb-1">현재 관리자 로그인 비밀번호</label>
                          <input 
                            type="password" 
                            value={currentAdminPw}
                            onChange={(e) => setCurrentAdminPw(e.target.value)}
                            className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition"
                            required
                          />
                        </div>
                        <div className="flex gap-2">
                          <div className="flex-1">
                            <label className="block text-xs font-bold text-slate-500 mb-1">새 비밀번호</label>
                            <input 
                              type="password" 
                              value={newAdminPw}
                              onChange={(e) => setNewAdminPw(e.target.value)}
                              className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition"
                              required
                            />
                          </div>
                          <div className="flex-1">
                            <label className="block text-xs font-bold text-slate-500 mb-1">새 비밀번호 확인</label>
                            <input 
                              type="password" 
                              value={confirmNewAdminPw}
                              onChange={(e) => setConfirmNewAdminPw(e.target.value)}
                              className="w-full px-4 py-2 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition"
                              required
                            />
                          </div>
                        </div>
                        <button type="submit" className="w-full bg-slate-800 hover:bg-slate-900 text-white text-xs font-bold py-2.5 rounded-xl shadow-md transition">
                          관리자 로그인 비밀번호 변경 완료
                        </button>
                      </form>
                    </div>
                  </div>
                )}

                {/* 바로가기 링크 관리 (Admin 전용) */}
                {user.isAdmin && (
                  <div className="bg-white rounded-3xl border border-slate-200 shadow-sm p-6 lg:col-span-2">
                    <div className="flex items-center justify-between border-b border-slate-100 pb-4 mb-4">
                      <div className="flex items-center gap-3">
                        <div className="p-2.5 bg-blue-50 rounded-xl text-blue-600">
                          <LinkIcon size={20} />
                        </div>
                        <div>
                          <h4 className="font-bold text-slate-800">대시보드 바로가기 링크 관리</h4>
                          <p className="text-xs text-slate-400">대시보드 상단에 노출될 외부 사이트 링크를 설정합니다.</p>
                        </div>
                      </div>
                    </div>

                    <form onSubmit={handleAddLink} className="flex flex-col sm:flex-row gap-3 mb-6 bg-slate-50 p-4 rounded-2xl border border-slate-100">
                      <div className="flex-1">
                        <input 
                          type="text" 
                          value={newLinkName}
                          onChange={(e) => setNewLinkName(e.target.value)}
                          placeholder="링크 이름 (예: 구글 사이트)"
                          className="w-full px-4 py-2 text-sm bg-white border border-slate-200 rounded-xl focus:ring-2 focus:ring-blue-500 outline-none"
                          required
                        />
                      </div>
                      <div className="flex-[2]">
                        <input 
                          type="url" 
                          value={newLinkUrl}
                          onChange={(e) => setNewLinkUrl(e.target.value)}
                          placeholder="URL (https://...)"
                          className="w-full px-4 py-2 text-sm bg-white border border-slate-200 rounded-xl focus:ring-2 focus:ring-blue-500 outline-none"
                          required
                        />
                      </div>
                      <button type="submit" className="bg-blue-600 hover:bg-blue-700 text-white px-6 py-2 rounded-xl text-sm font-bold shadow-sm whitespace-nowrap">
                        추가
                      </button>
                    </form>

                    <div className="space-y-2">
                      {dashboardLinks.map(link => (
                        <div key={link.id} className="flex flex-row items-center justify-between p-3 border border-slate-100 rounded-xl hover:bg-slate-50">
                          <div className="flex items-center gap-3 overflow-hidden pr-4">
                            <LinkIcon size={16} className="text-slate-400 shrink-0" />
                            <div className="truncate">
                              <p className="text-sm font-bold text-slate-700 truncate">{link.name}</p>
                              <p className="text-[10px] text-slate-400 truncate">{link.url}</p>
                            </div>
                          </div>
                          <button onClick={() => handleDeleteLink(link.id)} className="p-2 text-rose-400 hover:text-rose-600 hover:bg-rose-50 rounded-lg shrink-0">
                            <Trash2 size={16} />
                          </button>
                        </div>
                      ))}
                      {dashboardLinks.length === 0 && (
                        <p className="text-center text-sm text-slate-400 py-4 bg-slate-50 rounded-xl border border-dashed border-slate-200">등록된 링크가 없습니다.</p>
                      )}
                    </div>
                  </div>
                )}

                {/* 1단계: 사용자 본인 PIN 변경 섹션 */}
                <div className="bg-white rounded-3xl border border-slate-200 shadow-sm p-6">
                  <div className="flex items-center gap-3 border-b border-slate-100 pb-4 mb-4">
                    <div className="p-2.5 bg-indigo-50 rounded-xl text-indigo-600">
                      <KeyRound size={20} />
                    </div>
                    <div>
                      <h4 className="font-bold text-slate-800">내 보안 PIN 변경</h4>
                      <p className="text-xs text-slate-400">본인 계정의 개인정보 잠금 해제용 PIN 번호입니다.</p>
                    </div>
                  </div>

                  {myLiveAccount.canChangePin ? (
                    <form onSubmit={handleMyPinChangeSubmit} className="space-y-4 pt-2">
                      <div>
                        <label className="block text-xs font-bold text-slate-500 mb-1">현재 PIN 번호</label>
                        <input 
                          type="password" 
                          maxLength="4"
                          pattern="\d{4}"
                          value={currentPinInput}
                          onChange={(e) => setCurrentPinInput(e.target.value.replace(/\D/g, ''))}
                          placeholder="현재 4자리"
                          className="w-full px-4 py-2.5 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition"
                          required
                        />
                      </div>
                      <div className="grid grid-cols-2 gap-4">
                        <div>
                          <label className="block text-xs font-bold text-slate-500 mb-1">새 PIN 번호</label>
                          <input 
                            type="password" 
                            maxLength="4"
                            pattern="\d{4}"
                            value={newPinInput}
                            onChange={(e) => setNewPinInput(e.target.value.replace(/\D/g, ''))}
                            placeholder="숫자 4자리"
                            className="w-full px-4 py-2.5 bg-slate-50 border border-slate-200 rounded-xl text-center focus:ring-2 focus:ring-indigo-500 outline-none transition"
                            required
                          />
                        </div>
                        <div>
                          <label className="block text-xs font-bold text-slate-500 mb-1">새 PIN 확인</label>
                          <input 
                            type="password" 
                            maxLength="4"
                            pattern="\d{4}"
                            value={confirmNewPinInput}
                            onChange={(e) => setConfirmNewPinInput(e.target.value.replace(/\D/g, ''))}
                            placeholder="한 번 더 입력"
                            className="w-full px-4 py-2.5 bg-slate-50 border border-slate-200 rounded-xl text-center focus:ring-2 focus:ring-indigo-500 outline-none transition"
                            required
                          />
                        </div>
                      </div>

                      {pinChangeMessage.text && (
                        <div className={`text-xs p-3 rounded-xl flex items-center gap-2 ${
                          pinChangeMessage.type === 'error' ? 'bg-red-50 text-red-600' : 'bg-green-50 text-green-600'
                        }`}>
                          <div className={`w-1.5 h-1.5 rounded-full ${pinChangeMessage.type === 'error' ? 'bg-red-500' : 'bg-green-500'}`}></div>
                          {pinChangeMessage.text}
                        </div>
                      )}

                      <button 
                        type="submit" 
                        className="w-full bg-indigo-600 hover:bg-indigo-700 text-white text-xs font-bold py-3 rounded-xl shadow-md transition-colors"
                      >
                        내 PIN 변경 완료
                      </button>
                    </form>
                  ) : (
                    <div className="bg-red-50 border border-red-100 p-6 rounded-2xl text-center">
                      <Lock size={32} className="text-red-500 mx-auto mb-2" />
                      <p className="text-sm font-bold text-red-700">보안 PIN 직접 변경 권한이 없습니다</p>
                      <p className="text-xs text-red-500 mt-1 leading-relaxed">
                        개인정보 보호 정책에 따라 관리자(jay)가 귀하의 계정에 PIN 변경 권한을 차단했습니다. 변경을 원하시면 관리자에게 권한 부여를 요청해 주세요.
                      </p>
                    </div>
                  )}
                </div>

                {/* 2단계: 신규 가족 계정 추가 (Admin 전용) */}
                {user.isAdmin && (
                  <div className="bg-white rounded-3xl border border-slate-200 shadow-sm p-6">
                    <div className="flex items-center gap-3 border-b border-slate-100 pb-4 mb-4">
                      <div className="p-2.5 bg-emerald-50 rounded-xl text-emerald-600">
                        <Users size={20} />
                      </div>
                      <div>
                        <h4 className="font-bold text-slate-800">신규 가족 계정 등록</h4>
                        <p className="text-xs text-slate-400">아지트에 입장할 수 있는 새로운 구성원을 추가합니다.</p>
                      </div>
                    </div>

                    <form onSubmit={handleAddUser} className="space-y-4 pt-2">
                      <div>
                        <label className="block text-xs font-bold text-slate-500 mb-1">계정 ID</label>
                        <input 
                          type="text" 
                          placeholder="예: mom, sister" 
                          value={newUserId}
                          onChange={(e) => setNewUserId(e.target.value)}
                          className="w-full px-4 py-2.5 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition"
                          required
                        />
                      </div>
                      <div>
                        <label className="block text-xs font-bold text-slate-500 mb-1">초기 보안 PIN (숫자 4자리)</label>
                        <input 
                          type="text" 
                          maxLength="4"
                          placeholder="예: 1234" 
                          value={newUserPin}
                          onChange={(e) => setNewUserPin(e.target.value.replace(/\D/g, ''))}
                          className="w-full px-4 py-2.5 bg-slate-50 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none transition"
                          required
                        />
                      </div>
                      
                      <div className="flex items-center justify-between bg-slate-50 p-3 rounded-xl border border-slate-100">
                        <div>
                          <p className="text-xs font-bold text-slate-700">비밀번호 변경 권한 자동 부여</p>
                          <p className="text-[10px] text-slate-400">등록 후 사용자가 스스로 PIN을 바꿀 수 있게 합니다.</p>
                        </div>
                        <button
                          type="button"
                          onClick={() => setNewUserChangePermission(!newUserChangePermission)}
                          className={`w-11 h-6 flex items-center rounded-full p-1 duration-300 cursor-pointer ${newUserChangePermission ? 'bg-indigo-600' : 'bg-slate-300'}`}
                        >
                          <div className={`bg-white w-4 h-4 rounded-full shadow-md transform duration-300 ${newUserChangePermission ? 'translate-x-5' : 'translate-x-0'}`} />
                        </button>
                      </div>

                      <button 
                        type="submit" 
                        className="w-full bg-slate-800 hover:bg-slate-900 text-white text-xs font-bold py-3 rounded-xl shadow-md transition"
                      >
                        신규 가족 등록 완료
                      </button>
                    </form>
                  </div>
                )}
              </div>

              {/* 3단계: 전체 가족 계정 PIN 조회 / 일괄 변경 / 권한 제어 패널 (Admin 전용) */}
              {user.isAdmin && (
                <div className="bg-white rounded-3xl border border-slate-200 overflow-hidden shadow-sm">
                  <div className="p-6 border-b border-slate-100 bg-slate-50/50">
                    <h4 className="font-bold text-slate-800">가족 계정 보안 일괄 관리 패널</h4>
                    <p className="text-xs text-slate-400 mt-1">
                      모든 계정의 보안 PIN 비밀번호를 조회/수정할 수 있으며, 스스로 비밀번호를 변경할 수 있는 권한을 직접 컨트롤합니다.
                    </p>
                  </div>

                  <div className="divide-y divide-slate-100">
                    {allowedUsers.map(u => {
                      const isVisible = visiblePins[u.id] || false;
                      const customEditValue = adminEditingPins[u.id] !== undefined ? adminEditingPins[u.id] : '';
                      const isCurrentlyEditing = adminEditingPins[u.id] !== undefined;

                      return (
                        <div key={u.id} className="flex flex-col">
                          <div className="p-6 flex flex-col md:flex-row md:items-center justify-between gap-4 hover:bg-slate-50/30 transition-colors">
                            <div className="flex items-center gap-3">
                              <div className="w-10 h-10 rounded-full bg-slate-100 flex items-center justify-center font-bold text-slate-600">
                                {u.isAdmin ? 'AD' : u.id.substring(0,2).toUpperCase()}
                              </div>
                              <div>
                                <div className="flex items-center gap-2">
                                  <h5 className="font-bold text-slate-800">{u.isAdmin ? 'Admin' : u.id}</h5>
                                  {u.isAdmin && <span className="bg-indigo-50 text-indigo-600 text-[10px] font-black px-2 py-0.5 rounded">총괄관리자</span>}
                                </div>
                                <p className="text-xs text-slate-400 mt-0.5">
                                  상태: {u.canChangePin ? '직접 변경 가능' : '직접 변경 불가능 (관리자 통제)'}
                                </p>
                              </div>
                            </div>

                            {/* 보기, 수정 및 권한 통제 행렬 */}
                            <div className="flex flex-wrap items-center gap-4 md:gap-6 mt-3 md:mt-0 w-full md:w-auto border-t border-slate-100 pt-3 md:border-none md:pt-0">
                              
                              {/* PIN 보기 & 임시 편집 구조 */}
                              <div className="flex items-center gap-2 bg-slate-50 px-3 py-1.5 rounded-xl border border-slate-100">
                                <span className="text-xs font-bold text-slate-500">보안 PIN:</span>
                              {isCurrentlyEditing ? (
                                <input 
                                  type="text" 
                                  maxLength="4"
                                  value={customEditValue}
                                  onChange={(e) => handleAdminPinValueChange(u.id, e.target.value)}
                                  className="w-16 bg-white border border-indigo-200 rounded px-1 text-center text-xs font-bold py-0.5 outline-none focus:ring-1 focus:ring-indigo-500"
                                  placeholder="새 4자리"
                                />
                              ) : (
                                <span className="text-xs font-mono font-bold text-slate-800 tracking-wider">
                                  {isVisible ? u.pin : '••••'}
                                </span>
                              )}

                                <button
                                  type="button"
                                  disabled={u.id === ADMIN_ID}
                                  onClick={() => handleToggleChangePermission(u.id)}
                                  className={`w-11 h-6 flex items-center rounded-full p-1 duration-300 ${u.id === ADMIN_ID ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer'} ${u.canChangePin ? 'bg-indigo-600' : 'bg-slate-300'}`}
                                >
                                  <div className={`bg-white w-4 h-4 rounded-full shadow-md transform duration-300 ${u.canChangePin ? 'translate-x-5' : 'translate-x-0'}`} />
                                </button>
                              </div>

                              <div className="flex items-center gap-2 ml-auto md:ml-0">
                                {/* 알림 발송 버튼 */}
                                {u.id !== ADMIN_ID && (
                                  <button 
                                    onClick={() => setSendingNotiTo(sendingNotiTo === u.id ? null : u.id)}
                                    className={`px-3 py-1.5 text-[10px] font-bold rounded-lg border transition ${sendingNotiTo === u.id ? 'bg-indigo-50 text-indigo-600 border-indigo-200' : 'bg-white text-slate-500 border-slate-200 hover:bg-slate-50'}`}
                                  >
                                    알림 보내기
                                  </button>
                                )}

                                {/* 삭제 버튼 */}
                                {u.id !== ADMIN_ID && (
                                  <button 
                                    onClick={() => handleDeleteUser(u.id)} 
                                    className="p-2 text-slate-400 hover:text-red-500 hover:bg-slate-50 rounded-lg transition"
                                    title="가족 계정 삭제"
                                  >
                                    <Trash2 size={16} />
                                  </button>
                                )}
                              </div>
                            </div>
                          </div>
                          
                          {/* 인라인 알림 전송 폼 */}
                          {sendingNotiTo === u.id && (
                            <div className="p-4 bg-indigo-50/50 border-t border-indigo-50 flex gap-2 items-center mx-6 mb-4 rounded-xl">
                              <span className="text-xs font-bold text-indigo-600 shrink-0">메시지 전송 :</span>
                              <input 
                                type="text" 
                                value={notiMessage}
                                onChange={(e) => setNotiMessage(e.target.value)}
                                placeholder={`${u.id}님에게 보낼 알림 내용 입력`}
                                className="flex-1 px-3 py-2 text-xs rounded-lg border border-indigo-200 focus:ring-2 focus:ring-indigo-400 outline-none"
                              />
                              <button 
                                onClick={() => {
                                  if (!notiMessage.trim()) return;
                                  const updated = [{ id: Date.now(), to: u.id, message: notiMessage, date: new Date().toLocaleDateString(), isRead: false }, ...notifications];
                                  setNotifications(updated);
                                  saveNotificationsToDb(updated);
                                  setNotiMessage('');
                                  setSendingNotiTo(null);
                                  triggerAlert('전송 완료', `${u.id}님에게 알림을 성공적으로 보냈습니다.`);
                                }}
                                className="bg-indigo-600 text-white px-4 py-2 rounded-lg text-xs font-bold shadow-sm hover:bg-indigo-700"
                              >
                                보내기
                              </button>
                            </div>
                          )}
                        </div>
                      );
                    })}
                  </div>
                </div>
              )}
            </div>
          )}
        </div>
      </main>

      {/* 1. 일정 등록/수정 모달 */}
      {isModalOpen && (
        <div className="fixed inset-0 bg-slate-900/40 backdrop-blur-sm z-50 flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-lg rounded-3xl shadow-2xl overflow-hidden animate-in fade-in zoom-in-95 duration-200">
            <div className="p-6 border-b border-slate-100 flex justify-between items-center bg-indigo-50/30">
              <h4 className="font-bold text-slate-800">{editingSchedule ? '일정 수정' : '새 일정 등록'}</h4>
              <button onClick={() => setIsModalOpen(false)} className="text-slate-400 font-bold text-2xl">×</button>
            </div>
            <form onSubmit={saveSchedule} className="p-6 space-y-4">
              <div>
                <label className="block text-xs font-bold text-slate-500 mb-1.5 uppercase tracking-wide">일정 제목</label>
                <div className="flex gap-2">
                  <input 
                    type="text" 
                    value={formData.title}
                    onChange={(e) => setFormData({...formData, title: e.target.value})}
                    className="flex-1 px-4 py-2.5 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none" 
                    required 
                  />
                  <button 
                    type="button" 
                    onClick={suggestDetails}
                    className="bg-indigo-50 text-indigo-600 px-3 py-2 rounded-xl hover:bg-indigo-100 transition-colors flex items-center gap-1 text-xs font-bold"
                  >
                    <Sparkles size={14} /> AI 추천
                  </button>
                </div>
              </div>
              <div className="grid grid-cols-2 gap-4">
                <div>
                  <label className="block text-xs font-bold text-slate-500 mb-1.5 uppercase">날짜</label>
                  <input type="date" value={formData.date} onChange={(e) => setFormData({...formData, date: e.target.value})} className="w-full px-4 py-2.5 border rounded-xl" required />
                </div>
                <div>
                  <label className="block text-xs font-bold text-slate-500 mb-1.5 uppercase">카테고리</label>
                  <select value={formData.category} onChange={(e) => setFormData({...formData, category: e.target.value})} className="w-full px-4 py-2.5 border rounded-xl">
                    <option value="행사">🎂 행사</option>
                    <option value="가사">🧹 가사</option>
                    <option value="외식">🍱 외식</option>
                  </select>
                </div>
              </div>
              <div>
                <label className="block text-xs font-bold text-slate-500 mb-1.5 uppercase">상세 설명</label>
                <textarea 
                  rows="3"
                  value={formData.description}
                  onChange={(e) => setFormData({...formData, description: e.target.value})}
                  className="w-full px-4 py-2.5 border rounded-xl focus:ring-2 focus:ring-indigo-500 resize-none outline-none"
                ></textarea>
              </div>
              <div className="pt-4 flex gap-2">
                <button type="button" onClick={() => setIsModalOpen(false)} className="flex-1 bg-slate-100 font-bold py-3 rounded-xl text-slate-600">취소</button>
                <button type="submit" className="flex-1 bg-indigo-600 text-white font-bold py-3 rounded-xl shadow-lg shadow-indigo-100">등록</button>
              </div>
            </form>
          </div>
        </div>
      )}

      {/* 공지사항 모달 */}
      {isNoticeModalOpen && (
        <div className="fixed inset-0 bg-slate-900/40 backdrop-blur-sm z-50 flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-lg rounded-3xl shadow-2xl overflow-hidden animate-in fade-in zoom-in-95 duration-200">
            <div className="p-6 border-b border-slate-100 flex justify-between items-center bg-blue-50/30">
              <h4 className="font-bold text-slate-800">{editingNotice ? '공지사항 수정' : '새 공지사항 등록'}</h4>
              <button onClick={() => setIsNoticeModalOpen(false)} className="text-slate-400 font-bold text-2xl">×</button>
            </div>
            <form onSubmit={saveNotice} className="p-6 space-y-4">
              <div>
                <label className="block text-xs font-bold text-slate-500 mb-1.5 uppercase tracking-wide">공지 제목</label>
                <input 
                  type="text" 
                  value={noticeFormData.title}
                  onChange={(e) => setNoticeFormData({...noticeFormData, title: e.target.value})}
                  className="w-full px-4 py-2.5 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 outline-none" 
                  required 
                />
              </div>
              <div>
                <label className="block text-xs font-bold text-slate-500 mb-1.5 uppercase">상세 내용</label>
                <textarea 
                  rows="5"
                  value={noticeFormData.content}
                  onChange={(e) => setNoticeFormData({...noticeFormData, content: e.target.value})}
                  className="w-full px-4 py-2.5 border border-slate-200 rounded-xl focus:ring-2 focus:ring-indigo-500 resize-none outline-none"
                  required
                ></textarea>
              </div>
              <div className="pt-4 flex gap-2">
                <button type="button" onClick={() => setIsNoticeModalOpen(false)} className="flex-1 bg-slate-100 font-bold py-3 rounded-xl text-slate-600">취소</button>
                <button type="submit" className="flex-1 bg-slate-800 hover:bg-slate-900 text-white font-bold py-3 rounded-xl shadow-lg">등록</button>
              </div>
            </form>
          </div>
        </div>
      )}

      {/* 2. 개인정보 열람용 PIN 코드 입력 모달 (계정 독립 적용) */}
      {isPinModalOpen && (
        <div className="fixed inset-0 bg-slate-900/60 backdrop-blur-md z-50 flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-sm rounded-3xl shadow-2xl p-6 border border-slate-100 flex flex-col items-center animate-in fade-in zoom-in-95 duration-200">
            
            <div className="w-16 h-16 bg-amber-50 border border-amber-200 rounded-full flex items-center justify-center text-amber-500 mb-4 animate-bounce">
              <Lock size={28} />
            </div>

            <h3 className="text-lg font-bold text-slate-800">개인정보 잠금 해제</h3>
            <p className="text-xs text-slate-400 text-center mt-1 mb-6">
              가족 개인정보 보호를 위해 <span className="font-bold text-indigo-600">[{displayName}]</span> 계정의 보안 PIN을 입력하세요.
            </p>

            <form onSubmit={handlePinSubmit} className="w-full flex flex-col items-center">
              <div className="flex justify-center gap-4 mb-6">
                {[...Array(4)].map((_, index) => (
                  <div 
                    key={index}
                    className={`w-4 h-4 rounded-full border-2 transition-all ${
                      index < pinInput.length 
                        ? 'bg-indigo-600 border-indigo-600 scale-110' 
                        : 'bg-slate-50 border-slate-300'
                    }`}
                  />
                ))}
              </div>

              {pinError && (
                <p className="text-xs text-red-500 font-medium mb-4 text-center">{pinError}</p>
              )}

              {/* 스마트 키패드 */}
              <div className="grid grid-cols-3 gap-3 w-full max-w-[240px] mb-6">
                {[1, 2, 3, 4, 5, 6, 7, 8, 9].map((num) => (
                  <button
                    key={num}
                    type="button"
                    onClick={() => handlePinInputChange(num.toString())}
                    className="w-14 h-14 rounded-full bg-slate-50 hover:bg-indigo-50 text-slate-800 font-bold text-lg flex items-center justify-center border border-slate-100 active:scale-95 transition-all"
                  >
                    {num}
                  </button>
                ))}
                <button
                  type="button"
                  onClick={() => {
                    setPinInput('');
                    setIsPinModalOpen(false);
                    setPendingAction(null);
                  }}
                  className="w-14 h-14 rounded-full hover:bg-red-50 text-red-500 font-bold text-xs flex items-center justify-center active:scale-95 transition-all"
                >
                  취소
                </button>
                <button
                  type="button"
                  onClick={() => handlePinInputChange('0')}
                  className="w-14 h-14 rounded-full bg-slate-50 hover:bg-indigo-50 text-slate-800 font-bold text-lg flex items-center justify-center border border-slate-100 active:scale-95 transition-all"
                >
                  0
                </button>
                <button
                  type="button"
                  onClick={handlePinBackspace}
                  className="w-14 h-14 rounded-full hover:bg-slate-200 text-slate-600 font-bold text-xs flex items-center justify-center active:scale-95 transition-all"
                >
                  지우기
                </button>
              </div>
            </form>
          </div>
        </div>
      )}

      {/* 3. AI 챗봇 컴포넌트 */}
      <div 
        className="fixed bottom-8 right-8 z-40 flex flex-col items-end"
        style={{ transform: `translate(${chatOffset.x}px, ${chatOffset.y}px)` }}
      >
        {isChatOpen && isUnlocked && (
          <div className="w-80 md:w-96 h-[500px] bg-white rounded-3xl shadow-2xl border border-slate-200 flex flex-col overflow-hidden mb-4 animate-in slide-in-from-bottom-5">
            <div className="bg-indigo-600 p-4 text-white flex items-center justify-between">
              <div className="flex items-center gap-3">
                <div className="w-8 h-8 bg-white/20 rounded-lg flex items-center justify-center"><Zap size={16} /></div>
                <div>
                  <h4 className="text-xs font-bold uppercase tracking-widest">Gemini Family AI</h4>
                  <p className="text-[10px] opacity-70">실시간 스케줄 관리 중</p>
                </div>
              </div>
              <button onClick={() => setIsChatOpen(false)} className="hover:bg-white/10 p-1 rounded-lg">×</button>
            </div>

            <div className="flex-1 overflow-y-auto p-4 space-y-4 bg-slate-50/50">
              {chatMessages.map((msg, i) => (
                <div key={i} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
                  <div className={`max-w-[85%] px-4 py-2.5 rounded-2xl text-[13px] leading-relaxed ${
                    msg.role === 'user' 
                      ? 'bg-indigo-600 text-white rounded-tr-none' 
                      : 'bg-white text-slate-700 border border-slate-200 rounded-tl-none shadow-sm'
                  }`}>
                    {msg.text}
                  </div>
                </div>
              ))}
              {isAiTyping && (
                <div className="flex justify-start">
                  <div className="bg-white px-4 py-2.5 rounded-2xl rounded-tl-none shadow-sm flex gap-1">
                    <div className="w-1 h-1 bg-indigo-400 rounded-full animate-bounce"></div>
                    <div className="w-1 h-1 bg-indigo-400 rounded-full animate-bounce [animation-delay:0.2s]"></div>
                    <div className="w-1 h-1 bg-indigo-400 rounded-full animate-bounce [animation-delay:0.4s]"></div>
                  </div>
                </div>
              )}
              <div ref={chatEndRef}></div>
            </div>

            <form onSubmit={(e) => { e.preventDefault(); if(chatInput) { askAI(chatInput); setChatInput(''); } }} className="p-3 bg-white border-t flex gap-2">
              <input 
                type="text" 
                value={chatInput}
                onChange={(e) => setChatInput(e.target.value)}
                placeholder="비서에게 말하기..."
                className="flex-1 bg-slate-100 border-none rounded-xl px-4 py-2 text-[13px] outline-none"
              />
              <button disabled={isAiTyping} className="bg-indigo-600 text-white p-2 rounded-xl hover:bg-indigo-700 transition-colors disabled:opacity-50">
                <Send size={16} />
              </button>
            </form>
          </div>
        )}

        {/* 챗봇 트리거 버튼 */}
        <button 
          onPointerDown={handleChatPointerDown}
          onClick={(e) => {
            if (dragStartPos.current?.isDragging) return;
            if (isChatOpen) {
              setIsChatOpen(false);
            } else {
              secureAccess('open_chat', () => setIsChatOpen(true));
            }
          }}
          style={{ touchAction: 'none' }}
          className={`w-14 h-14 rounded-full shadow-2xl flex items-center justify-center transition-all duration-300 relative ${
            isChatOpen && isUnlocked 
              ? 'bg-slate-800 text-white rotate-90' 
              : 'bg-indigo-600 text-white hover:scale-110'
          }`}
        >
          {isChatOpen && isUnlocked ? (
            <Edit2 size={24} />
          ) : (
            <>
              <MessageSquare size={24} />
              {!isUnlocked && (
                <span className="absolute -top-1 -right-1 bg-amber-500 border-2 border-slate-50 w-5 h-5 rounded-full flex items-center justify-center text-[10px]">
                  <Lock size={10} className="text-white" />
                </span>
              )}
            </>
          )}
        </button>
      </div>

      {/* 4. 커스텀 알림/확인 대화상자 모달 (인앱 팝업) */}
      {modalAlert.isOpen && (
        <div className="fixed inset-0 bg-slate-900/40 backdrop-blur-sm z-[100] flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-sm rounded-3xl shadow-2xl p-6 border border-slate-100 flex flex-col items-center animate-in fade-in zoom-in-95 duration-200">
            <div className={`w-12 h-12 rounded-full flex items-center justify-center mb-4 ${
              modalAlert.type === 'confirm' ? 'bg-rose-50 text-rose-500 border border-rose-100' : 'bg-indigo-50 text-indigo-500 border border-indigo-100'
            }`}>
              {modalAlert.type === 'confirm' ? <Trash2 size={22} /> : <ShieldCheck size={22} />}
            </div>
            <h4 className="text-base font-bold text-slate-800 mb-2">{modalAlert.title}</h4>
            <p className="text-xs text-slate-500 text-center mb-6 leading-relaxed whitespace-pre-wrap">{modalAlert.message}</p>
            
            <div className="flex gap-2 w-full">
              {modalAlert.type === 'confirm' ? (
                <>
                  <button 
                    type="button" 
                    onClick={() => setModalAlert(prev => ({ ...prev, isOpen: false }))} 
                    className="flex-1 bg-slate-100 hover:bg-slate-200 text-slate-600 font-bold py-2.5 rounded-xl text-xs transition"
                  >
                    취소
                  </button>
                  <button 
                    type="button" 
                    onClick={() => {
                      if (modalAlert.onConfirm) modalAlert.onConfirm();
                      setModalAlert(prev => ({ ...prev, isOpen: false }));
                    }} 
                    className="flex-1 bg-rose-600 hover:bg-rose-700 text-white font-bold py-2.5 rounded-xl text-xs transition shadow-md shadow-rose-100"
                  >
                    삭제하기
                  </button>
                </>
              ) : (
                <button 
                  type="button" 
                  onClick={() => setModalAlert(prev => ({ ...prev, isOpen: false }))} 
                  className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-2.5 rounded-xl text-xs transition shadow-md shadow-indigo-100"
                >
                  확인
                </button>
              )}
            </div>
          </div>
        </div>
      )}
    </div>
  );
};

export default FamilySite;
