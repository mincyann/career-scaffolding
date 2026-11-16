기존 설계를 확인했고, 요구사항(간소화 + AI 피드백 확실 적용 + 5단계 목표 수준 연동)에 맞춰 단일 HTML 파일로 전면 재작성했습니다. 기존 코드는 7단계 흐름과 4개 파일(`app.py`, 모듈 2개, 1,825줄짜리 예시 JSON)로 구성되어 있었고, 선택한 목표 레벨은 시각화 눈금 계산에만 쓰이고 실제 안내 내용에는 반영되지 않았습니다. 새 버전은 이 두 문제를 동시에 해결합니다.

## 단순화 설계

* **7단계 → 3단계:** 목표 설정 → 계획 작성(핵심 질문 4개) → AI 피드백·저장. 기존의 독립 미션, 루브릭, 링커리어 패널, 경험 점검 등 부가 기능은 제거했습니다 (Unknown, n.d.).
* **Streamlit(stlite) + 4개 파일 → 단일 `index.html`:** Python 런타임 없이 GitHub Pages에 파일 하나만 올리면 동작합니다.
* **레벨이 실제로 작동:** 5단계(일단 찍먹~완전 정복)가 고정 안내 문구와 AI 시스템 프롬프트 양쪽에 직접 반영됩니다. 저레벨일수록 절차형 안내, 고레벨일수록 질문형 안내만 제공하는 스캐폴딩 강도 조절(fading) 구조입니다.
* **AI 피드백 확실 적용:** 학생이 자신의 API 키를 입력하고 버튼 한 번으로 각 질문별 피드백 + 총평을 받습니다. 키가 없으면 고정 안내만으로도 계획이 완성되는 이중 구조입니다.

---

## 전체 코드 (`index.html`)

```html
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>진로 계획 스캐폴딩</title>
<style>
:root{--main:#4f6df5;--soft:#eef1fe;--bg:#f6f7fb;--line:#e4e7ee;--muted:#8a8f9c}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:#1c2333;font-family:-apple-system,'Apple SD Gothic Neo','Malgun Gothic',sans-serif;line-height:1.6}
.wrap{max-width:760px;margin:0 auto;padding:32px 16px 96px}
h1{font-size:1.45rem;margin:0 0 4px}
.sub{color:var(--muted);font-size:.9rem;margin:0 0 20px}
.card{background:#fff;border:1px solid var(--line);border-radius:14px;padding:20px;margin:14px 0}
.card h2{font-size:1.05rem;margin:0 0 12px}
label{font-weight:600;font-size:.92rem;display:block;margin:12px 0 6px}
input[type=text],input[type=password],textarea{width:100%;border:1px solid var(--line);border-radius:8px;padding:10px 12px;font:inherit;background:#fff}
textarea{min-height:90px;resize:vertical}
.hint{background:var(--soft);border-left:3px solid var(--main);border-radius:0 8px 8px 0;padding:8px 12px;font-size:.85rem;color:#3d4666;margin:6px 0 4px}
.levels{display:flex;gap:6px;flex-wrap:wrap}
.levels label{flex:1;min-width:118px;border:1px solid var(--line);border-radius:10px;padding:10px;text-align:center;cursor:pointer;font-weight:600;font-size:.83rem;margin:0}
.levels input{display:none}
.levels label:has(input:checked){border-color:var(--main);background:var(--soft)}
#levelDesc{font-size:.85rem;color:var(--muted);margin:10px 0 0}
.btn{background:var(--main);color:#fff;border:none;border-radius:8px;padding:11px 18px;font:inherit;font-weight:600;cursor:pointer}
.btn:disabled{background:#c3c8d4;cursor:not-allowed}
.btn.ghost{background:#fff;color:var(--main);border:1px solid var(--main)}
.fb{border:1px dashed var(--main);background:#fbfcff;border-radius:10px;padding:12px;font-size:.88rem;margin:8px 0;white-space:pre-wrap;display:none}
.fb.on{display:block}
.fb b{color:var(--main)}
.status{font-size:.85rem;min-height:1.2em}
.ok{color:#1a7f4b}.err{color:#c0392b}
.note{font-size:.78rem;color:var(--muted)}
.row{display:flex;gap:10px;flex-wrap:wrap}.row>div{flex:1;min-width:180px}
details{margin:8px 0;padding:8px;background:#fafbff;border-radius:8px}
hr{border:none;border-top:1px solid var(--line);margin:16px 0}
</style>
</head>
<body>
<div class="wrap">
<h1>나의 진로 학습 계획</h1>
<p class="sub">3단계 · 약 10분 · 목표 레벨에 맞춘 안내와 AI 피드백</p>

<div class="card">
<h2>1단계 · 목표 설정</h2>
<label for="skill">키우고 싶은 역량</label>
<input id="skill" type="text" list="skillList" placeholder="예: 데이터 분석, 문제 정의, 발표, 팀 리더십">
<datalist id="skillList">
<option>데이터 분석·해석</option>
<option>문제 정의·기획</option>
<option>프로그래밍·구현</option>
<option>발표·커뮤니케이션</option>
<option>팀 프로젝트 리더십</option>
</datalist>
<label>목표 레벨 (4주 후 도전 수준)</label>
<div class="levels" id="levels"></div>
<p id="levelDesc"></p>
</div>

<div class="card">
<h2>2단계 · 계획 작성</h2>
<p class="note">각 질문 아래의 안내는 선택한 레벨에 따라 바뀝니다. 답변은 자동 저장됩니다.</p>
<div id="questions"></div>
</div>

<div class="card">
<h2>3단계 · AI 피드백 받기</h2>
<details>
<summary>AI 설정 (내 API 키 사용)</summary>
<div class="row">
<div><label for="apiUrl">API 주소 (OpenAI 호환)</label><input id="apiUrl" type="text" value="https://api.openai.com/v1/chat/completions"></div>
<div><label for="apiModel">모델</label><input id="apiModel" type="text" value="gpt-4o-mini"></div>
</div>
<label for="apiKey">API 키</label>
<input id="apiKey" type="password" placeholder="sk-...">
<label style="font-weight:400;font-size:.85rem"><input type="checkbox" id="remember"> 이 탭 세션 동안만 키 유지 (탭을 닫으면 삭제)</label>
</details>
<p style="margin:10px 0"><button class="btn" id="aiBtn">AI 피드백 받기</button></p>
<p class="status" id="aiStatus"></p>
<div class="fb" id="aiSummary"></div>
<hr>
<button class="btn ghost" id="mdBtn">계획서 내려받기 (.md)</button>
<p class="note">AI 피드백은 참고용이며 최종 판단은 학습자에게 있습니다. 키는 이 브라우저에서 직접 사용되며 별도 서버로 전송되지 않습니다. 키가 없어도 2단계의 고정 안내로 계획을 완성할 수 있습니다.</p>
</div>
</div>

<script>
const $=id=>document.getElementById(id);
const LEVELS=[
{n:1,name:'일단 찍먹',desc:'가볍게 시작하며 이 역량이 무엇인지 경험해요',
 ai:'학습자는 완전 초심자입니다. 각 답변에 대해 오늘 당장 실행할 수 있는 아주 작고 구체적인 첫 걸음을 2~3개 절차로 제안하고, 전문 용어는 쉬운 말로 풀어 주세요.'},
{n:2,name:'기본기 쌓기',desc:'기본 절차를 따라 하며 기초를 다져요',
 ai:'기본기를 익히는 단계입니다. 표준 절차와 간단한 예시를 제안하되, 자기 상황에 맞게 수정하도록 유도하세요.'},
{n:3,name:'제법 잘함',desc:'익숙한 과제는 스스로 수행해요',
 ai:'어느 정도 스스로 수행하는 단계입니다. 답변의 강점을 한 줄로 짚고, 구체화를 위한 질문 1~2개와 개선 방향만 제시하세요. 절차를 일방적으로 알려 주지 마세요.'},
{n:4,name:'고수 모드',desc:'새로운 상황에도 배운 방법을 적용·조정해요',
 ai:'숙련 단계입니다. 정답이나 절차를 제시하지 말고, 답변의 가정·우선순위·논리적 비약을 점검하는 질문만 하세요.'},
{n:5,name:'완전 정복',desc:'복잡한 과제도 독립적으로 수행하고 설명해요',
 ai:'독립 수행 단계입니다. 실행 방법을 일절 제시하지 말고, 계획의 취약점(일정 현실성·검증 기준·리스크)을 짚는 메타 질문만 하세요.'}
];
const QUESTIONS=[
{id:'q1',label:'① 현재 상태',ph:'이 역량을 어디까지 해봤고, 지금 무엇이 가장 막히나요?',
 hintD:'사실을 적어보세요. 예: "수업에서 배웠고, 실전에서 쓸 때마다 어디서부터 시작할지 막혀요"',
 hintQ:'가장 최근 경험 하나와 그때 막힌 지점은 무엇이었나요?'},
{id:'q2',label:'② 4주 목표와 완료 기준',ph:'4주 후 무엇을 할 수 있게 되고, 무엇으로 달성을 확인하나요?',
 hintD:'"행동 + 결과물 + 확인 기준"을 한 문장으로. 예: "가상 데이터로 분석 보고서 1장을 쓰고 동료 피드백을 받는다"',
 hintQ:'그 목표가 4주 안에 현실적인 이유는? 달성 여부를 판정할 가장 엄격한 기준은?'},
{id:'q3',label:'③ 이번 주 첫 활동',ph:'이번 주에 시도할 가장 작은 활동 하나는?',
 hintD:'30~60분 안에 끝낼 수 있게. 예: "공개 데이터셋 하나를 열어 컬럼 의미 정리하기"',
 hintQ:'그 활동이 목표 달성에 가장 크게 기여하는 이유는 무엇인가요?'},
{id:'q4',label:'④ 언제, 몇 분?',ph:'요일·시간·소요 시간을 정해보세요.',
 hintD:'캘린더에 바로 적을 수 있게. 예: "수요일 저녁 8시, 40분"',
 hintQ:'그 시간을 지키지 못할 가장 가능성 높은 상황과 대안은?'}
];
let state={level:3,skill:'',answers:{}};
function save(){localStorage.setItem('careerPlan',JSON.stringify(state))}
function load(){try{const s=JSON.parse(localStorage.getItem('careerPlan'));if(s){state=Object.assign(state,s)}}catch(e){}}
function esc(s){return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;')}
function hintFor(q){
  if(state.level<=2) return q.hintD;
  if(state.level===3) return q.hintD+' / '+q.hintQ;
  return q.hintQ;
}
function renderLevels(){
  $('levels').innerHTML=LEVELS.map(l=>
    '<label><input type="radio" name="lv" value="'+l.n+'"'+(state.level===l.n?' checked':'')+'>'+l.n+'. '+l.name+'</label>').join('');
  document.querySelectorAll('input[name=lv]').forEach(r=>r.onchange=()=>{state.level=+r.value;renderHints();save()});
  renderHints();
}
function renderQuestions(){
  $('questions').innerHTML=QUESTIONS.map(q=>
    '<label for="'+q.id+'">'+q.label+'</label>'+
    '<textarea id="'+q.id+'" placeholder="'+q.ph+'">'+esc(state.answers[q.id]||'')+'</textarea>'+
    '<div class="hint" id="hint-'+q.id+'"></div>'+
    '<div class="fb" id="fb-'+q.id+'"></div>').join('');
  QUESTIONS.forEach(q=>{$(q.id).oninput=e=>{state.answers[q.id]=e.target.value;save()}});
  renderHints();
}
function renderHints(){
  const l=LEVELS[state.level-1];
  $('levelDesc').textContent='LV'+l.n+' '+l.name+' · '+l.desc;
  QUESTIONS.forEach(q=>{$('hint-'+q.id).textContent='안내: '+hintFor(q)});
}
async function aiFeedback(){
  const key=$('apiKey').value.trim();
  if(!key){$('aiStatus').className='status err';$('aiStatus').textContent='API 키를 먼저 입력해 주세요. 키가 없어도 2단계의 고정 안내로 계획을 완성할 수 있어요.';return}
  if($('remember').checked) sessionStorage.setItem('ckey',key);
  const st=$('aiStatus');st.className='status';st.textContent='AI 피드백 생성 중...';
  $('aiBtn').disabled=true;
  const l=LEVELS[state.level-1];
  const qa=QUESTIONS.map(q=>'['+q.id+'] '+q.label+'\n'+(state.answers[q.id]||'(미작성)').trim()).join('\n\n');
  const sys='당신은 대학생 진로 학습 계획 코치입니다. 학습자의 목표 레벨은 '+l.n+'단계 "'+l.name+'"입니다. 지침: '+l.ai+' 학습자의 답변을 대신 작성하지 마세요. 반드시 다음 JSON 형식만으로 응답하세요: {"feedbacks":[{"qid":"q1","comment":"피드백 1~2문장"},(q2,q3,q4 동일 구조)],"summary":"계획 전체 총평 2문장"}';
  try{
    const res=await fetch($('apiUrl').value.trim(),{method:'POST',
      headers:{'Content-Type':'application/json','Authorization':'Bearer '+key},
      body:JSON.stringify({model:$('apiModel').value.trim(),temperature:0.4,
        messages:[{role:'system',content:sys},{role:'user',content:'키우려는 역량: '+$('skill').value+'\n\n'+qa}]})});
    if(!res.ok){throw new Error(res.status===401?'API 키가 올바르지 않습니다 (401)':res.status===429?'사용량 한도 초과 (429)':'API 오류('+res.status+')')}
    const data=await res.json();
    const json=JSON.parse(data.choices[0].message.content.replace(/```json|```/g,'').trim());
    QUESTIONS.forEach(q=>{const f=(json.feedbacks||[]).find(x=>x.qid===q.id);const el=$('fb-'+q.id);
      if(f){el.innerHTML='<b>AI 피드백</b>\n'+esc(f.comment);el.classList.add('on')}});
    $('aiSummary').innerHTML='<b>총평</b>\n'+esc(json.summary||'');$('aiSummary').classList.add('on');
    st.className='status ok';st.textContent='피드백 완료. 참고하여 답변을 직접 수정해 보세요.';
  }catch(e){st.className='status err';st.textContent='실패: '+e.message}
  finally{$('aiBtn').disabled=false}
}
function exportMD(){
  const l=LEVELS[state.level-1];
  let md='# 나의 진로 학습 계획\n\n- 역량: '+$('skill').value+'\n- 목표 레벨: LV'+l.n+' '+l.name+'\n- 작성일: '+new Date().toLocaleDateString('ko-KR')+'\n\n';
  QUESTIONS.forEach(q=>{md+='## '+q.label+'\n'+((state.answers[q.id]||'').trim()||'(미작성)')+'\n\n'});
  md+='> 개인 학습용 문서이며 객관적 역량 인증이 아닙니다.\n';
  const a=document.createElement('a');
  a.href=URL.createObjectURL(new Blob([md],{type:'text/markdown'}));
  a.download='my-career-plan.md';a.click();
}
load();
renderLevels();renderQuestions();
$('skill').value=state.skill||'';
$('skill').oninput=e=>{state.skill=e.target.value;save()};
$('aiBtn').onclick=aiFeedback;
$('mdBtn').onclick=exportMD;
if(sessionStorage.getItem('ckey')){$('apiKey').value=sessionStorage.getItem('ckey');$('remember').checked=true}
</script>
</body>
</html>
```

---

## 적용 방법

1. 레포지토리의 `index.html`을 위 파일로 교체 후 커밋·푸시하면 GitHub Pages에 바로 반영됩니다. 기존의 `app.py`, `pathfinder_*.py`, 예시 JSON, stlite 로더는 모두 삭제해도 동작합니다.
2. **AI 제공자 교체:** 기본값은 OpenAI의 chat completions 엔드포인트이지만, OpenAI 호환 API라면 주소와 모델명만 바꿔 다른 제공자도 쓸 수 있습니다. 학생 입장에서는 사이트의 "AI 설정"에서 직접 입력하면 됩니다.
3. **질문 커스터마이징:** `QUESTIONS` 배열의 항목만 수정하면 질문 수·내용이 바뀌고, 레벨별 안내(`hintD`=절차형, `hintQ`=질문형)도 각 항목에 함께 정의됩니다.

---

## 핵심 동작 방식

* **레벨 연동:** `hintFor()`가 LV1~2는 절차형 안내(`hintD`), LV3는 둘 다, LV4~5는 질문형 안내(`hintQ`)만 노출합니다. AI 호출 시에는 `LEVELS` 배열의 `ai` 필드가 시스템 프롬프트에 그대로 주입되어, 같은 답변이어도 레벨에 따라 피드백 스타일이 달라집니다. 이것이 이전 버전에서 빠져 있던 "목표 수준별 차등 스캐폴딩"의 핵심 연결부입니다 (Unknown, n.d.).
* **AI 피드백 흐름:** 답변 4개와 역량, 목표 레벨을 하나의 요청으로 보내고, 응답을 JSON으로 파싱해 각 질문 바로 아래에 피드백 박스로 표시합니다. 401(키 오류)·429(한도 초과) 등의 오류는 한국어 안내로 안에 표시되며, 실패해도 고정 안내는 그대로 사용할 수 있어 AI 의존 없이 서비스가 끊기지 않습니다.
* **데이터 보존:** 답변·레벨·역량은 `localStorage`에 자동 저장되어 새로고침에도 유지되고, API 키는 체크박스를 켠 경우에만 `sessionStorage`에 보관되어 탭을 닫으면 사라집니다. 계획서는 마크다운 파일로 내려받을 수 있습니다.

---

## 운영 시 주의사항

기존 코드의 윤리 고지처럼, 학생 개인 키를 브라우저에서 직접 호출하는 방식은 시연·수업용으로는 적합하지만 공개 서비스로 확장할 때는 서버 측 프록시 방식으로 교체하는 것이 권장됩니다 (Unknown, n.d.). 또한 일부 API 제공자는 브라우저 직접 호출(CORS)을 제한할 수 있으니, 실제 사용할 제공자에서 한번 테스트해 보시기 바랍니다. 페이지 하단의 안내문에도 "피드백은 참고용이며 최종 판단은 학습자에게 있다"는 문구를 유지해, 계획을 AI가 대신 쓰지 않는다는 설계 철학을 그대로 담았습니다 (Unknown, n.d.).

---

## References

* Unknown. (n.d.). https://github.com/ys-fighteam/career-scaffolding/commit/d921d59b005ee8d1b4afd4e805fec791ad1b410c.patch