# 김민석 / Portfolio

전자전기컴퓨터공학부에서 공부하며 컴퓨터공학 분야의 연구와 프로젝트를 수행했습니다. AI 모델 실험, AI Agent 연동, 백엔드 개발 등 서로 다른 형태의 프로젝트를 경험했습니다. 새로운 기술을 사용할 때 단순히 결과를 받아들이기보다, 실제 코드의 실행 흐름과 데이터·결과를 직접 확인하며 필요한 범위까지 구현하는 것을 중요하게 생각합니다.

## Projects

### Vibee — AI 개발 지원 앱

비전공자와 초보 개발자가 Codex 또는 Claude Code로 프로젝트를 진행할 때, 최초 요구사항과 기술 결정의 문맥을 잃지 않도록 돕는 로컬 연동 앱입니다. 구현 전에는 대화형 인터뷰로 요구사항을 구조화하고, 구현 후에는 설계 이탈·구조 개선점 확인 같은 기능을 제공합니다.

**Role**: 2인 팀 팀장 · 콘셉트/기능/UX 설계 · Agent–App 연결 구조 구현

**What I Did**
- AI가 제시한 "한 번에 앱 설계도 생성" 방식을 반려하고, 한 번에 하나씩 질문하는 멀티턴 인터뷰 방식으로 설계 흐름을 바꿈
- 사용자가 직접 정한 내용과 AI가 보완한 내용을 구분해서 관리하도록 데이터 구조 설계
- App Server(에이전트 실행) / WebSocket(진행 이벤트) / MCP(문맥 조회·결과 반환)로 역할을 나눠 Agent–App 연결 구현
- Codex와 Claude의 서로 다른 이벤트를 공통 형식으로 변환
- 브라우저 요청 → 실제 파일 수정 → 진행 이벤트 → MCP 호출 → 결과 반환까지, Codex·Claude 각각 동일한 9개 항목으로 동작 검증

**Repository**: https://github.com/PIPYI/vibee

---

### 지역 미션 여행 앱 — 한국관광공사 오픈API 활용

대중교통 여행자가 버스를 기다리는 시간 동안 주변 지역의 미션을 수행할 수 있도록 하는 앱을 개발하고 있습니다(현재 개발 중).

**Role**: 팀장 · 사용자 흐름/데이터 관계 설계 · 백엔드 구현(협업자와 절반씩 분담, 공통 기반·인증·미션 진행 담당)

**What I Did**
- 회원가입 → 장소 탐색 → 미션 진행 → 인증까지 사용자 흐름을 먼저 설계도로 정리하고, 사용자·장소·미션·진행·인증의 데이터 관계와 API 구조를 구체화
- Spring Boot 기반 백엔드 코드를 직접 작성 (Spring Security/JWT/BCrypt, PostgreSQL/Flyway 같은 구체적인 구현 방법은 AI의 코드 가이드와 피드백을 참고해 적용)
- 인증 반려(`Rejected`)와 미션 진행 실패(`Failed`)를 같은 상태로 처리하던 구조를 분리해, 반려된 인증도 같은 진행 건에 재제출할 수 있도록 흐름을 수정
- 동일 사용자가 같은 미션을 중복 진행하지 못하도록 동시성 처리 적용, 소유권·상태 기반 예외 처리 구현

**Repository**: _(추후 업데이트)_

---

### Sound AI 합주 피드백 시스템

기준 음원과 합주·라이브 연주를 악기 stem 단위로 비교해 연주 차이를 분석하는 시스템입니다. 학교 작품 경진대회에서 장려상(4등상)을 수상했습니다 — 소프트웨어 7팀·하드웨어 약 20팀이 함께 출전해 통합 수상한 대회로, 상위 3개 상은 모두 하드웨어 팀이 받았고 본 팀은 수상팀 중 유일한 소프트웨어 팀이었습니다.

**Role**: 평가 기준/실험 흐름 설계 · AI Agent를 활용한 반복 실험 · 실험 결과 검증

**What I Did**
- 여러 Sound AI 모델을 비교하기 위한 평가 기준을 설정하고, 동일 입력·동일 조건에서 실험이 이루어지는지 확인
- 실패한 실험이 결과에서 누락되지 않았는지, 모델별 출력 형식이 서로 비교 가능한지 점검
- stem별 활성 구간 분석, downbeat 기반 정렬, DSP 기반 구간 비교, MuQ 기반 유사도 검증, DSP·MuQ 결합 매칭 실험을 진행
- AI Agent에는 반복 실행과 결과 정리를 맡기고, 실험 조건과 결과 채택 여부는 직접 확인

**Repository**: 팀 공유 비공개 저장소 (공개 전환 협의 중, 확정되면 링크 추가 예정)

---

### Computer Vision Research — 컴퓨터비전 연구실 학부연구생

컴퓨터비전 연구실에서 6개월간 학부연구생으로 활동하며 논문·방법론을 학습해 발표하고, 연구 주제를 코드로 구현해 랩미팅에서 결과를 공유했습니다. 연구실 활동의 일환으로 **제2회 자율주행 AI 챌린지**의 Semantic Segmentation 과제에 참여했습니다.

**Role**: 학부연구생 · Semantic Segmentation 구현·실험

**What I Did**
- DINOv2를 teacher model로 사용하는 teacher-student 구조와 KL divergence 기반 loss를 구현
- 서로 다른 backbone과 segmentation head를 결합했을 때 feature map 크기·채널 수가 맞지 않던 문제를, 각 모델의 forward 과정과 tensor shape를 직접 추적해 해결
- 여러 backbone·segmentation head 조합의 정확도뿐 아니라 FLOPs·FPS까지 비교 정리
- 목표한 성능에는 도달하지 못했지만, 결과를 임의로 해석하지 않고 확인된 실험 조건과 결과를 구분해 보고

**Repository**: 비공개 연구실 프로젝트로 공개 저장소 없음

## Skills / Experience

**AI / ML**
Python · PyTorch · Computer Vision · Semantic Segmentation · Sound AI · AI Agent 활용 및 검증

**Backend**
Java · Spring Boot · PostgreSQL · REST API · 사용자 흐름 및 데이터 관계 설계

**AI Agent / Integration**
Codex · Claude Code · MCP · WebSocket · AI Agent 결과 검증
