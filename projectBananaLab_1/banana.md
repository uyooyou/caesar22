# 사내 문서 기반 AI 챗봇 서비스 요구사항 명세서 (MVP)

## 1. 기획 배경
- 사내 문서(규정집, 매뉴얼, 보고서 등) 검색 효율이 낮아 정보 탐색 비용이 큼 → 업무 생산성 저하.
- 범용 LLM은 내부 데이터/맥락에 최적화되지 않음 → 사내 맞춤형 AI 챗봇 필요.
- AI + RAG + Agent로 정확한 답변과 실사용 가능한 자동화를 목표로 하며 Notion/Slack/Email 등 연동.

## 2. 서비스 소개
- 내부 문서를 업로드/학습하고, 자연어 질문에 대해 즉시 답변(출처 포함)을 제공하는 챗봇.
- 문서 업로드 & 학습(PDF/DOCX/TXT), 실시간 질의응답(LLM+RAG), 출처 기반 답변, 외부 서비스 연동, 채팅 기록 저장.

## 3. 범위 (MVP)
- 문서 업로드/처리: 텍스트 추출 → 청크 분할(500~1000자, 100자 overlap) → 임베딩 → Vector DB 저장.
- 검색형 Q&A: 쿼리 임베딩 → Top-k 검색 → 컨텍스트 구성 → 답변 생성(출처 포함).
- 외부 연동(선택 1~2채널): Slack, Email 우선. 추후 Notion/기타 확장.
- 채팅 이력 저장/조회: 사용자별 최근 N건.
- 관리: /health, /stats, /search.

## 4. 개발 범위 기획 및 리소스
- 회사 리소스
  - 웹문서화: 위키, 지라, 노션, 구글독스
  - 로컬문서: Word, Excel, PDF, …
  - 메일: Gmail/사내메일/MS 등
  - 메신저: Slack/MS 등
  - 코드 형상관리: GitHub
  - DB(MySQL/MongoDB/Elasticsearch 등): 서비스/분석/기타
  - API 통해 수집 가능 항목은 점진적 통합
- 회사 업무 자동화(오픈된 환경 위주)
  - 회의실 예약, 메일 전송, 문서화/데이터 분석 자동화, 전자 결제(후순위)
- 개인 문서
  - 개인 데이터 업로드도 고려(개인/공용 분리 저장 및 권한)
- 기능 개선 후보(후순위)
  - DB/코드형상관리/추가 서비스 범위 확장, 전자결제 연동, 온프레미스 LLM(SLM)

## 5. 역할 분담(예시)
- FE & BE(Agent 코어): 1명 — ReAct/그래프 에이전트 코어, 툴 통합
- BE: 1명 — 히스토리/메타데이터 DB 및 API 개발
- FE: 1명 — 어드민/채팅 UI, 로그인 및 설정 화면
- Tool 개발: 3명 — email/slack/rag 등 도구 개발, 더미 도구/더미 에이전트로 사전 테스트
- MCP 마켓/연결: 1명 — MCP 리서치/통합

## 6. 아키텍처 개요
- Frontend: React(어드민, 채팅장, 프로필/설정, 로그인)
- Backend: FastAPI(경량 API), Spring(비-AI CRUD/멤버 관리 등 연동 가능시)
- Agent: LangChain 기반 ReAct/Graph Agent, 도구 호출 구조
- RAG: 임베딩/검색(Chroma/FAISS/Pinecone 등) + LLM 생성
- DB: MySQL(계정/채팅 기록), MongoDB(문서 메타), Redis(캐시), Vector DB(Pinecone/Chroma)
- 스트리밍: SSE 기반 응답 스트리밍(우선순위 높음)

## 7. 기술 스택
- LLM: GPT-4o, LLaMA-3, Mistral, KoAlpaca(한국어)
- Embedding: OpenAI text-embedding-3-small/large 또는 sentence-transformers 대체
- RAG: LangChain/LlamaIndex
- Vector DB: 로컬(Chroma/FAISS), 클라우드(Pinecone/Weaviate)
- Agent: LangChain Agent(ReAct), LangGraph 검토
- Frontend: React + JavaScript
- Backend: Python + FastAPI, (Spring은 계정/권한/CRUD 확장 시 고려)
- DB: MySQL, MongoDB, Redis, Vector DB

## 8. 에이전트/툴 구조
- System Prompt: 목적/역할/제약 명시
- Tools(예시):
  - rag.py: 쿼리 임베딩, 유사도 검색, 컨텍스트 반환
  - email.py: 메일 전송(더미→실서비스 전환)
  - slack.py: 메시지 전송(더미→웹훅/봇 토큰)
  - 기타: 회의실 예약, 문서화 자동화 등 확장
- 개발 전 더미 도구/더미 에이전트로 IO 계약 테스트
- 폴더 구조 제안: `agent/`(에이전트 코어), `tools/`(각 툴), `services/`(RAG/임베딩), `apis/`

## 9. RAG 설계
- 문서 파이프라인: 업로드 → 텍스트 추출(pdf/docx/txt) → 정규화 → 청크 분할 → 임베딩 → Vector DB 저장
- 검색: 쿼리 임베딩 → Top-k(기본 5) → rerank(옵션) → 컨텍스트 생성 → LLM 생성
- 응답: 핵심 답변 + 출처(문서명/페이지·섹션/링크)
- Fallback: LLM 미사용/오류 시 규칙 기반 요약 및 근거 부족 표시

## 10. 기능 요구사항
1) 문서 업로드/처리
- 다중 파일 업로드, 부분 성공 처리, 메타데이터(파일명/업로더/업시각/페이지 등)
2) 검색/질의응답
- 한국어 우선, 영어 혼용 허용. 출처 필수 포함.
3) 채팅 이력
- user_id, question, answer, sources(JSON), created_at 저장/조회
4) 외부 연동(선택)
- Slack/Email 1~2채널 우선, 환경변수 기반 설정, 실패 로깅/재시도
5) 관리/모니터링
- /health, /stats, /search 제공
6) 스트리밍
- SSE 기반 답변 토큰 스트리밍, 타임아웃/중단 처리

## 11. 비기능 요구사항
- 성능: 평균 2~5초(모델/네트워크에 따라 변동)
- 보안: 내부망 가정, API Key 최소 보호(추후 사내 인증/권한)
- 개인정보/기밀 마스킹 고려, 감사 로그
- 구성 가능성: 임베딩/LLM/Vector DB 교체 가능
- 이식성: Windows/Ubuntu 로컬 실행

## 12. API 설계(초안)
- POST /documents: 파일 업로드 → 추출/분할/임베딩/색인
- GET /search?q=...&k=5: 유사 청크 검색
- POST /chat: { user_id, question, history? } → 답변+출처 저장
- GET /history?user_id=...&limit=20: 최근 대화 조회
- GET /health: 헬스 체크
- GET /stats: 색인/모델/버전 정보
- GET /chat/stream (SSE): 질문 스트리밍 응답

## 13. 데이터 모델(요약)
- Vector 문서(청크): id, text, embedding, metadata{ file_name, page, section, uploaded_by, uploaded_at }
- ChatHistory: id, user_id, question, answer, sources(json), created_at
- FileMetadata: file_id, file_name, mime_type, size, uploaded_by, uploaded_at, processed_at

## 14. 보안/권한(초안)
- 로그인은 간단 아이디 기반(데모), 어드민 권한으로 업로드/설정 접근
- 개인/공용 데이터 분리 저장 및 조회 제어(테넌시 확장 고려)

## 15. FE 요구사항(요약)
- 어드민 페이지: 공용 데이터 업로드/설정/권한(간단 로그인)
- 채팅장: Q&A, 출처 뱃지, SSE 스트리밍, 히스토리 보기
- 프로필 설정: 개인 데이터 업로드, 도구/API 키 설정 페이지
- 로그인: 경량 아이디 로그인

## 16. 일정(WBS, 1개월/4명)
- 1주차: 요구사항·아키·DB/Vector 설계, UI 와이어프레임
- 2주차: FastAPI 기본 API, 업로드→임베딩→벡터저장, RAG 프로토타입
- 3주차: React UI, 채팅 이력, Agent 구성(ReAct/툴), Slack/Email 1종 연동
- 4주차: 통합/테스트, RAG/Agent 튜닝, UI 개선, 사용자 테스트 및 데모 배포

## 17. 목표/진행 전략
- 역할 분담을 빠르게 확정
- 각 핵심 기술 테스트 스크립트로 품질 기준 합의
- 정확도보다 버그 없는 기본 기능 우선 구현 → 통합 → 고도화
- CLI로 모듈별 검증 후 서비스 통합(python module.py)

## 18. MCP 사용 가이드(요약)
- MCP 클라이언트: LangChain MultiServerMCPClient 활용
- 참고: `https://github.com/langchain-ai/langchain-mcp-adapters`, `https://smithery.ai/`, `https://wikidocs.net/286989`
- Cursor에서 MCP: `https://docs.cursor.com/ko/tools/mcp` (context7 연동 참고)
- 전략: 개발 시 최신 개발 문서 소스로 MCP 호출, 툴/Agent 테스트 자동화에 활용

## 19. 품질/평가 기준
- 실패/예외 처리 명확(부분 성공, 오류 메시지 구체화)
- 답변에 출처 최소 1개 이상(없으면 근거 부족 안내)
- 한글 처리(정규화/문장 분리) 품질 확보

## 20. 향후 과제(로드맵)
- 권한/테넌트 기반 접근 제어, 보안 강화
- 다국어 임베딩/LLM 스위칭, 프롬프트 최적화
- Notion/Slack/Email 정식 양방향 연동 및 업무 자동화 확장
- 자동 평가(Eval) 파이프라인/피드백 루프
- 운영 콘솔/모니터링 고도화, 온프레미스 LLM

---
본 문서는 MVP 기준의 요구사항 명세서이며, 구현 과정에서 합리적 범위 내에서 조정될 수 있습니다.
