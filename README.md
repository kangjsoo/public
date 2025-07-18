import React, { useState, useEffect, useCallback, useMemo, useRef } from 'react';
import { PieChart, Pie, Cell, BarChart, Bar, XAxis, YAxis, Tooltip, Legend, ResponsiveContainer } from 'recharts';
import DOMPurify from 'dompurify';
// Firebase imports
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, getDocs, query, deleteDoc, doc, writeBatch, onSnapshot } from 'firebase/firestore';

// --- Firebase Initialization ---
// These global variables are provided by the Canvas environment.
// Do NOT modify these lines.
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
// Corrected: Ensure initialAuthToken uses __initial_auth_token correctly
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null; 

// Initialize Firebase app
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const dbFirestore = getFirestore(app);

// --- Helper Components ---

// Custom Modal Component for alerts and confirmations
const CustomModal = ({ children, onClose }) => (
  <div className="fixed inset-0 bg-black bg-opacity-60 flex items-center justify-center z-50 p-4 animate-fade-in-fast">
    <div className="bg-white p-8 rounded-2xl shadow-2xl max-w-md w-full relative">
      <button onClick={onClose} className="absolute top-3 right-3 text-gray-500 hover:text-gray-800 transition-colors">
        <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><line x1="18" y1="6" x2="6" y2="18"></line><line x1="6" y1="6" x2="18" y2="18"></line></svg>
      </button>
      {children}
    </div>
  </div>
);

// Tooltip Component for question explanations
const HelpTooltip = ({ text }) => {
  const [visible, setVisible] = useState(false);
  return (
    <div className="relative inline-block ml-2">
      <button 
        onMouseEnter={() => setVisible(true)}
        onMouseLeave={() => setVisible(false)}
        // Added hover scale effect
        className="text-gray-400 hover:text-purple-600 transition-colors hover:scale-110 transition-transform"
      >
        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><circle cx="12" cy="12" r="10"></circle><line x1="12" y1="16" x2="12" y2="12"></line><line x1="12" y1="8" x2="12.01" y2="8"></line></svg>
      </button>
      {visible && (
        <div className="absolute bottom-full left-1/2 -translate-x-1/2 mb-2 w-64 bg-gray-800 text-white text-sm rounded-lg p-3 shadow-lg z-10 animate-fade-in-fast" style={{ wordBreak: 'keep-all' }}>
          {text}
        </div>
      )}
    </div>
  );
};


// --- Database Helper (Firestore) ---
const db = {
  // Add a new submission to Firestore
  async addSubmission(submission, userId) {
    if (!userId) {
      console.error("Firestore: userId is not defined for addSubmission.");
      return;
    }
    // Store data in a private collection for the user
    const submissionsCollectionRef = collection(dbFirestore, `artifacts/${appId}/users/${userId}/submissions`);
    await addDoc(submissionsCollectionRef, submission);
  },
  // Get all submissions from Firestore for a specific user
  // Note: For real-time updates, onSnapshot is used directly in AdminPage.
  // This function is kept for consistency if a one-time fetch is ever needed.
  async getAllSubmissions(userId) {
    if (!userId) {
      console.error("Firestore: userId is not defined for getAllSubmissions.");
      return [];
    }
    const submissionsCollectionRef = collection(dbFirestore, `artifacts/${appId}/users/${userId}/submissions`);
    const querySnapshot = await getDocs(submissionsCollectionRef);
    const submissions = querySnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
    return submissions;
  },
  // Clear all submissions for a specific user
  async clearAllSubmissions(userId) {
    if (!userId) {
      console.error("Firestore: userId is not defined for clearAllSubmissions.");
      return;
    }
    const submissionsCollectionRef = collection(dbFirestore, `artifacts/${appId}/users/${userId}/submissions`);
    const querySnapshot = await getDocs(submissionsCollectionRef);
    const batch = writeBatch(dbFirestore); // Use a batch for multiple deletes
    querySnapshot.docs.forEach((d) => {
      batch.delete(doc(dbFirestore, `artifacts/${appId}/users/${userId}/submissions`, d.id));
    });
    await batch.commit();
  },
};


// --- Application Data ---

// MBTI questions for the test (글 길이 줄임)
const mbtiQuestions = [
    { id: 1, type: 'E/I', question: "산책 갈 준비 됐냥? 문소리만 나도...", options: [{ text: "현관에서 '빨리 나가자!' 온몸으로 표현해요.", type: "E" }, { text: "산책줄 들 때까지 조용히 기다려요.", type: "I" }], tooltip: "강아지가 외부 자극에 얼마나 즉각적이고 활발하게 반응하는지 확인하는 질문입니다." },
    { id: 2, type: 'E/I', question: "새 강아지 친구를 만났을 때, 첫 반응은?", options: [{ text: "꼬리 흔들며 다가가서 놀자고 재롱 부려요.", type: "E" }, { text: "멀리서 지켜보다가 조심스럽게 탐색해요.", type: "I" }], tooltip: "사회적 상황에서 에너지를 얻는지(외향적), 아니면 신중하게 관찰하며 에너지를 쓰는지(내향적) 알아봅니다." },
    { id: 3, type: 'E/I', question: "혼자 집에 있을 때, 우리 강아지는?", options: [{ text: "가족 없어서 외로워하고 문 앞에서 기다려요.", type: "E" }, { text: "혼자만의 시간을 즐기며 편안하게 쉬거나 잠을 자요.", type: "I" }], tooltip: "혼자 있는 시간에 대한 반응을 통해 사회적 상호작용의 필요성 정도를 파악합니다." },
    { id: 4, 'type': 'S/N', question: "주인이 간식 봉지 열 때, 우리 강아지 눈빛은?", options: [{ text: "간식 봉지 '실물'에 눈 고정, 냄새와 모양에 집중해요.", type: "S" }, { text: "봉지 소리만으로 이미 간식 파티 상상해요.", type: "N" }], tooltip: "정보를 받아들일 때, 오감으로 확인된 '사실'에 집중하는지, 아니면 소리 같은 단서로 '가능성'을 상상하는지 알아봅니다." },
    { id: 5, type: 'S/N', question: "처음 가는 놀이터 도착! 우리 강아지는...", options: [{ text: "바닥 냄새부터 킁킁 맡으며 실시간으로 파악해요.", type: "S" }, { text: "전체 분위기 훑어보고 '어디서 뛰어볼까?' 큰 그림을 그려요.", type: "N" }], tooltip: "새로운 환경을 탐색할 때, 구체적인 사실(냄새)부터 파악하는지, 전체적인 맥락과 가능성을 먼저 보는지 확인합니다." },
    { id: 6, type: 'S/N', question: "변화가 생겼을 때 우리 강아지는?", options: [{ text: "기존 방식을 유지하거나 익숙한 것에 안정을 찾아요.", type: "S" }, { text: "새로운 가능성 탐색하거나 호기심을 보여요.", type: "N" }], tooltip: "예상치 못한 변화에 대해, 익숙하고 현실적인 것을 선호하는지, 새로운 가능성에 더 흥미를 느끼는지 알아봅니다." },
    { id: 7, type: 'T/F', question: "주인이 슬퍼 보일 때, 우리 강아지 행동은?", options: [{ text: "주인 슬픔을 '문제'로 인식, 해결하려 장난감 가져다줘요.", type: "T" }, { text: "제 감정 읽고 옆에 와서 위로해주거나 핥아줘요.", type: "F" }], tooltip: "결정을 내리거나 반응할 때, 객관적인 해결책을 찾으려 하는지(사고형), 아니면 감정적인 공감과 조화를 우선하는지(감정형) 봅니다." },
    { id: 8, type: 'T/F', question: "다른 강아지와 싸움 났을 때, 우리 강아지는?", options: [{ text: "옳고 그름 따지듯, 영역/권리 주장하며 단호하게 대처해요.", type: "T" }, { text: "상대방 기분 살피며 싸움 피하거나 빨리 화해하려 해요.", type: "F" }], tooltip: "갈등 상황에서 논리적 원칙에 따라 행동하는지, 관계의 조화를 유지하려 하는지 파악합니다." },
    { id: 9, type: 'T/F', question: "주인이 꾸중할 때, 우리 강아지는?", options: [{ text: "왜 꾸중 듣는지 '논리적' 이유 파악하고 다음 생각해요.", type: "T" }, { text: "주인 표정/목소리 톤에 따라 감정적으로 풀 죽거나 불안해해요.", type: "F" }], tooltip: "부정적인 피드백에 대해, 원인 분석에 집중하는지, 감정적으로 먼저 반응하는지 알아봅니다." },
    { id: 10, type: 'J/P', question: "하루 일과 중 '밥 먹는 시간'에 대한 생각은?", options: [{ text: "규칙적인 시간에 맞춰 '밥 시간 됐잖아!' 정확해요.", type: "J" }, { text: "그때그때 배고프면 먹고, 아니면 나중에 먹어도 돼요.", type: "P" }], tooltip: "생활 방식이 계획적이고 체계적인지(판단형), 아니면 유연하고 즉흥적인지(인식형) 알아봅니다." },
    { id: 11, type: 'J/P', question: "새로운 장소로 이사했을 때, 우리 강아지는?", options: [{ text: "적응 시간 걸리지만, 꾸준히 탐색하며 새 집 규칙 익혀요.", type: "J" }, { text: "새 환경 호기심 많고, 바로 자유롭게 돌아다니며 탐험해요.", type: "P" }], tooltip: "새로운 환경에 적응할 때, 체계적으로 파악하려 하는지, 아니면 자유롭게 탐색하며 적응하는지 확인합니다." },
    { id: 12, type: 'J/P', question: "놀이 시간이 끝나고, 우리 강아지는?", options: [{ text: "놀았던 장난감 스스로 정리하거나 공간 정돈하려 해요.", type: "J" }, { text: "놀이 끝나면 바로 다른 일 몰두, 장난감은 그냥 둬요.", type: "P" }], tooltip: "활동을 마무리하는 방식을 통해, 정리정돈과 마무리를 중시하는지, 아니면 유연하게 다음 활동으로 넘어가는지 봅니다." }
];

// MBTI result descriptions
const mbtiResults = {
  ISTJ: { nickname: "꼼꼼한 집사견 🐕‍🦺", summary: "조용하고 성실하며 규칙적인 것을 좋아하는 책임감 강한 강아지!", traits: "세상 모든 것이 예측 가능하고 안정적인 것을 선호하는 타입이에요. 주인에게 무한한 충성심을 보여주며, 정해진 루틴 안에서 가장 큰 행복을 느낀답니다. 차분하고 신뢰할 수 있는 강아지지만, 가끔은 너무 진지해서 '혹시 시계 보는 거 아니야?' 하는 착각이 들기도 해요.", compatibleOwner: "ESTP, ESFP (새로운 에너지를 불어넣어 줄 수 있는 활동적인 견주)", caution: "갑작스러운 변화나 예측 불가능한 상황에 스트레스를 받을 수 있어요. 규칙적인 생활 패턴을 유지해주고, 새로운 것을 도입할 때는 충분한 시간을 주고 서서히 익숙해지도록 도와주세요.", recommendedPlay: "퍼즐 장난감, 노즈 워크, 정해진 훈련 규칙을 따르는 놀이 (예: 숨바꼭질)", customCare: "규칙적인 생활 패턴 유지, 안정적이고 예측 가능한 환경 조성, 갑작스러운 변화 최소화." },
  ISFJ: { nickname: "다정한 수호견 💖", summary: "온화하고 배려심 많으며 가족을 지키는 것에 진심인 강아지!", traits: "가족을 내 몸처럼 여기는 따뜻한 마음의 소유자예요. 주인의 감정을 기가 막히게 알아채고, 묵묵히 옆을 지켜주는 든든한 강아지랍니다. 하지만 가끔은 너무 타인을 신경 쓰느라 자기 주장을 못 할 때도 있어요.", compatibleOwner: "ENTP, ESTP (강아지에게 긍정적인 자극과 리더십을 제공해 줄 수 있는 견주)", caution: "다른 가족이나 친구에게 너무 몰두하여 자신의 욕구를 놓칠 수 있어요. 충분한 애정과 관심을 표현해주고, 스트레스에 민감하니 안정적인 환경을 제공하는 것이 중요해요.", recommendedPlay: "가족과 함께하는 부드러운 놀이, 인형 장난감, 숨바꼭질, 공놀이", customCare: "충분한 애정 표현과 스킨십, 스트레스 없는 환경." },
  INFJ: { nickname: "신비로운 통찰견 🔮", summary: "직관적이고 깊이 있으며 주인의 감정을 꿰뚫어 보는 영혼의 동반자!", traits: "주인의 마음을 읽는 능력자! 말하지 않아도 주인의 기분을 알아채고 조용히 곁을 지켜주는 섬세한 강아지예요. 깊은 생각에 잠기거나 몽상에 빠지는 것을 좋아하며, 가끔은 너무 신비로워서 '네가 사람이야, 강아지야?' 하는 생각이 들 때도 있답니다.", compatibleOwner: "ENTP, ESTP (새로운 생각과 활력을 불어넣어 줄 수 있는 견주)", caution: "주인의 감정에 너무 동화되어 스트레스를 받거나, 혼자만의 시간을 너무 많이 보내서 우울해질 수 있어요. 정서적 지지와 함께 적당한 사회화 활동이 필요해요.", recommendedPlay: "의미 있는 놀이, 평화로운 활동 (예: 다른 강아지와의 그룹 놀이, 조용한 산책), 부드러운 인형 놀이", customCare: "조용하고 평화로운 환경, 충분한 사색 시간 보장, 주인의 정서적 지지." },
  INTJ: { nickname: "독립적인 전략견 🧠", summary: "논리적이고 계획적이며 자신만의 세계가 확고한 똑똑이 강아지!", traits: "계획대로 움직이는 것을 선호하고, 혼자서도 잘 노는 독립적인 타입이에요. 주어진 상황을 분석하고 자신만의 전략을 세우는 것을 좋아한답니다. '나만의 시간을 방해하지 마!' 라고 말하는 듯 시크한 매력이 넘쳐요.", compatibleOwner: "ENFJ, ESFJ (강아지의 계획성을 존중하며 필요한 교류를 제공해 줄 수 있는 견주)", caution: "사회성이 부족하거나 고집이 셀 수 있어요. 충분한 지적 자극을 제공하고, 새로운 상황에 대한 적응 훈련을 꾸준히 해주는 것이 좋아요.", recommendedPlay: "복잡한 퍼즐 장난감, 숨겨진 간식 찾기, 새로운 명령어 배우기", customCare: "개인 공간 존중, 지적 자극 제공, 규칙적인 루틴 유지, 사회화 훈련." },
  ISTP: { nickname: "실용적인 분석견 🛠️", summary: "조용하지만 호기심 많고 손재주(!)가 좋은 행동파 강아지!", traits: "직접 만져보고 경험하며 배우는 것을 좋아하는 강아지예요. 조용히 지켜보다가도 흥미로운 것이 있으면 바로 행동으로 옮기는 타입이랍니다. '이거 어떻게 분해하는 거지?' 하고 탐색하는 모습을 자주 볼 수 있어요.", compatibleOwner: "ENFJ, ESFJ (안정감을 제공하고 활동을 독려하는 견주)", caution: "충동적인 면이 있어 사고를 칠 수 있어요. 충분한 에너지 발산 기회와 함께 안전한 환경을 제공하는 것이 중요해요. 너무 많은 규칙은 스트레스를 줄 수 있어요.", recommendedPlay: "분해 가능한 장난감, 숨겨진 간식 찾기, 어질리티, 낚시 놀이", customCare: "자유로운 활동 공간, 다양한 자극 제공, 안전한 환경 조성, 직접 경험할 기회 제공." },
  ISFP: { nickname: "온화한 예술견 🎨", summary: "부드럽고 감성적이며 아름다운 것을 사랑하는 감성파 강아지!", traits: "예술가적 기질을 가진 감수성 풍부한 강아지예요. 아름다운 풍경을 감상하거나 부드러운 음악을 듣는 것을 좋아하고, 자신의 감정을 표현하는 데 능숙하답니다. '오늘 산책길 낙엽 색깔이 참 예쁘네!' 라고 생각할 것 같은 타입이에요.", compatibleOwner: "ENTJ, ESTJ (안정감을 주면서도 예술적 감성을 이해해 주는 견주)", caution: "환경 변화에 민감하고 상처를 잘 받을 수 있어요. 부드러운 목소리로 대화하고, 평화로운 분위기를 조성해 주는 것이 중요해요.", recommendedPlay: "색깔 있는 장난감, 부드러운 인형, 음악과 함께하는 놀이, 산책 중 자연물 탐색", customCare: "아름답고 평화로운 환경, 감성적인 교감, 충분한 애정 표현, 부드러운 대화." },
  INFP: { nickname: "이상주의 중재견 🌈", summary: "이상적이고 평화로운 세상을 꿈꾸며 조화를 추구하는 강아지!", traits: "세상 모든 것이 평화롭고 조화로운 것을 바라는 순수한 영혼이에요. 다른 강아지나 사람들과의 갈등을 싫어하고, 항상 긍정적인 관계를 유지하려 노력한답니다. '모두 사이좋게 지내자!'라고 외치는 듯한 평화주의자예요.", compatibleOwner: "ESTP, ESTJ (현실적인 조언과 안정적인 리더십을 제공하는 견주)", caution: "너무 이상적이어서 현실과 괴리감을 느낄 수 있고, 갈등 상황에서 회피하려는 경향이 있어요. 사회화 훈련과 함께 문제 해결 능력을 키워주는 것이 필요해요.", recommendedPlay: "의미 있는 놀이, 평화로운 활동 (예: 옆에서 함께 낮잠), 부드러운 인형 놀이", customCare: "평화로운 환경, 충분한 이해와 존중, 갈등 없는 분위기 조성, 진정성 있는 교감." },
  INTP: { nickname: "논리적인 사색견 🔬", summary: "논리적이고 호기심 많으며 세상의 모든 것을 분석하는 천재 강아지!", traits: "세상의 모든 원리를 탐구하고 싶어 하는 지적인 호기심이 넘쳐요. 문제에 부딪히면 자신만의 방식으로 해결책을 찾는 것을 좋아한답니다. '음... 이 간식은 왜 동그랗지?' 하고 고민할 것 같은 과학자 타입이에요.", compatibleOwner: "ESFJ, ENFJ (따뜻한 정서적 지지와 사회화를 돕는 견주)", caution: "사회성이 부족하거나 혼자만의 세계에 빠질 수 있어요. 지적 자극과 함께 충분한 사회화 훈련이 필요해요. 가끔은 너무 골똘히 생각해서 엉뚱한 행동을 하기도 해요.", recommendedPlay: "논리 퍼즐, 두뇌 자극 장난감, 숨겨진 간식 찾기, 새로운 기술 배우기", customCare: "지적 자극 제공, 탐구할 수 있는 환경 조성, 충분한 개인 시간 보장, 규칙적인 활동." },
  ESTP: { nickname: "활동적인 스턴트견 🤸", summary: "활동적이고 즉흥적이며 모험을 좋아하는 에너자이저 강아지!", traits: "앉아있기 싫어하는 타고난 모험가! 새로운 경험을 두려워하지 않고, 에너지를 발산하는 것에 큰 즐거움을 느껴요. '일단 해보고 생각하자!' 스타일의 진정한 행동파랍니다.", compatibleOwner: "INFJ, ISFJ (강아지의 넘치는 에너지를 이해하고 받아주는 견주)", caution: "너무 즉흥적이어서 사고를 칠 가능성이 높아요. 충분한 운동량과 함께 안전한 환경을 제공하는 것이 중요해요. 너무 많은 제약은 스트레스를 줄 수 있어요.", recommendedPlay: "원반 던지기, 공놀이, 어질리티, 수영, 달리기 등 격렬한 활동", customCare: "충분한 운동량 제공, 다양한 외부 활동, 새로운 경험 기회 제공, 안전한 활동 공간." },
  ESFP: { nickname: "자유로운 인싸견 🥳", summary: "활발하고 사교적이며 사람들과 어울리는 것을 좋아하는 인기쟁이 강아지!", traits: "주변의 시선을 한 몸에 받는 타고난 스타! 사람과 다른 강아지들과 어울리는 것을 세상에서 가장 좋아해요. 언제나 즐거움을 추구하며 분위기 메이커 역할을 톡톡히 한답니다. '모두 함께 놀자!'라고 말하는 듯한 붙임성 좋은 강아지예요.", compatibleOwner: "INTP, INTJ (강아지의 사교성을 이해하고 따뜻하게 받아주는 견주)", caution: "외로움을 많이 타거나 관심받지 못하면 힘들어할 수 있어요. 충분한 사회적 상호작용과 애정 표현이 필요해요. 지나치게 산만해지지 않도록 주의해야 해요.", recommendedPlay: "그룹 놀이, 숨바꼭질, 새로운 사람 만나기, 애교 부리기, 댄스(?)", customCare: "사회적 환경 조성, 충분한 상호작용, 많은 관심과 애정, 즐거운 분위기." },
  ENFP: { nickname: "열정적인 탐험견 🔥", summary: "열정적이고 창의적이며 항상 새로운 가능성을 꿈꾸는 강아지!", traits: "넘치는 호기심과 상상력으로 가득한 강아지예요. 새로운 아이디어가 샘솟고, 모든 것에 의미를 부여하며 세상을 탐험한답니다. '오늘은 또 어떤 신나는 일이 벌어질까?' 하는 기대감으로 가득 차 있어요.", compatibleOwner: "ISTP, ISTJ (강아지의 에너지를 존중하고 현실적인 가이드라인을 제공하는 견주)", caution: "산만하거나 너무 많은 것을 시도하려 할 수 있어요. 에너지 발산과 함께 집중력을 높이는 훈련이 필요해요. 가끔은 너무 앞서나가서 주인을 당황시키기도 해요.", recommendedPlay: "창의적인 놀이, 상상력 게임 (예: 보이지 않는 친구와 놀기), 탐험 활동, 숨바꿈질", customCare: "창의적인 환경, 다양한 경험 제공, 자유로운 사고 존중, 활발한 교감." },
  ENTP: { nickname: "뜨거운 토론견 🗣️", summary: "창의적이고 논쟁적이며 지적 도전을 즐기는 논리왕 강아지!", traits: "세상의 모든 규칙에 '왜?'라는 질문을 던지는 토론의 왕! 주인의 말에도 논리적으로 반박하는 듯한 모습을 보여줘요. 똑똑하고 재치 넘치지만, 가끔은 너무 고집이 세서 '네가 이기나 내가 이기나 해보자' 싶을 때도 있답니다.", compatibleOwner: "INFJ, ISFJ (강아지의 지적 호기심을 이해하고 따뜻하게 받아주는 견주)", caution: "지루함을 잘 느끼고 도전적인 것을 좋아해요. 충분한 지적 자극을 제공하지 않으면 문제 행동으로 이어질 수 있어요. 고집이 세거나 논쟁적인 태도를 보일 수 있으니 일관된 훈련이 중요해요.", recommendedPlay: "도전적인 게임, 논리 퍼즐, 숨겨진 물건 찾기, 새로운 기술 개발", customCare: "지적 자극, 창의적인 환경, 도전적인 과제 제공, 일관된 리더십." },
  ESTJ: { nickname: "엄격한 관리견 🏆", summary: "체계적이고 책임감 있으며 질서를 중요하게 생각하는 리더 강아지!", traits: "가정의 질서와 규율을 중요하게 생각하는 타고난 리더예요. 뭐든지 계획대로 움직여야 직성이 풀리고, 주변을 관리하려는 본능이 강하답니다. '자, 이제 밥 시간이야! 다들 줄 서!' 라고 외칠 것 같은 타입이에요.", compatibleOwner: "INFP, INTP (강아지의 계획성을 존중하고 새로운 시야를 열어주는 견주)", caution: "융통성이 부족하고 고집이 셀 수 있어요. 명확한 규칙과 일관된 훈련이 필요해요. 지나친 통제는 스트레스를 유발할 수 있으니 적당한 자유를 허용해 주세요.", recommendedPlay: "조직적인 게임, 규칙이 명확한 놀이, 리더십 활동 (예: 산책 시 앞장서기)", customCare: "질서 있는 환경, 명확한 역할 부여, 예측 가능한 일상, 일관된 리더십." },
  ESFJ: { nickname: "사교적인 외교견 🤝", summary: "친근하고 협력적이며 다른 동물들과 잘 어울리는 사회생활 만렙 강아지!", traits: "주변 모든 이들과 화목하게 지내고 싶어 하는 타고난 외교관이에요. 친구들과 어울리고, 가족 구성원들을 챙기는 것을 좋아한답니다. '모두 함께 놀자!'라고 말하는 듯한 붙임성 좋은 강아지예요.", compatibleOwner: "INTP, INTJ (강아지의 사교성을 지지하고 따뜻하게 감싸주는 견주)", caution: "타인의 시선을 의식하거나 외로움을 많이 탈 수 있어요. 충분한 사회적 교류와 애정 표현이 필수예요. 가끔은 너무 남의 일에 참견하려 들기도 해요.", recommendedPlay: "그룹 놀이, 협력 게임, 새로운 친구 만나기, 사람들과 어울리기", customCare: "사회적 환경, 충분한 상호작용, 많은 관심과 애정, 공동체 활동 참여." },
  ENFJ: { nickname: "열정적인 선도견 ✨", summary: "카리스마 있고 영감을 주며 다른 강아지들을 이끄는 리더 강아지!", traits: "주변에 긍정적인 영향력을 끼치는 타고난 리더예요. 다른 강아지들을 이끌고, 주인의 기분을 북돋아 주는 데 능하답니다. '나만 따라와! 더 멋진 세상을 보여줄게!'라고 외칠 것 같은 카리스마 넘치는 강아지예요.", compatibleOwner: "ISTP, ISTJ (강아지의 리더십을 이해하고 현실적인 지원을 아끼지 않는 견주)", caution: "다른 강아지들에게 너무 많은 기대를 하거나, 자신만 앞서나가서 피로감을 느낄 수 있어요. 충분한 휴식과 함께 다른 강아지들의 개성을 존중하는 훈련이 필요해요.", recommendedPlay: "리더십 게임, 영감적인 활동 (예: 새로운 묘기 개발), 그룹 놀이 주도", customCare: "리더십 환경, 영감적인 자극, 충분한 책임감 부여, 다른 강아지들과의 긍정적 교류." },
  ENTJ: { nickname: "대담한 통솔견 👑", summary: "자신감 있고 전략적이며 목표 달성을 좋아하는 카리스마 강아지!", traits: "목표를 향해 거침없이 돌진하는 타고난 전략가예요. 자신감이 넘치고, 어떤 상황에서도 해결책을 찾아내는 데 능하답니다. '이 구역의 대장은 나야!'라고 말하는 듯한 위풍당당한 강아지예요.", compatibleOwner: "ISFP, INFP (강아지의 강인함을 이해하고 부드러운 교감을 제공하는 견주)", caution: "고집이 세거나 다른 강아지들에게 권위적으로 보일 수 있어요. 인내심 있는 훈련과 함께 다른 강아지들과의 조화로운 관계를 배우는 것이 중요해요. 너무 많은 경쟁은 스트레스를 줄 수 있어요.", recommendedPlay: "전략 게임, 도전적인 활동 (예: 복잡한 어질리티 코스), 간식 쟁취 게임", customCare: "도전적인 환경, 성취 기회 제공, 명확한 목표 제시, 일관된 훈련." },
};

// Survey questions for additional feedback
const surveyQuestions = [
    { id: 'q1_preferredExperts', question: "펫 전문가가 집으로 직접 찾아와 제공하는 서비스 중, 어떤 전문가를 원하시나요?", type: 'checkbox', options: [ { label: "훈련사", icon: "🎓" }, { label: "미용사", icon: "✂️" }, { label: "전문펫시터", icon: "🐾" }, { label: "수의사", icon: "🩺" }, { label: "영양사", icon: "🥗" } ], hasOther: true },
    { id: 'q2_newServiceIdeas', question: "우리 강아지와 관련해서 새로 생기면 좋을 것 같은 서비스가 있다면 가감 없이 적어주세요! (어떤 아이디어든 환영해요!)", type: 'textarea', minLength: 0, maxLength: 500 },
    { id: 'q3_previousServiceFeedback', question: "이전에 이용했던 펫 서비스가 있다면, 업체명과 이용 만족도(상/중/하), 그리고 개선되었으면 하는 점을 자유롭게 적어주세요.", type: 'textarea', minLength: 0, maxLength: 500 },
    { id: 'q4_appropriateFee', question: "펫을 위한 맞춤형 프리미엄 서비스가 신설된다면 1시간 이용 시 적정 요금을 선택해주세요. (단일선택)", type: 'radio', options: ["1~2만원", "2~3만원", "3~4만원", "5~6만원"] }
];

// --- Main App Component ---

function App() {
  // State Management
  const [page, setPage] = useState('start'); // Current page: start, mbti, result, survey, complete, admin
  const [mbtiAnswers, setMbtiAnswers] = useState([]); // User's MBTI answers
  const [mbtiResult, setMbtiResult] = useState(null); // Calculated MBTI result
  const [shuffledQuestions, setShuffledQuestions] = useState([]); // Shuffled MBTI questions
  const [currentQuestionIndex, setCurrentQuestionIndex] = useState(0); // Current question index
  const [isAdmin, setIsAdmin] = useState(false); // Admin mode status
  const [showAdminModal, setShowAdminModal] = useState(false); // Admin login modal visibility
  const [showConfirmModal, setShowConfirmModal] = useState(false); // Confirmation/alert modal visibility
  const [modalContent, setModalContent] = useState(null); // Content for the confirmation/alert modal
  const inactivityTimer = useRef(null); // Timer for inactivity

  // Firebase specific states
  const [firebaseInitialized, setFirebaseInitialized] = useState(false);
  const [currentUserId, setCurrentUserId] = useState(null);

  // --- Firebase Initialization and Auth Listener ---
  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, async (user) => {
      if (user) {
        // User is signed in
        setCurrentUserId(user.uid);
      } else {
        // User is signed out, try to sign in anonymously or with custom token
        try {
          if (initialAuthToken) {
            await signInWithCustomToken(auth, initialAuthToken);
          } else {
            await signInAnonymously(auth);
          }
        } catch (error) {
          console.error("Firebase Auth Error:", error);
          // Optionally show an error message if anonymous sign-in fails
          setModalContent(
            <div>
                <h3 className="text-xl font-bold text-red-600 mb-4">인증 오류</h3>
                <p className="text-gray-700 mb-6">사용자 인증에 실패했습니다. 앱을 새로고침하거나 나중에 다시 시도해주세요.</p>
                <div className="flex justify-end">
                    <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                </div>
            </div>
          );
          setShowConfirmModal(true);
        }
      }
      setFirebaseInitialized(true); // Mark Firebase as initialized after auth check
    });

    return () => unsubscribe(); // Cleanup auth listener on component unmount
  }, []); // Run only once on component mount

  // --- Core Logic ---

  // Resets the application to its initial state
  const resetApp = useCallback(() => {
    setPage('start');
    setMbtiAnswers([]);
    setMbtiResult(null);
    setCurrentQuestionIndex(0);
    setShuffledQuestions([]);
  }, []);
  
  // Resets the inactivity timer
  const resetInactivityTimer = useCallback(() => {
    clearTimeout(inactivityTimer.current); // Clear any existing timer
    inactivityTimer.current = setTimeout(() => {
      // If on MBTI test page and inactive, show session expiry modal
      if (page === 'mbti') {
        setModalContent(
            <div>
                <h3 className="text-xl font-bold text-yellow-600 mb-4">세션 만료</h3>
                <p className="text-gray-700 mb-6">5분 동안 활동이 없어 시작 페이지로 돌아갑니다.</p>
                <div className="flex justify-end">
                    <button onClick={() => { setShowConfirmModal(false); resetApp(); }} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                </div>
            </div>
        );
        setShowConfirmModal(true);
      }
    }, 300000); // 5 minutes (300,000 milliseconds)
  }, [page, resetApp]);

  // Effect to manage inactivity timer based on page and question index
  useEffect(() => {
    if (page === 'mbti') {
      resetInactivityTimer(); // Start/reset timer when on MBTI page
    }
    return () => clearTimeout(inactivityTimer.current); // Clear timer on component unmount
  }, [page, currentQuestionIndex, resetInactivityTimer]);
  
  // Effect to shuffle questions when the MBTI test starts
  useEffect(() => {
    if (page === 'mbti' && shuffledQuestions.length === 0) {
      setShuffledQuestions([...mbtiQuestions].sort(() => Math.random() - 0.5));
    }
  }, [page, shuffledQuestions]);

  // Handles MBTI answer selection
  const handleMbtiAnswer = (answer) => {
    resetInactivityTimer(); // Reset timer on user interaction
    const newAnswers = [...mbtiAnswers];
    newAnswers[currentQuestionIndex] = answer;
    setMbtiAnswers(newAnswers);

    // Move to next question or calculate result if all questions answered
    if (currentQuestionIndex < shuffledQuestions.length - 1) {
      setCurrentQuestionIndex(prev => prev + 1);
    } else {
      calculateMbtiResult(newAnswers);
      setPage('result'); // Transition to result page
    }
  };

  // Navigates to the previous question
  const handleBack = () => {
    if (currentQuestionIndex > 0) {
      setCurrentQuestionIndex(prev => prev - 1);
    }
  };

  // Calculates the final MBTI result based on answers
  const calculateMbtiResult = (answers) => {
    const counts = { E: 0, I: 0, S: 0, N: 0, T: 0, F: 0, J: 0, P: 0 };
    answers.forEach(answer => {
      if (answer) counts[answer]++;
    });
    
    let result = "";
    result += counts.E >= counts.I ? 'E' : 'I';
    result += counts.S >= counts.N ? 'S' : 'N';
    result += counts.T >= counts.F ? 'T' : 'F';
    result += counts.J >= counts.P ? 'J' : 'P'; // Corrected: J/P should be based on count

    setMbtiResult({
      type: result,
      nickname: mbtiResults[result].nickname,
    });
  };

  // Handles survey submission
  const handleSubmitSurvey = async (surveyResponses) => {
    // Sanitize survey responses to prevent XSS attacks
    const sanitizedResponses = {};
    for (const key in surveyResponses) {
        if (typeof surveyResponses[key] === 'string') {
            sanitizedResponses[key] = DOMPurify.sanitize(surveyResponses[key]); 
        } else {
            sanitizedResponses[key] = surveyResponses[key];
        }
    }

    // Create a submission object with unique ID and timestamp
    const submission = {
      submissionId: crypto.randomUUID(),
      timestamp: new Date().toISOString(),
      mbtiResult: mbtiResult,
      surveyResponses: sanitizedResponses,
    };
    
    // Ensure Firebase is initialized and userId is available before adding submission
    if (firebaseInitialized && currentUserId) {
      try {
        await db.addSubmission(submission, currentUserId); // Add submission to Firestore
        setPage('complete'); // Transition to completion page
      } catch (error) {
        console.error("Error submitting survey to Firestore:", error);
        setModalContent(
          <div>
              <h3 className="text-xl font-bold text-red-600 mb-4">제출 실패</h3>
              <p className="text-gray-700 mb-6">설문 제출 중 오류가 발생했습니다. 잠시 후 다시 시도해주세요.</p>
              <div className="flex justify-end">
                  <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
              </div>
          </div>
        );
        setShowConfirmModal(true);
      }
    } else {
      console.error("Cannot submit survey: Firebase not initialized or user not authenticated.");
      // Optionally show an error modal to the user
      setModalContent(
        <div>
            <h3 className="text-xl font-bold text-red-600 mb-4">오류 발생</h3>
            <p className="text-gray-700 mb-6">사용자 인증에 문제가 발생하여 설문을 제출할 수 없습니다. 잠시 후 다시 시도해주세요.</p>
            <div className="flex justify-end">
                <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
            </div>
        </div>
      );
      setShowConfirmModal(true);
    }
  };

  // --- Render Methods ---

  // Renders the appropriate page based on the current 'page' state
  const renderPage = () => {
    // Show a loading indicator if Firebase is not yet initialized
    if (!firebaseInitialized) {
      return (
        <div className="flex flex-col items-center justify-center h-64">
          <div className="animate-spin rounded-full h-16 w-16 border-t-4 border-b-4 border-purple-500"></div>
          <p className="mt-4 text-gray-600">데이터를 불러오는 중입니다...</p>
        </div>
      );
    }

    switch (page) {
      case 'mbti': return <MbtiTestPage />;
      case 'result': return <ResultPage />;
      case 'survey': return <SurveyPage onSubmit={handleSubmitSurvey} />;
      case 'complete': return <CompletionPage />;
      case 'admin': return <AdminPage onExit={() => { setIsAdmin(false); setPage('start'); }} currentUserId={currentUserId} />;
      default: return <StartPage />;
    }
  };

  // --- Sub-Components for Pages ---

  // Start Page component
  const StartPage = () => (
    <div className="text-center flex flex-col items-center animate-fade-in">
       <svg className="w-24 h-24 mb-4 text-purple-500" viewBox="0 0 24 24" fill="currentColor"><path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8zm-2.5-7.5l5 3.5-5 3.5v-7zm5-1.5c.83 0 1.5-.67 1.5-1.5S15.33 4 14.5 4 13 4.67 13 5.5s.67 1.5 1.5 1.5zm-5 0c.83 0 1.5-.67 1.5-1.5S10.33 4 9.5 4 8 4.67 8 5.5s.67 1.5 1.5 1.5z"/></svg>
      <h2 className="text-2xl md:text-3xl font-bold text-gray-800 mb-4">강아지의 성격을 알아보고 맞춤형 케어를 찾아보세요!</h2>
      <p className="text-gray-600 mb-4">3분만에 우리 강아지의 숨겨진 성격을 발견할 수 있어요.</p>
      <p className="text-gray-500 mb-8 text-sm">간단한 12개 질문에 답하고 맞춤형 케어 팁과 서비스 설문에 참여해보세요.</p>
      <button onClick={() => setPage('mbti')} className="px-8 py-4 bg-purple-600 text-white font-bold rounded-full shadow-lg hover:bg-purple-700 transition-all transform hover:scale-105 focus:outline-none focus:ring-4 focus:ring-purple-300">
        테스트 시작하기
      </button>
    </div>
  );

  // MBTI Test Page component
  const MbtiTestPage = () => {
    if (shuffledQuestions.length === 0) return <div>로딩 중...</div>; // Show loading if questions not shuffled yet
    const question = shuffledQuestions[currentQuestionIndex];
    const progress = ((currentQuestionIndex + 1) / shuffledQuestions.length) * 100;

    return (
      <div className="animate-fade-in">
        <div className="mb-6">
          <div className="flex justify-between items-center mb-2">
            <span className="text-sm font-semibold text-purple-700">진행도</span>
            <span className="text-sm font-semibold text-gray-600">{currentQuestionIndex + 1} / {shuffledQuestions.length}</span>
          </div>
          <div className="w-full bg-gray-200 rounded-full h-2.5">
            <div className="bg-purple-500 h-2.5 rounded-full transition-all duration-500" style={{ width: `${progress}%` }}></div>
          </div>
        </div>
        <h2 className="text-xl md:text-2xl font-bold text-gray-800 mb-6 text-center h-24 flex items-center justify-center" style={{ wordBreak: 'keep-all' }}>
          {question.question}
          <HelpTooltip text={question.tooltip} />
        </h2>
        <div className="space-y-4">
          {question.options.map((option, idx) => (
            <button
              key={idx}
              onClick={() => handleMbtiAnswer(option.type)}
              className="w-full p-4 border-2 rounded-xl text-left text-lg transition-all duration-200 focus:outline-none focus:ring-2 focus:ring-purple-500 focus:ring-opacity-50 hover:bg-purple-50 hover:border-purple-400"
            >
              {option.text}
            </button>
          ))}
        </div>
        <div className="mt-8 flex justify-center">
          {currentQuestionIndex > 0 && (
            <button onClick={handleBack} className="px-6 py-2 bg-gray-200 text-gray-700 font-semibold rounded-full hover:bg-gray-300 transition-colors">
              이전
            </button>
          )}
        </div>
      </div>
    );
  };
  
  // Result Page component
  const ResultPage = () => {
    // isExpanded state is now initialized to true to show details by default
    const [isExpanded, setIsExpanded] = useState(true); 
    const resultData = mbtiResults[mbtiResult.type]; // Get MBTI result details

    // Function to share results on various SNS platforms
    const shareOnSns = (social) => {
      const url = encodeURIComponent(window.location.href);
      const text = encodeURIComponent(`우리 강아지 MBTI는 ${mbtiResult.type} ${resultData.nickname}래요! 멍BTI 테스트 해보세요! #강아지MBTI #멍BTI`);
      const title = encodeURIComponent(`우리 강아지 MBTI 결과: ${resultData.nickname}`);
      let shareUrl = '';

      switch(social) {
        case 'twitter':
          shareUrl = `https://twitter.com/intent/tweet?url=${url}&text=${text}`;
          break;
        case 'naver':
          shareUrl = `https://share.naver.com/web/shareView.nhn?url=${url}&title=${title}`;
          break;
        case 'threads':
          const threadsText = encodeURIComponent(`우리 강아지 MBTI는 ${mbtiResult.type} ${resultData.nickname}래요! 멍BTI 테스트 해보세요! \n\n${window.location.href}`);
          shareUrl = `https://www.threads.net/intent/post?text=${threadsText}`;
          break;
        default:
          return;
      }
      window.open(shareUrl, '_blank', 'width=600,height=600');
    };

    // Function to manually copy text to clipboard for sharing
    const handleManualShare = (platform) => {
        const textToCopy = `우리 강아지 MBTI는 ${mbtiResult.type} ${resultData.nickname}래요! 🐶\n\n${resultData.summary}\n\n#강아지MBTI #멍BTI #댕댕이MBTI #반려견성격테스트`;

        const textArea = document.createElement("textarea");
        textArea.value = textToCopy;
        textArea.style.position = "fixed";
        textArea.style.top = "0";
        textArea.style.left = "0";
        document.body.appendChild(textArea);
        textArea.focus();
        textArea.select();

        try {
            document.execCommand('copy'); // Use execCommand for broader compatibility in iframes
            setModalContent(
                <div>
                    <h3 className="text-xl font-bold text-green-600 mb-4">복사 완료!</h3>
                    <p className="text-gray-700 mb-6">결과 내용이 클립보드에 복사되었어요. 이제 {platform}에 붙여넣기하여 공유해보세요!</p>
                    <div className="flex justify-end">
                        <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                    </div>
                </div>
            );
        } catch (err) {
            setModalContent(
                <div>
                    <h3 className="text-xl font-bold text-red-600 mb-4">복사 실패</h3>
                    <p className="text-gray-700 mb-6">복사 기능이 지원되지 않는 브라우저입니다.</p>
                     <div className="flex justify-end">
                        <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                    </div>
                </div>
            );
        }

        document.body.removeChild(textArea);
        setShowConfirmModal(true);
    };

    return (
      <div className="text-center animate-fade-in">
        <div className="bg-white p-6 rounded-lg">
          <h2 className="text-3xl font-extrabold text-purple-700 mb-2">
            우리 강아지는 <span className="text-purple-900">{mbtiResult.type}</span>!
          </h2>
          <p className="text-xl sm:text-2xl font-semibold text-gray-800 mb-4" style={{ wordBreak: 'keep-all' }}>{resultData.nickname}</p>
          <div className="w-40 h-40 mx-auto bg-purple-100 rounded-full flex items-center justify-center mb-6">
            <span className="text-6xl">{mbtiResults[mbtiResult.type].nickname.split(' ').pop()}</span>
          </div>
          {/* Moved survey button to be directly under the trophy icon */}
          <button onClick={() => setPage('survey')} className="px-8 py-4 bg-blue-600 text-white text-xl font-bold rounded-full shadow-lg hover:bg-blue-700 transition-all transform hover:scale-105 focus:outline-none focus:ring-4 focus:ring-blue-300 mt-6 mb-4"> {/* Added mt-6 and mb-4 for spacing */}
            맞춤형 서비스 설문 참여하기
          </button>
          <p className="text-lg text-gray-700 mb-6 font-semibold bg-yellow-100 border-l-4 border-yellow-400 p-4 rounded-r-lg" style={{ wordBreak: 'keep-all' }}>
            <span className="font-bold text-purple-600">[{mbtiResult.type}]</span> 특성을 가진 우리 강아지, 어떤 맞춤형 케어가 필요할까요? <br/>우리 아이에게 필요한 맞춤형 케어를 원하시는 분만 클릭!
          </p>
          <div className="bg-purple-50 p-6 rounded-xl text-left mb-6 shadow-inner space-y-3">
             <p style={{ wordBreak: 'keep-all' }}><strong>한 줄 요약:</strong> {resultData.summary}</p>
             <div className={isExpanded ? 'space-y-3' : 'hidden'}>
                <p><strong>성격 특성:</strong> {resultData.traits}</p>
                <p><strong>찰떡궁합 견주 유형:</strong> {resultData.compatibleOwner}</p>
                <p><strong>주의할 점:</strong> {resultData.caution}</p>
                <p><strong>추천 놀이:</strong> {resultData.recommendedPlay}</p>
                <p><strong>맞춤 케어:</strong> {resultData.customCare}</p>
             </div>
             <button onClick={() => setIsExpanded(!isExpanded)} className="text-purple-600 font-semibold hover:underline">
               {isExpanded ? '간략히 보기' : '자세히 보기'}
             </button>
          </div>


        </div>
        <div className="my-6 flex justify-center items-center space-x-2 sm:space-x-3">
            {/* Social Share Buttons with hover scale effect */}
            <button onClick={() => handleManualShare('인스타그램')} className="p-2 bg-gradient-to-br from-yellow-400 via-red-500 to-purple-600 text-white rounded-full shadow-md hover:opacity-90 transition-opacity hover:scale-110 transition-transform" title="인스타그램에 공유">
                <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><rect x="2" y="2" width="20" height="20" rx="5" ry="5"></rect><path d="M16 11.37A4 4 0 1 1 12.63 8 4 4 0 0 1 16 11.37z"></path><line x1="17.5" y1="6.5" x2="17.51" y2="6.5"></line></svg>
            </button>
            <button onClick={() => handleManualShare('틱톡')} className="p-2 bg-black text-white rounded-full shadow-md hover:opacity-90 transition-opacity flex items-center justify-center hover:scale-110 transition-transform" title="틱톡에 공유">
                 <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor"><path d="M12.525.02c1.31-.02 2.61-.01 3.91-.02.08 1.53.63 3.09 1.75 4.17 1.12 1.11 2.7 1.62 4.24 1.79v4.03c-1.44-.05-2.89-.35-4.2-.97-.57-.26-1.1-.59-1.62-.93-.01 2.92.01 5.84-.02 8.75-.08 1.4-.54 2.79-1.35 3.94-1.31 1.92-3.58 3.17-5.91 3.21-2.43.05-4.84-.95-6.6-2.8-1.78-1.88-2.8-4.42-2.8-7.15 0-2.55 1.02-5.02 2.8-6.81 1.77-1.78 4.2-2.79 6.75-2.78.01 2.87-.01 5.74.02 8.61.01.07 0 .14-.01.21-.23-.12-.48-.23-.7-.35-1.03-.52-2.12-.9-3.26-1.1-1.02-.18-2.05-.15-3.07.02v4.03c.94-.01 1.89-.01 2.83 0 .5.01 1 .05 1.48.15 1.1.23 2.15.68 3.02 1.5.88.81 1.49 1.91 1.62 3.08.02.16.04.33.05.51v-9.42c-.23-.11-.47-.22-.7-.34-1.17-.59-2.38-.99-3.63-1.16-1.15-.16-2.3-.14-3.44.03V4.03c1.25-.01 2.5-.01 3.75 0 .01 0 .01 0 .02 0z"/></svg>
            </button>
            <button onClick={() => shareOnSns('naver')} className="p-2 bg-[#03C75A] text-white rounded-full shadow-md hover:opacity-90 transition-opacity flex items-center justify-center hover:scale-110 transition-transform" title="네이버 블로그에 공유">
                <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
                    <path d="M16.273 12.845 7.727 4.3v8.545h8.546V4.3L7.727 12.845v5.364h8.546v-5.364z"/>
                </svg>
            </button>
            <button onClick={() => shareOnSns('threads')} className="p-2 bg-black text-white rounded-full shadow-md hover:opacity-90 transition-opacity hover:scale-110 transition-transform" title="Threads에 공유">
                <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor"><path d="M13.17,8.25c0-1.29-1.05-2.34-2.34-2.34H5.34C4.05,5.91,3,6.96,3,8.25v2.23c0,0.41,0.34,0.75,0.75,0.75 s0.75-0.34,0.75-0.75V8.25c0-0.41,0.34-0.75,0.75-0.75h5.49c0.41,0,0.75,0.34,0.75,0.75v7.5c0,0.41-0.34,0.75-0.75,0.75h-2.2 c-0.41,0-0.75,0.34-0.75,0.75s0.34,0.75,0.75,0.75h2.2c1.29,0,2.34-1.05,2.34-2.34v-7.5H13.17z M20.25,10.48 c-0.41,0-0.75,0.34-0.75,0.75v4.5c0,0.41-0.34,0.75-0.75,0.75h-5.49c-0.41,0-0.75-0.34-0.75-0.75v-2.23c0-0.41-0.34-0.75-0.75-0.75 s-0.75,0.34-0.75,0.75v2.23c0,1.29,1.05,2.34,2.34,2.34h5.49c1.29,0,2.34-1.05,2.34-2.34v-4.5 C21,10.82,20.66,10.48,20.25,10.48z"/></svg>
            </button>
            {/* Updated Facebook icon to a filled 'f' logo */}
            <button onClick={() => handleManualShare('페이스북')} className="p-2 bg-blue-600 text-white rounded-full shadow-md hover:bg-blue-700 transition-colors hover:scale-110 transition-transform" title="페이스북에 공유">
                <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor">
                    <path d="M17 2H14c-3.86 0-7 3.14-7 7v3H4v4h3v7h4v-7h3l1-4h-4V9c0-1.1.9-2 2-2h3V2z"/>
                </svg>
            </button>
            <button onClick={() => shareOnSns('twitter')} className="p-2 bg-black text-white rounded-full shadow-md hover:opacity-90 transition-opacity hover:scale-110 transition-transform" title="X에 공유"><svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="currentColor"><path d="M18.244 2.25h3.308l-7.227 8.26 8.502 11.24H16.17l-5.214-6.817L4.99 21.75H1.68l7.73-8.835L1.254 2.25H8.08l4.713 6.231zm-1.161 17.52h1.833L7.084 4.126H5.117z"/></svg></button>
        </div>
      </div>
    );
  };

  // Survey Page component
  const SurveyPage = ({ onSubmit }) => {
    const [surveyResponses, setSurveyResponses] = useState({}); // State for survey answers
    const [errors, setErrors] = useState({}); // State for validation errors

    // Handles checkbox changes
    const handleCheckboxChange = (id, value) => {
      const current = surveyResponses[id] || [];
      const newValues = current.includes(value)
        ? current.filter(item => item !== value)
        : [...current, value];
      setSurveyResponses(prev => ({ ...prev, [id]: newValues }));
    };

    // Handles text input changes
    const handleTextChange = (id, value) => {
      setSurveyResponses(prev => ({ ...prev, [id]: value }));
    };
    
    // Validates survey responses
    const validate = () => {
        const newErrors = {};
        surveyQuestions.forEach(q => {
            if (q.type === 'textarea') {
                const value = surveyResponses[q.id] || '';
                if (q.minLength > 0 && value.trim().length > 0 && value.trim().length < q.minLength) {
                    newErrors[q.id] = `${q.minLength}자 이상 입력해주세요.`;
                }
                if (value.length > q.maxLength) {
                    newErrors[q.id] = `${q.maxLength}자 이하로 입력해주세요.`;
                }
                // Basic check for common script injection characters
                if (/[<>;]/.test(value)) {
                    newErrors[q.id] = '특수문자(<, >, ;)는 사용할 수 없습니다.';
                }
            }
        });
        setErrors(newErrors);
        return Object.keys(newErrors).length === 0; // Return true if no errors
    };

    // Submits the survey if valid
    const submit = () => {
        if (validate()) {
            onSubmit(surveyResponses);
        }
    };

    return (
      <div className="animate-fade-in">
        <h2 className="text-2xl font-bold text-gray-700 mb-6 text-center">펫 서비스 설문</h2>
        <div className="space-y-8">
          {surveyQuestions.map(q => (
            <div key={q.id} className="bg-gray-50 p-6 rounded-xl shadow-sm border border-gray-200">
              <p className="text-lg font-semibold text-gray-800 mb-4">{q.question}</p>
              {q.type === 'checkbox' && (
                <div className="space-y-3">
                  {q.options.map((opt, idx) => (
                    <label key={idx} className="flex items-center text-gray-700 cursor-pointer p-2 rounded-md hover:bg-gray-100 hover:scale-[1.01] transition-transform">
                      <input type="checkbox" checked={(surveyResponses[q.id] || []).includes(opt.label)} onChange={() => handleCheckboxChange(q.id, opt.label)} className="form-checkbox h-5 w-5 text-purple-600 rounded focus:ring-purple-500" />
                      <span className="ml-3 text-2xl">{opt.icon}</span>
                      <span className="ml-2">{opt.label}</span>
                    </label>
                  ))}
                  {q.hasOther && (
                    <div className="flex items-center mt-3">
                      <label className="flex items-center text-gray-700 cursor-pointer p-2 rounded-md hover:bg-gray-100 hover:scale-[1.01] transition-transform">
                        <input type="checkbox" checked={(surveyResponses[q.id] || []).includes('기타')} onChange={() => handleCheckboxChange(q.id, '기타')} className="form-checkbox h-5 w-5 text-purple-600 rounded focus:ring-purple-500" />
                        <span className="ml-3">기타</span>
                      </label>
                      {(surveyResponses[q.id] || []).includes('기타') && (
                        <input type="text" onChange={e => handleTextChange(`${q.id}_other`, e.target.value)} value={surveyResponses[`${q.id}_other`] || ''} className="ml-3 flex-grow p-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-purple-500" placeholder="기타 의견을 입력하세요" />
                      )}
                    </div>
                  )}
                </div>
              )}
              {q.type === 'radio' && (
                <div className="space-y-3">
                    {q.options.map((option, idx) => (
                        <label key={idx} className="flex items-center text-gray-700 cursor-pointer p-2 rounded-md hover:bg-gray-100 hover:scale-[1.01] transition-transform">
                            <input 
                                type="radio" 
                                name={q.id}
                                value={option}
                                checked={surveyResponses[q.id] === option}
                                onChange={(e) => handleTextChange(q.id, e.target.value)}
                                className="form-radio h-5 w-5 text-purple-600 focus:ring-purple-500" 
                            />
                            <span className="ml-3">{option}</span>
                        </label>
                    ))}
                </div>
              )}
              {q.type === 'textarea' && (
                <>
                  <textarea 
                    value={surveyResponses[q.id] || ''} 
                    onChange={e => handleTextChange(q.id, e.target.value)} 
                    rows="4" 
                    className={`w-full p-3 border rounded-md focus:outline-none focus:ring-2 transition-colors ${errors[q.id] ? 'border-red-500 ring-red-500' : 'border-gray-300 focus:ring-purple-500'}`} 
                    placeholder="답변을 입력해주세요." 
                    maxLength={q.maxLength}
                  ></textarea>
                  <div className="flex justify-between text-sm text-gray-500 mt-1">
                    <span>{errors[q.id] && <span className="text-red-500">{errors[q.id]}</span>}</span>
                    <span>{(surveyResponses[q.id] || '').length} / {q.maxLength}</span>
                  </div>
                </>
              )}
            </div>
          ))}
        </div>
        <div className="text-center mt-8">
          <button onClick={submit} className="px-8 py-4 bg-green-600 text-white text-xl font-bold rounded-full shadow-lg hover:bg-green-700 transition-all transform hover:scale-105 focus:outline-none focus:ring-4 focus:ring-green-300">
            설문 제출하기
          </button>
        </div>
      </div>
    );
  };

  // Completion Page component
  const CompletionPage = () => {
    // Function to copy the current URL to clipboard
    const copyUrlToClipboard = () => {
      // Changed to copy the origin (home screen URL) instead of the full current page URL
      const url = window.location.origin; 
      const textArea = document.createElement("textarea");
      textArea.value = url;
      textArea.style.position = "fixed"; // Prevent scrolling to bottom
      textArea.style.top = "0";
      textArea.style.left = "0";
      document.body.appendChild(textArea);
      textArea.focus();
      textArea.select();
      try {
        document.execCommand('copy');
        setModalContent(
            <div>
                <h3 className="text-xl font-bold text-green-600 mb-4">URL 복사 완료!</h3>
                <p className="text-gray-700 mb-6">앱의 홈 화면 주소가 클립보드에 복사되었습니다.</p>
                <div className="flex justify-end">
                    <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                </div>
            </div>
        );
      } catch (err) {
        setModalContent(
            <div>
                <h3 className="text-xl font-bold text-red-600 mb-4">URL 복사 실패</h3>
                <p className="text-gray-700 mb-6">URL 복사 기능이 지원되지 않는 브라우저입니다. 직접 복사해주세요.</p>
                <div className="flex justify-end">
                    <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                </div>
            </div>
        );
      }
      document.body.removeChild(textArea);
      setShowConfirmModal(true);
    };

    // Removed KakaoTalk Share function and SDK loading
    // const shareKakaoTalk = () => { /* ... */ };
    // useEffect(() => { /* ... */ }, []);

    return (
      <div className="text-center animate-fade-in">
        <h2 className="text-3xl font-extrabold text-green-700 mb-4">설문에 참여해주셔서 감사합니다! 🎉</h2>
        <div className="bg-green-50 p-6 rounded-xl shadow-inner mb-8">
          <p className="text-lg text-gray-700 mb-4">보호자님은 이제 <span className="font-bold text-green-800">강아지 성격 전문가</span>입니다! 🏅</p>
          <p className="text-gray-600">
            <span className="font-bold">{mbtiResult.nickname}</span> 강아지의 특성을 이해하고, <br/>더 나은 펫 라이프를 위한 소중한 의견을 주셨습니다.
          </p>
        </div>
        <div className="flex flex-col space-y-4 mb-8">
            <button 
                onClick={copyUrlToClipboard} 
                className="px-8 py-4 bg-purple-600 text-white text-xl font-bold rounded-full shadow-lg hover:bg-purple-700 transition-all transform hover:scale-105 focus:outline-none focus:ring-4 focus:ring-purple-300"
            >
                URL 복사하기
            </button>
            {/* KakaoTalk Share Button Removed */}
        </div>
        <button onClick={resetApp} className="px-8 py-4 bg-purple-600 text-white text-xl font-bold rounded-full shadow-lg hover:bg-purple-700 transition-all transform hover:scale-105 focus:outline-none focus:ring-4 focus:ring-purple-300">
          다시 테스트하기
        </button>
      </div>
    );
  };
  
  // Admin Page component for viewing statistics and managing data
  const AdminPage = ({ onExit, currentUserId }) => {
    const [submissions, setSubmissions] = useState([]); // All raw submissions
    const [filteredSubmissions, setFilteredSubmissions] = useState([]); // Filtered submissions for display
    const [filter, setFilter] = useState({ keyword: '', mbti: 'all', sort: 'newest' }); // Filters and sort options
    
    // Fetch submissions from Firestore in real-time using onSnapshot
    useEffect(() => {
        // Only fetch if Firebase is initialized and a user ID is available
        if (!firebaseInitialized || !currentUserId) return; 

        const submissionsCollectionRef = collection(dbFirestore, `artifacts/${appId}/users/${currentUserId}/submissions`);
        // Set up real-time listener for submissions
        const unsubscribe = onSnapshot(submissionsCollectionRef, (snapshot) => {
            const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
            setSubmissions(data);
        }, (error) => {
            console.error("Error fetching submissions:", error);
            // Handle error, maybe show a message to the admin
            setModalContent(
                <div>
                    <h3 className="text-xl font-bold text-red-600 mb-4">데이터 로딩 오류</h3>
                    <p className="text-gray-700 mb-6">설문 데이터를 불러오는 중 오류가 발생했습니다.</p>
                    <div className="flex justify-end">
                        <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                    </div>
                </div>
            );
            setShowConfirmModal(true);
        });

        return () => unsubscribe(); // Cleanup listener on component unmount
    }, [firebaseInitialized, currentUserId]); // Re-run when Firebase initialized or user changes

    // Apply filters and sorting whenever submissions or filter state changes
    useEffect(() => {
        let data = [...submissions];
        // Apply keyword filter
        if (filter.keyword) {
            const keyword = filter.keyword.toLowerCase();
            data = data.filter(s => JSON.stringify(s).toLowerCase().includes(keyword));
        }
        // Apply MBTI type filter
        if (filter.mbti !== 'all') {
            data = data.filter(s => s.mbtiResult.type === filter.mbti);
        }
        // Apply sorting
        // IMPORTANT: Sorting is done in memory as orderBy() in Firestore queries can require indexes
        if (filter.sort === 'newest') data.sort((a, b) => new Date(b.timestamp) - new Date(a.timestamp));
        if (filter.sort === 'oldest') data.sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));
        if (filter.sort === 'mbti') data.sort((a, b) => a.mbtiResult.type.localeCompare(b.mbtiResult.type));
        
        setFilteredSubmissions(data);
    }, [submissions, filter]);

    // Memoized statistics calculation for performance
    const stats = useMemo(() => {
        // Calculate MBTI type distribution
        const mbtiStats = submissions.reduce((acc, s) => {
            const type = s.mbtiResult.type;
            acc[type] = (acc[type] || 0) + 1;
            return acc;
        }, {});
        
        // Calculate preferred experts distribution
        const serviceStats = (key) => submissions.reduce((acc, s) => {
            const responses = s.surveyResponses[key] || [];
            responses.forEach(res => {
                acc[res] = (acc[res] || 0) + 1;
            });
            return acc;
        }, {});

        // Calculate appropriate fee distribution
        const feeStats = submissions.reduce((acc, s) => {
            const fee = s.surveyResponses.q4_appropriateFee;
            if (fee) {
                acc[fee] = (acc[fee] || 0) + 1;
            }
            return acc;
        }, {});

        return {
            mbti: Object.entries(mbtiStats).map(([name, value]) => ({ name, value })),
            experts: Object.entries(serviceStats('q1_preferredExperts')).map(([name, value]) => ({ name, value })),
            fees: Object.entries(feeStats).map(([name, value]) => ({ name, value })),
        };
    }, [submissions]);

    // Colors for charts
    const COLORS = ['#8884d8', '#82ca9d', '#ffc658', '#ff8042', '#0088FE', '#00C49F', '#FFBB28', '#FF8080'];
    
    // Exports filtered submissions to CSV
    const exportToCsv = () => {
        let csvContent = "data:text/csv;charset=utf-8,";
        csvContent += "제출ID,제출시간,MBTI유형,MBTI별칭,선호전문가,새서비스아이디어,기존서비스피드백,적정요금\n";
        
        filteredSubmissions.forEach(s => {
            const row = [
                s.submissionId,
                s.timestamp,
                s.mbtiResult.type,
                s.mbtiResult.nickname,
                (s.surveyResponses.q1_preferredExperts || []).join(';'),
                `"${s.surveyResponses.q2_newServiceIdeas || ''}"`, // Enclose with quotes for CSV
                `"${s.surveyResponses.q3_previousServiceFeedback || ''}"`, // Enclose with quotes for CSV
                s.surveyResponses.q4_appropriateFee || '',
            ].join(',');
            csvContent += row + "\n";
        });

        const encodedUri = encodeURI(csvContent);
        const link = document.createElement("a");
        link.setAttribute("href", encodedUri);
        link.setAttribute("download", "survey_submissions.csv");
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    };
    
    // Exports all submissions to JSON
    const exportToJson = () => {
        const jsonContent = "data:text/json;charset=utf-8," + encodeURIComponent(JSON.stringify(submissions, null, 2));
        const link = document.createElement("a");
        link.setAttribute("href", jsonContent);
        link.setAttribute("download", "survey_backup.json");
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    };
    
    // Handles clearing all data from Firestore
    const handleResetData = () => {
        setModalContent(
            <div>
                <h3 className="text-xl font-bold text-red-600 mb-4">모든 데이터 초기화</h3>
                <p className="text-gray-700 mb-6">정말로 모든 설문 데이터를 영구적으로 삭제하시겠습니까? 이 작업은 되돌릴 수 없습니다.</p>
                <div className="flex justify-end space-x-4">
                    <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">취소</button>
                    <button onClick={async () => {
                        if (currentUserId) {
                            try {
                                await db.clearAllSubmissions(currentUserId);
                                setSubmissions([]); // Clear local state after successful deletion
                                setShowConfirmModal(false);
                            } catch (error) {
                                console.error("Error clearing data:", error);
                                setModalContent(
                                    <div>
                                        <h3 className="text-xl font-bold text-red-600 mb-4">삭제 실패</h3>
                                        <p className="text-gray-700 mb-6">데이터 삭제 중 오류가 발생했습니다. 다시 시도해주세요.</p>
                                        <div className="flex justify-end">
                                            <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                                        </div>
                                    </div>
                                );
                                setShowConfirmModal(true);
                            }
                        } else {
                            console.error("Cannot clear data: User not authenticated.");
                            setModalContent(
                                <div>
                                    <h3 className="text-xl font-bold text-red-600 mb-4">오류 발생</h3>
                                    <p className="text-gray-700 mb-6">데이터를 삭제할 수 없습니다. 사용자 인증 상태를 확인해주세요.</p>
                                    <div className="flex justify-end">
                                        <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                                    </div>
                                </div>
                            );
                            setShowConfirmModal(true);
                        }
                    }} className="px-4 py-2 bg-red-600 text-white rounded-lg">삭제</button>
                </div>
            </div>
        );
        setShowConfirmModal(true);
    };

    return (
        <div className="animate-fade-in">
            <div className="flex justify-between items-center mb-6">
                <h2 className="text-3xl font-extrabold text-gray-800">관리자 통계</h2>
                <button onClick={onExit} className="px-4 py-2 bg-gray-200 rounded-lg">나가기</button>
            </div>
            <p className="text-gray-600 mb-2">현재 사용자 ID: <span className="font-bold text-purple-600 break-all">{currentUserId || '인증 중...'}</span></p>
            <p className="text-gray-600 mb-8">총 제출 건수: <span className="font-bold text-purple-600">{submissions.length}</span> 건</p>
            
            {/* Stats Section */}
            <div className="grid grid-cols-1 md:grid-cols-2 gap-8 mb-8">
                <div className="bg-blue-50 p-6 rounded-xl shadow-md">
                    <h3 className="text-xl font-bold text-blue-700 mb-4">MBTI 유형별 통계</h3>
                    <ResponsiveContainer width="100%" height={300}>
                        <PieChart>
                            <Pie data={stats.mbti} dataKey="value" nameKey="name" cx="50%" cy="50%" outerRadius={80} fill={COLORS[0]} label>
                                {stats.mbti.map((entry, index) => <Cell key={`cell-${index}`} fill={COLORS[index % COLORS.length]} />)}
                            </Pie>
                            <Tooltip />
                            <Legend />
                        </PieChart>
                    </ResponsiveContainer>
                </div>
                <div className="bg-green-50 p-6 rounded-xl shadow-md">
                    <h3 className="text-xl font-bold text-green-700 mb-4">선호 전문가 통계</h3>
                     <ResponsiveContainer width="100%" height={300}>
                        <BarChart data={stats.experts} layout="vertical" margin={{ top: 5, right: 20, left: 50, bottom: 5 }}>
                            <XAxis type="number" />
                            <YAxis type="category" dataKey="name" width={80} />
                            <Tooltip />
                            <Bar dataKey="value" fill={COLORS[1]} />
                        </BarChart>
                    </ResponsiveContainer>
                </div>
            </div>
            <div className="bg-yellow-50 p-6 rounded-xl shadow-md mb-8">
                <h3 className="text-xl font-bold text-yellow-700 mb-4">적정 요금 통계</h3>
                <ResponsiveContainer width="100%" height={300}>
                    <BarChart data={stats.fees} margin={{ top: 5, right: 30, left: 20, bottom: 5 }}>
                        <XAxis dataKey="name" />
                        <YAxis />
                        <Tooltip />
                        <Legend />
                        <Bar dataKey="value" fill={COLORS[2]} />
                    </BarChart>
                </ResponsiveContainer>
            </div>

            {/* Submissions List */}
            <div className="bg-gray-50 p-6 rounded-xl shadow-md">
                <h3 className="text-2xl font-bold text-gray-800 mb-4">개별 피드백 목록</h3>
                {/* Filters */}
                <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-4">
                    <input type="text" placeholder="키워드 검색..." value={filter.keyword} onChange={e => setFilter(f => ({...f, keyword: e.target.value}))} className="p-2 border rounded-md" />
                    <select value={filter.mbti} onChange={e => setFilter(f => ({...f, mbti: e.target.value}))} className="p-2 border rounded-md">
                        <option value="all">모든 MBTI</option>
                        {Object.keys(mbtiResults).map(type => <option key={type} value={type}>{type}</option>)}
                    </select>
                    <select value={filter.sort} onChange={e => setFilter(f => ({...f, sort: e.target.value}))} className="p-2 border rounded-md">
                        <option value="newest">최신순</option>
                        <option value="oldest">오래된순</option>
                        <option value="mbti">MBTI순</option>
                    </select>
                </div>
                {/* Actions */}
                <div className="flex space-x-4 mb-4">
                    <button onClick={exportToCsv} className="px-4 py-2 bg-green-600 text-white rounded-md">CSV 내보내기</button>
                    <button onClick={exportToJson} className="px-4 py-2 bg-blue-600 text-white rounded-md">JSON 백업</button>
                    <button onClick={handleResetData} className="px-4 py-2 bg-red-600 text-white rounded-md">데이터 초기화</button>
                </div>
                {/* List */}
                <div className="space-y-4 max-h-96 overflow-y-auto pr-2">
                    {filteredSubmissions.map(s => (
                        <div key={s.id} className="bg-white p-4 rounded-lg shadow">
                            <p className="text-sm text-gray-500">{new Date(s.timestamp).toLocaleString()}</p>
                            <p className="font-bold">{s.mbtiResult.type} ({s.mbtiResult.nickname})</p>
                            <details className="text-sm text-gray-700">
                                <summary className="cursor-pointer">상세보기</summary>
                                <ul className="list-disc list-inside mt-2 space-y-1">
                                    <li><strong>선호 전문가:</strong> {(s.surveyResponses.q1_preferredExperts || []).join(', ')}</li>
                                    {/* Corrected closing tag from </b> to </strong> */}
                                    <li><strong>새 아이디어:</strong> {s.surveyResponses.q2_newServiceIdeas || 'N/A'}</li>
                                    <li><strong>이전 경험:</strong> {s.surveyResponses.q3_previousServiceFeedback || 'N/A'}</li>
                                    <li><strong>적정 요금:</strong> {s.surveyResponses.q4_appropriateFee || 'N/A'}</li>
                                </ul>
                            </details>
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
  };
  
  // Handles admin login process
  const handleAdminLogin = () => {
    const checkPassword = () => {
        const passwordInput = document.getElementById('admin-password');
        if (!passwordInput) return;
        const password = passwordInput.value;

        if (password === 'admin1234') { // Hardcoded password for demonstration purposes
            setIsAdmin(true);
            setPage('admin');
            setShowAdminModal(false);
        } else {
            setShowAdminModal(false);
            // Use timeout to prevent modal conflict and ensure proper rendering
            setTimeout(() => {
                setModalContent(
                    <div>
                        <h3 className="text-xl font-bold text-red-600 mb-4">로그인 실패</h3>
                        <p className="text-gray-700 mb-6">비밀번호가 올바르지 않습니다.</p>
                        <div className="flex justify-end">
                            <button onClick={() => setShowConfirmModal(false)} className="px-4 py-2 bg-gray-200 rounded-lg">확인</button>
                        </div>
                    </div>
                );
                setShowConfirmModal(true);
            }, 200);
        }
    };

    setModalContent(
        <div>
            <h3 className="text-xl font-bold mb-4">관리자 로그인</h3>
            <input 
                type="password" 
                id="admin-password" 
                className="w-full p-2 border rounded-md mb-4" 
                placeholder="비밀번호" 
                onKeyDown={(e) => e.key === 'Enter' && checkPassword()}
            />
            <button 
                onClick={checkPassword} 
                className="w-full px-4 py-2 bg-purple-600 text-white rounded-lg"
            >
                로그인
            </button>
        </div>
    );
    setShowAdminModal(true);
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-purple-50 via-pink-50 to-blue-50 flex flex-col items-center justify-center p-4 font-sans">
      <div className="w-full max-w-3xl bg-white/80 backdrop-blur-sm p-6 md:p-10 rounded-3xl shadow-lg border border-gray-200">
        <header className="text-center mb-8">
          <h1 className="text-3xl md:text-4xl font-extrabold text-gray-800 flex items-center justify-center">
            {/* Admin login trigger icon with hover scale effect */}
            <span onClick={handleAdminLogin} className="cursor-pointer text-4xl mr-3 hover:scale-110 transition-transform" title="관리자 모드">🐕</span>
            강아지 MBTI 테스트
          </h1>
        </header>
        <main>
          {renderPage()}
        </main>
      </div>
      {/* Modals for admin login and confirmations */}
      {showAdminModal && <CustomModal onClose={() => setShowAdminModal(false)}>{modalContent}</CustomModal>}
      {showConfirmModal && <CustomModal onClose={() => setShowConfirmModal(false)}>{modalContent}</CustomModal>}
    </div>
  );
}

export default App;
