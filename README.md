# 📊 ADsP 핵심요약 퀴즈

**ADsP(데이터분석 준전문가)** 자격시험 핵심 요약 문제를 풀면서 공부하는 Streamlit 퀴즈 앱이에요.
데이터 이해, 데이터 분석 기획, 데이터 분석 과목별 문제를 SQLite DB로 관리해요.

## 기능

- 과목별로 문제를 골라서 풀기
- 채점 결과와 함께 해설 확인
- CSV 데이터가 DB보다 최신이면 앱 실행 시 **자동으로 DB를 재생성**
  (Streamlit Cloud처럼 `git pull`만으로 코드가 갱신되는 배포 환경에서, 예전 DB가 새 CSV 내용을
  반영 못 하는 문제를 막기 위함)

## 기술 스택

| 기술 | 역할 |
|---|---|
| **Streamlit** | 퀴즈 화면 |
| **SQLite** | 문제은행 저장 (`db.py`, `build_db.py`) |
| **pandas / CSV** | `data/questions.csv`, `data/cbt_questions.csv`가 원본 문제 데이터 |

## 파일 구조

```
adsp_quiz/
├── app.py                    # Streamlit 앱 진입점
├── db.py                      # SQLite 연결 및 조회
├── build_db.py                 # CSV → SQLite DB 빌드
├── generate_questions.py       # 문제 데이터 생성/정리 스크립트
├── generate_cbt_questions.py    # CBT 문제은행 데이터 생성/정리 스크립트
├── cbt_source_44_49.py          # CBT 원본(44~49회) 자료 처리
├── data/
│   ├── questions.csv              # 정리된 핵심요약 문제
│   └── cbt_questions.csv          # CBT(컴퓨터 기반 시험) 문제은행
└── requirements.txt
```

## 실행하기

```bash
pip install -r requirements.txt
streamlit run app.py
```

## 트러블슈팅

**① Streamlit Cloud 배포 후 헬스체크가 계속 실패함**

- 원인: `.streamlit/config.toml`에 로컬 개발용 포트(8504)가 `[server]` 섹션에 하드코딩돼
  있어서, 플랫폼이 기본 포트(8501)로 보내는 헬스체크가 계속 실패했음.
- 해결: `[server]` 설정 자체를 지워서 플랫폼이 포트를 알아서 관리하게 함 (로컬 실행 포트는
  실행 스크립트의 CLI 인자로 따로 지정).

**② `streamlit>=1.61`에서 헬스체크마다 500 에러**

- 원인: 내장 gzip 미들웨어가 새 `starlette`(1.4.0+)가 요구하는 `thread_minimum_size` 인자
  없이 `GZipResponder`를 호출해서 크래시.
- 해결: `requirements.txt`에 `starlette<1.4.0` 상한 고정.

**③ CSV를 갱신해서 푸시해도 배포 환경엔 예전 문제만 계속 나옴**

- 원인: DB 파일(`quiz.db`)이 이미 존재하면 무조건 재생성을 건너뛰는 로직이었는데,
  Streamlit Cloud는 `git pull`만으로 코드(CSV 포함)를 갱신하고 DB 파일은 그대로 남아있어서
  코드와 데이터가 어긋났음(CBT 문제가 0개로 보이는 등).
- 해결: CSV 파일의 수정 시각이 DB보다 최신이면 재생성하도록 변경.

**④ 오답노트의 "다시 풀기" 버튼을 눌러도 퀴즈 탭으로 안 넘어감**

- 원인: 버튼 콜백에서 `ss.nav = "퀴즈"`처럼 **`key`가 바인딩된 위젯의 `session_state` 값을
  같은 스크립트 실행 중에 직접 재할당**하려 했는데, 이건 Streamlit이 막고 있는 패턴이라
  조용히 무시됨.
- 해결: 대신 `ss["_pending_nav"] = "퀴즈"`로 "예약"만 해두고, 다음 렌더링 맨 앞에서 그 예약값을
  읽어 실제 `nav` 위젯이 그려지기 *전에* 반영하는 pending-nav 패턴으로 변경.

**⑤ 실전 시험 모드에서 1→2→3과목 순서가 아니라 문제가 뒤죽박죽으로 나옴**

- 원인: 과목별로 문항을 뽑은 다음, 전체 리스트를 한 번 더 `random.shuffle()`해서 과목 순서까지
  섞여버림.
- 해결: 과목 **내부**에서만 무작위로 섞고, 과목 순서(1→2→3) 자체는 그대로 유지하도록 마지막
  전체 셔플을 제거.

```diff
      subj_ids = [random.choice(groups[s][c]) for c in core_ids[:n]]
      ids.extend(subj_ids)
- random.shuffle(ids)
  return ids
```

---

🤖 이 저장소의 README는 Claude Code와 함께 작성했어요.
