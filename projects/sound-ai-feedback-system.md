# Reference vs Performance Audio Analysis

기준(reference) 음원과 합주 녹음(performance)을 비교해서, 악기별(드럼/베이스/보컬/반주) 타이밍·피치·코드 차이 후보를 찾아 사람이 읽을 피드백으로 정리하는 시스템이다.

자동 분석 결과는 확정된 연주 오류가 아니다. Stem 분리, onset/downbeat 검출, pitch/chroma 추정, correspondence(시간축 대응) 오차가 섞여 있을 수 있으므로, 생성된 음원·그래프·CSV를 사람이 함께 확인해야 한다.

## 실행

```bash
python -m pip install 'setuptools>=77,<81' cython==3.3.0
python -m pip install -r requirements.txt
python -m pip install --no-build-isolation --no-deps rmvpe-onnx==0.2.3 \
  "madmom @ git+https://github.com/CPJKU/madmom.git@27f032e8947204902c675e5e341a3faf5dc86dae" \
  "adtof @ git+https://github.com/MZehren/ADTOF.git@b3968fb332f69b65ee07c089fc62f436503755db"  # see requirements.txt's own comment for why these are separate
python -m webapp.app
```

브라우저에서 `http://127.0.0.1:7860`(포트는 `GRADIO_SERVER_PORT`로 변경 가능)을 연다. Reference/Performance 음원 두 개를 올리고 CPU worker 수, stem 캐시 재사용 여부를 선택한 뒤 분석을 시작한다. 드럼/베이스 검출기(ADTOF/madmom)와 correspondence 방식(downbeat)은 검증된 조합으로 고정되어 있어 선택지가 아니다(2026-09-18). OpenAI API 키·모델은 사이드바에서 선택 입력하며, 키가 없으면 규칙 기반 피드백으로 대체된다.

지원 입력 형식: WAV, FLAC, MP3, M4A, OGG.

시작이 느리거나 멈춘 것처럼 보일 때는 [`docs/running_the_app.md`](docs/running_the_app.md)
(실제로 겪었던 증상·대응 정리)를 참고한다.

## 아키텍처

| 위치 | 역할 |
|---|---|
| 브라우저 | 음원 업로드, 실행 요청, 피드백 확인, 구간 청취 |
| 로컬 Python 서버 (`webapp/app.py`) | Gradio UI, 파이프라인 실행 제어, 결과 저장 |
| 로컬 CPU (`webapp/pipeline_v2.py` → `webapp/core/*`) | correspondence, stem별 onset/pitch/코드 분석, MuQ cosine 비교 |
| Hugging Face 원격 GPU (`remote_gpu_service/`) | BS-RoFormer 6-stem 분리, MuQ 인코딩, 보컬 RMVPE F0 추정 |
| OpenAI API (선택) | 구조화된 분석 결과를 자연어 피드백으로 요약. 음원 자체는 전달하지 않음 |

원격 GPU 기본 주소는 `https://uoseceif-remote-gpu-service.hf.space`이며 `REMOTE_GPU_SPACE_URL`로 바꿀 수 있다. 로컬 CUDA는 필요 없지만, 신규 분석에는 네트워크와 원격 GPU 접근이 필요하다.

### 파이프라인 단계

```text
Reference + Performance 업로드
        │
        ├─ 사전 준비 확인 (설치/다운로드 없이 의존성·경로만 점검)
        ├─ 입력 검증 (ffprobe, 형식·크기·길이)
        │
        ├─ 원격 GPU: BS-RoFormer 6-stem 분리 ─┐
        └─ 로컬 CPU: correspondence           ┤ (동시 실행)
                                               │
                correspondence 방식(기본 dtw_extrema):
                  MrMsDTW(chroma+DLNCO) + cliff-crop (H1)
                  → forced-envelope-extrema 보정 (Track B)
                실험적 대안(downbeat, opt-in):
                  레퍼런스/퍼포먼스 각자 독립 다운비트 검출 후 순번대로 대응
                                               │
        분리 + correspondence 완료 후 병렬 진행
        ├─ 로컬 CPU: drum/bass/vocal onset/pitch 분석 (탐색 반경은
        │    기준 음원 BPM에 비례해 유동적으로 결정, 각 스템별 검증된 상한 이내로 clamp)
        │    ├─ drum: ADTOF onset (또는 opt-in SuperFlux)
        │    ├─ bass: madmom onset (또는 opt-in RMS rising-edge)
        │    └─ vocal: RMVPE 기반 F0 + onset (원격 GPU에서 실행, CPU 체인과 병렬)
        ├─ 로컬 CPU: guitar/piano 근음 진단 (other+guitar+piano 믹스다운 → BTC 코드 인식,
        │    correspondence 정렬 프레임별 크로마 코사인 유사도로 화성 유사도 산출)
        └─ 원격 GPU: 대응 구간별(전 스템 공통, other 포함) MuQ 임베딩 추출 + cosine 유사도
                                               │
        마디(bar) 그리드 (madmom 다운비트 + correspondence 매핑)
        → onset 이벤트/MuQ 결과/근음 진단에 공통 region_id 태깅
                                               │
        AnalysisResult 조립 → OpenAI 구조화 피드백 (또는 규칙 기반 폴백)
                                               │
        결과 JSON 저장 → 브라우저에서 피드백 카드 + 대응 구간 청취
```

- **correspondence 방식**: GUI는 `downbeat`(레퍼런스/퍼포먼스 각자 독립 다운비트 검출 후 순번대로 대응)로 고정되어 있다(2026-09-18). 기존 기본값이었던 `dtw_extrema`(SyncToolbox MrMsDTW + H1 cliff-crop + Track B 강제극값 보정)는 실제 곡에서 dense correspondence p95 잔차 2.3초·극값 약 50% 미매칭이 재현되어(원인: 이 방식 자체의 한계) GUI 선택지에서 제외했다 — `run_pipeline()`을 직접 호출하는 코드는 여전히 `correspondence_method="dtw_extrema"`를 기본값으로 받아들인다.
- **드럼/베이스 검출기**: GUI는 팀원 벤치마크에서 정확도가 더 높게 나온 ADTOF(드럼)/madmom(베이스)로 고정되어 있다(2026-09-18) — 기존 DSP(SuperFlux/RMS) 옵트인은 GUI 선택지에서 제외했다.
- **마디 그리드**: madmom 다운비트를 correspondence 표에 매핑해 만든 공통 구간(`region_id`)을, MuQ 비교와 drum/bass/vocal 온셋 이벤트 태깅이 함께 사용한다. "들어보기" 재생도 이 구간을 우선 사용하고, 구간이 없으면 고정폭 창으로 폴백한다.
- **MuQ**: 기본 모드(`corresponding_clips`)는 correspondence로 정해진 구간마다 실제 오디오 클립을 그대로 잘라(워핑 없음) 쌍으로 인코딩·비교한다. `MUSIC_ENCODER_INPUT_MODE=stem_sequence`로 이전 전곡 시퀀스 풀링 방식으로 되돌릴 수 있다.
- **Guitar/Piano 근음**: 근음(root) 자체 일치 여부는 신뢰도가 낮음이 실측으로 확인되어 UI/피드백에서 쓰지 않는다 — 대신 correspondence로 정렬한 프레임별 크로마 코사인 유사도(`harmonic_similarity_mean`)만 사용한다.
- **Other(반주)**: 자체 코드 진행 일치도 비교 기능은 없다 — 다른 stem과 동일하게 MuQ 임베딩 유사도만 받는다(예전에 있던 `other` 전용 코드 일치율/경계 타이밍 비교는 실제 파이프라인에서 호출되지 않는 죽은 코드였음이 확인되어 2026-09-18에 제거했다).
- **온셋 탐색 반경**: drum/bass/vocal 매칭 반경은 기준 음원 BPM에서 유도한 값(각 스템별 실데이터로 검증된 최소/최대로 clamp)을 쓴다 — 고정값이 아니라 곡 템포에 맞춰 유동적으로 움직인다.
- **보컬 F0 필터**: RMVPE 원시 F0를 반음 단위로 양자화한 뒤 80ms 이하 지속 구간(비브라토·트래킹 노이즈)을 인접한 더 긴 구간에 흡수시킨 값을 피치 비교(이벤트 대표음, 리뷰 오디오)에만 쓴다 — 온셋 후보 검출 자체는 원시 F0를 그대로 쓴다.

## 코드 구조

```text
webapp/
├─ app.py                       # Gradio UI
├─ pipeline_v2.py               # 파이프라인 오케스트레이션
├─ preflight.py                 # 분석 전 환경 점검 (설치/다운로드 없음)
└─ core/                        # 재사용 가능한 단계별 로직
   ├─ config.py                 # AppSettings, 환경변수
   ├─ schemas.py                # AnalysisResult/FeedbackResult 등 결과 계약
   ├─ job_manager.py            # job별 작업 폴더
   ├─ command.py                # subprocess 실행과 로그
   ├─ feedback.py               # OpenAI structured output / 규칙 기반 폴백
   ├─ summarize.py              # MuQ 등 공용 요약
   ├─ artifacts.py              # 결과 JSON 저장
   ├─ dense_correspondence.py   # Track A (MrMsDTW + cliff-crop), in-process
   ├─ bar_regions.py            # 다운비트 기반 마디 그리드
   ├─ harmonic_audio.py         # other+guitar+piano 믹스다운
   ├─ other_playback.py         # Other 원속도 재생
   └─ stages/                   # separation/correspondence/downbeat_correspondence/
                                 # region_music_encoder/stem_music_encoder/... 어댑터

core_project/
├─ src/music_encoder/           # MuQ 비교 로직
└─ experiments/stem_performance_analysis/
   ├─ correspondence_onset_refinement/   # Track B (강제극값 보정)
   ├─ drum_bass_vocal_onset_matching/    # drum/bass/vocal onset 검출·매칭
   └─ other_chord_btc/                   # BTC 코드 인식 (기타/피아노 근음 진단용)

remote_gpu_service/             # BS-RoFormer 분리 + MuQ 인코딩 + 보컬 RMVPE F0 (원격 GPU Space)
scripts/                        # verify_*.py 등 개발용 도구
tests/                          # webapp/core_project 회귀 테스트
```

`core_project/experiments/` 아래 위 세 디렉토리만 현재 파이프라인이 실제로 호출한다 — 폴더 이름이 비슷해 보여도 이 셋 밖의 디렉토리를 추가할 때는 실제 호출 경로를 먼저 확인할 것.

## 환경변수

| 이름 | 기본값 | 의미 |
|---|---:|---|
| `AUDIO_COMPARISON_PROJECT_ROOT` | 저장소의 `core_project/` 자동 탐색 | 분석 코어 루트 |
| `AUDIO_COMPARISON_JOB_ROOT` | `jobs/` | 사용자 job 결과 저장 위치 |
| `AUDIO_COMPARISON_STEM_CACHE_DIR` | `stem_cache/` | 분리된 stem 캐시 |
| `GRADIO_SERVER_NAME` / `GRADIO_SERVER_PORT` | `0.0.0.0` / `7860` | Gradio 서버 바인딩 |
| `MAX_UPLOAD_MB` | `500` | 파일별 최대 업로드 크기 |
| `MAX_DURATION_SEC` | `900` | 파일별 최대 길이(초) |
| `MAX_CONCURRENT_JOBS` | `1` | GPU 보호용 동시 job 수 |
| `EVENT_ANALYSIS_WORKERS` | `1` | stem 분석 동시 실행 수 (1 또는 2) |
| `JOB_RETENTION_HOURS` | `24` | 오래된 job 정리 기준 |
| `OPENAI_MODEL` | `gpt-5-mini` | 자연어 피드백 생성 모델 |
| `REMOTE_GPU_SPACE_URL` | `https://uoseceif-remote-gpu-service.hf.space` | BS-RoFormer 분리·MuQ 인코딩·보컬 RMVPE F0 추정을 위임할 원격 GPU Space |
| `MUSIC_ENCODER_INPUT_MODE` | `corresponding_clips` | `stem_sequence`로 설정 시 이전 전곡 시퀀스 풀링 방식으로 복귀 |

## OpenAI API 설정

사이드바에 API 키를 입력하거나 `OPENAI_API_KEY` 환경변수로 설정한다. 키가 없으면 규칙 기반 피드백으로 자동 대체된다. OpenAI에는 음원·파일명·해시·job ID·로컬 경로를 보내지 않고, 구조화된 분석 수치·상태만 전송한다 (`store=False`, Pydantic `FeedbackResult` 구조로 파싱).

## 검증

```bash
python -m pytest tests/ -q
```

## 배포

현재 이 앱 자체를 배포하지 않는다 — 로컬에서 `python -m webapp.app`으로만 실행한다. `Dockerfile`/`scripts/build_bundle.py`(HF Space용 독립 배포 폴더 생성기)는 실제로 쓰이지 않아 제거했다(2026-09-30). 원격 GPU 연산(BS-RoFormer 분리·MuQ 인코딩·보컬 RMVPE F0)만 `remote_gpu_service/`를 통해 별도로 배포되어 있다.
