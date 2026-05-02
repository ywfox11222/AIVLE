# 에이블 (Able) — Streamlit 학습 가이드

> 음성 인식 AI 개인비서 ‘에이블’의 Streamlit 구현.
> 1단계(현재): UI 골격 + 데모 데이터로 동작 / 2단계: 노트북의 OpenAI 코드 연동.

---

## 0. 빠른 실행

```bash
# 1) 패키지 설치
pip install -r requirements.txt

# 2) 실행 (이 폴더에서)
streamlit run app.py
```

브라우저가 자동으로 열리지 않으면 터미널에 출력된 `http://localhost:8501` 을 직접 연다.

기본 로그인: 아이디 `kim.daeri` / 비번 `demopass`

---

## 1. 폴더 구조 한눈에

```
able_app/
├── app.py                       ← 진입점. 로그인 화면.
├── pages/                       ← Streamlit이 자동으로 사이드바 메뉴로 만들어줌
│   ├── 1_대시보드.py
│   ├── 2_회의_분석.py
│   ├── 3_To-Do_관리.py
│   ├── 4_캘린더.py
│   ├── 5_음성_비서.py
│   └── 6_설정.py
├── core/                        ← 비즈니스 로직 (페이지에서 import)
│   ├── auth.py                  로그인 검증
│   ├── llm.py                   OpenAI 호출 (1단계는 mock 함수)
│   ├── email_sender.py          SendGrid (노트북 미션 1과 동일)
│   ├── store.py                 session_state 관리 헬퍼
│   ├── models.py                dataclass 정의
│   └── demo_data.py             1단계 시드 데이터
├── assets/style.css             추가 커스텀 스타일
├── .streamlit/config.toml       Streamlit 테마 (미니멀 비즈니스 톤)
└── requirements.txt
```

### 왜 이렇게 나눴나
- **`pages/` 폴더**: 파일 하나가 메뉴 하나. 파일명 앞의 숫자(`1_`, `2_`...)가 사이드바 정렬 순서. 언더스코어는 공백으로 표시됨.
- **`core/`**: UI와 로직을 분리. 페이지 파일이 점점 커지는 걸 막기 위해 LLM 호출, 데이터 저장 같은 건 모듈로 빼둔다.
- **이렇게 하면 좋은 점**: 팀원이 페이지 하나씩 맡아 작업해도 충돌이 적다. 2단계에서 OpenAI 연동할 때도 `core/llm.py` 한 파일만 바꾸면 모든 페이지에 적용된다.

---

## 2. Streamlit 핵심 개념 5가지

이걸 알면 코드의 80%가 이해된다.

### 2-1. 위에서 아래로 다시 읽힌다 (rerun 모델)
Streamlit 스크립트는 **사용자가 위젯을 건드릴 때마다 처음부터 다시 실행된다.**
React처럼 “상태가 바뀌면 변경된 부분만 다시 그린다” 가 아니다. 그냥 통째로 다시 돈다.

```python
import streamlit as st
count = 0
if st.button("증가"):
    count += 1
st.write(count)   # 버튼을 눌러도 항상 1만 나옴 (스크립트가 재실행되며 count=0 초기화)
```

→ 그래서 “값을 유지하려면” `st.session_state`에 넣어야 한다.

### 2-2. session_state는 “페이지 새로고침을 견디는 변수 저장소”
브라우저 탭이 살아 있는 동안 유지된다. 모든 페이지에서 공유된다.

```python
import streamlit as st

# 첫 진입 시 초기화
if "count" not in st.session_state:
    st.session_state.count = 0

if st.button("증가"):
    st.session_state.count += 1   # 누를 때마다 +1 됨

st.write(st.session_state.count)
```

이 프로젝트에서는 `core/store.py`가 이 패턴을 캡슐화한다. (예: `store.get_todos()`, `store.add_todo(...)`)

### 2-3. 위젯에는 `key`를 주는 게 안전
같은 페이지에 같은 위젯이 여러 개면 충돌난다. 그리고 `key="이름"` 을 주면 자동으로 `st.session_state["이름"]` 에 값이 들어간다.

```python
st.text_input("이메일", key="email_input")
# → 사용자가 입력한 값을 st.session_state.email_input 으로 어디서든 읽을 수 있음
```

### 2-4. `st.rerun()` 으로 강제 재실행
위젯 외부에서 상태를 바꾼 뒤 즉시 화면을 갱신하고 싶을 때 쓴다.

```python
if st.button("로그아웃"):
    st.session_state.clear()
    st.rerun()
```

### 2-5. 멀티페이지 라우팅은 폴더 컨벤션
`pages/` 폴더 안의 .py 파일이 자동으로 메뉴가 된다.
페이지 간 이동은 두 가지:
```python
st.switch_page("pages/2_회의_분석.py")  # 즉시 이동
st.page_link("pages/2_회의_분석.py", label="회의 분석")  # 클릭 가능한 링크
```

---

## 3. 기타 자주 쓰는 위젯 치트시트

| 하고 싶은 것 | 코드 |
|---|---|
| 제목 | `st.title("제목")` / `st.header(...)` / `st.subheader(...)` |
| 본문 | `st.write("...")` 또는 `st.markdown("...")` |
| 입력칸 | `st.text_input("라벨", key="...")` |
| 비밀번호 입력 | `st.text_input("PW", type="password")` |
| 버튼 | `if st.button("클릭"): ...` |
| 체크박스 | `st.checkbox("동의", key="...")` |
| 셀렉트 박스 | `st.selectbox("선택", ["A","B"])` |
| 라디오 | `st.radio("뷰", ["리스트","매트릭스"], horizontal=True)` |
| 파일 업로드 | `st.file_uploader("파일", type=["m4a","wav","txt"])` |
| 음성 녹음 | `st.audio_input("녹음")` (Streamlit 1.31+) |
| 알림 박스 | `st.info(...)`, `st.success(...)`, `st.warning(...)`, `st.error(...)` |
| 토스트 | `st.toast("저장됨", icon="✅")` |
| 컬럼 분할 | `c1, c2 = st.columns(2)` 후 `with c1: ...` |
| 탭 | `t1, t2 = st.tabs(["녹음","파일"])` |
| 사이드바 | `with st.sidebar: ...` |
| 컨테이너/카드 | `with st.container(border=True): ...` |
| 표 | `st.dataframe(df)` 또는 `st.data_editor(df, num_rows="dynamic")` |
| 진행 표시 | `with st.spinner("분석 중..."): ...` |
| 오디오 재생 | `st.audio(audio_bytes, format="audio/mp3")` |

---

## 4. 노트북 코드와의 매핑

| 노트북 미션 | 어디에 살릴까 |
|---|---|
| 미션 1: `send_email()` | `core/email_sender.py` (그대로 복붙) → 회의분석/음성비서/설정에서 호출 |
| 미션 2: 기본 챗봇 | `core/llm.py` 의 `chat(prompt, sys_role)` |
| 미션 3: 미팅 요약 + 김 대리 업무 | `core/llm.py` 의 `analyze_meeting(text)` (회의분석 페이지에서 호출) |
| 미션 4: STT (Whisper) | `core/llm.py` 의 `stt(audio_bytes)` |
| 미션 5: TTS | `core/llm.py` 의 `tts(text, voice="alloy")` |
| 미션 6: 음성 → 음성 | 회의분석 + 음성비서 페이지의 조합 |
| 미션 7: 도구 사용 (이메일 자동 발송) | `core/llm.py` 의 `chat_with_tools(...)` |

> **1단계에서는** `core/llm.py` 의 함수가 모두 “mock(가짜) 응답”을 반환한다. UI 흐름만 먼저 검증.
> **2단계에서는** mock 자리에 노트북의 OpenAI 호출 코드를 그대로 옮겨 넣으면 된다. (자세히는 `2단계_연동가이드.md` 참고)

---

## 5. 학습 순서 추천

코드 양이 많아 보여도 핵심은 5개 파일이야.

1. **`app.py`** — 가장 단순. 로그인만 한다. session_state 패턴 익히기.
2. **`core/store.py`** — “상태를 관리한다”는 게 뭔지. session_state 캡슐화 예시.
3. **`pages/1_대시보드.py`** — 카드 레이아웃, columns, container(border) 사용법.
4. **`pages/2_회의_분석.py`** — 가장 복잡. 탭, 파일 업로드, 분석 결과 4분할, 액션 버튼.
5. **`pages/5_음성_비서.py`** — `st.audio`로 TTS 결과 재생, 배속 컨트롤.

그 다음에 To-Do, 캘린더, 설정 보면 패턴이 반복돼서 쉬워.

---

## 6. 자주 막히는 포인트

- **“바뀐 게 화면에 안 나와요”** → `st.rerun()` 호출했는지 / session_state에 저장했는지 확인.
- **“두 번 누르면 동작해요”** → rerun 모델 때문. 버튼 클릭 직후 session_state를 바꾸고 그 결과를 같은 실행에서 읽으면 한 박자 늦게 반영될 수 있다. 가장 깔끔한 해결: 변경 후 `st.rerun()`.
- **“pages 폴더 안 파일이 메뉴에 안 떠요”** → 파일명에 한글/특수문자가 있으면 가끔 인식이 안 된다. 숫자 prefix는 꼭 붙이고, OS별로 안 되면 영문명으로 바꿔본다.
- **“secrets에 넣은 API 키가 안 읽혀요”** → `.streamlit/secrets.toml` 위치는 `app.py` 와 같은 디렉토리 기준 `.streamlit/`. `st.secrets["OPENAI_API_KEY"]` 로 접근.

---

## 7. 다음 단계

이 1단계로 UI/플로우 검증이 끝나면, `2단계_연동가이드.md` 를 보고 노트북의 OpenAI/SendGrid 코드를 `core/llm.py`, `core/email_sender.py` 에 채워 넣으면 된다.
