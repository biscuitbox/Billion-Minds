# Billion Minds

> 4K 캔버스 위에 매시간 새로 칠해지는 829만 픽셀로 던지는 질문 —
> *과연 인류의 마음은 하나로 모일 수 있는가?*

## 프로젝트 소개

**Billion Minds**는 4K UHD 해상도(3,840 × 2,160 = **8,294,400픽셀**)의 캔버스 위에
매 시간마다 무작위 색깔을 새로 칠하는 웹 기반 수학적 확률 실험입니다.

매 시간 새로 그려진 그림은 **최초의 그림(Genesis Frame)** 과 픽셀 단위로 비교되며,
두 그림 사이의 **유사도(Similarity)** 가 백분율로 기록됩니다.
시간이 흐르며 누적되는 유사도 변화율은 꺾은선 그래프로 시각화되어,
"무한한 무작위성 속에서 동일한 결과는 얼마나 자주 나타나는가?"를 관측하게 합니다.

이 프로젝트의 본질적인 질문은 단순합니다.

> **인류의 마음, 혹은 목적은 과연 일치될 수 있는가?**

> 프로젝트 이름의 "Billion"은 80억 인류의 마음을 향한 **철학적 은유**입니다.
> 실제 캔버스는 표준 모니터에서 표시 가능한 최대치인 4K UHD(약 829만 픽셀)로 구현됩니다.

---

## 왜 4K인가? — 픽셀 상한 결정 근거

초기 구상은 10억 픽셀이었으나, 실제 구현 가능성 검토 결과 다음과 같은 한계로 인해 **4K UHD를 상한**으로 결정했습니다.

| 항목 | 10억 픽셀 (31,623×31,623) | **4K UHD (3,840×2,160)** |
|---|---|---|
| 총 픽셀 수 | 1,000,000,000 | **8,294,400** |
| 24bit 원본 크기 / frame | 약 2.79 GB | 약 23.7 MB |
| 1년치 (시간당 갱신) 저장량 | 약 24.5 TB | 약 208 GB |
| 브라우저 Canvas 제한 | ❌ 대부분 16,384px 한계 초과 | ✅ 네이티브 렌더링 가능 |
| GPU 단일 텍스처 메모리 | 약 4 GB | 약 32 MB |
| 픽셀별 비교 처리 시간 | 분 단위 | 수십 ms |

4K는 사용자가 **물리적으로 한 화면에 담을 수 있는 최대 픽셀 수**이자,
브라우저·GPU·스토리지 모두에서 실시간 처리가 가능한 현실적 상한입니다.

---

## 핵심 컨셉

- **Genesis Frame** — 사이트가 처음 생성한 8,294,400픽셀의 무작위 그림. 모든 비교의 기준점.
- **Hourly Refresh** — 매 정시(00분)마다 4K 캔버스를 무작위 색으로 새로 칠합니다.
- **Similarity Score** — 매 갱신마다 Genesis Frame과 픽셀별로 비교하여 일치율(%)을 산출합니다.
- **Convergence Graph** — 누적된 유사도를 꺾은선 그래프로 표시하여 변화율을 추적합니다.

---

## 수학적 배경

각 픽셀이 24bit 색상(약 1,677만 색)을 무작위로 가질 때,
임의의 두 픽셀이 정확히 같은 색일 확률은 약 $\frac{1}{2^{24}} \approx 5.96 \times 10^{-8}$ 입니다.

4K 캔버스(N = 8,294,400 픽셀) 전체의 유사도 기댓값은:

$$
E[\text{Similarity}] = \frac{1}{2^{24}} \approx 5.96 \times 10^{-8} \approx 0.00000596\%
$$

기댓값으로 일치하는 픽셀 수는:

$$
E[\text{Matches}] = N \times \frac{1}{2^{24}} = 8{,}294{,}400 \times \frac{1}{16{,}777{,}216} \approx 0.494
$$

즉, **두 4K 프레임 사이에서 평균적으로 약 0.5개의 픽셀만이 우연히 일치**합니다.
대부분의 갱신에서는 0개 또는 1개 픽셀만 일치하며, 2개 이상 일치할 확률은 약 9% 수준에 불과합니다(포아송 근사).

이 미세한 확률을 시간에 따라 관측함으로써, "수학적 우연"과 "의미 있는 수렴" 사이의 거리를 시각화합니다.

---

## 결정성과 시드 (Determinism & Seeding)

프로젝트의 모든 무작위 프레임은 **시드(seed)로 재생성 가능**해야 합니다. 이는 저장 비용 절감(오래된 PNG 폐기 후 시드만 보관)과 검증 가능성을 위한 핵심 사양입니다.

- **PRNG 알고리즘**: `numpy.random.Generator(PCG64)` 채택.
  - 채택 이유: 통계적 품질이 입증되어 있고, 멀티스트림(`SeedSequence.spawn()`) 지원으로 frame별 독립적 seed 파생이 가능하며, NumPy가 명시적으로 재현성을 보장합니다.
- **Seed 형식**: `(genesis_seed_hex, frame_index)` 튜플.
  - `genesis_seed_hex`: `secrets.token_hex(32)` 로 단 1회 생성. 256비트 엔트로피.
  - `frame_index`: 정수 (0 = Genesis, 1, 2, ... = 시간순 갱신).
  - 실제 PRNG 시드는 `SeedSequence(int(genesis_seed_hex, 16)).spawn(frame_index + 1)[-1]` 로 파생.
- **버전 정책**: 모든 frame 메타에 `prng_version` 필드 기록 (예: `"numpy-2.x-pcg64"`).
  - PRNG 라이브러리 메이저 버전 변경 시 비트 단위 재현이 깨질 수 있음을 인정하고, 해당 frame은 `archived` 플래그로 표시. PNG 원본이 남아있는 동안만 실제 픽셀 비교 가능.
- **Genesis seed 보관**: 분실 시 프로젝트 종료. 매니저 환경 변수 + 별도 오프라인 백업 2곳 저장 의무 (운영 §참조).

---

## 유사도 정의 (Similarity Definition)

- **기본 지표 = Exact Match** (절대 변경 금지).
  - 픽셀 일치 조건: `R == R' AND G == G' AND B == B'`.
  - 출력: `match_count` (0 ~ 8,294,400 사이 정수), `similarity_pct = 100 * match_count / 8294400`.
- **선택 지표** (Phase 4 이후 토글 옵션):
  - **Euclidean RGB distance** — 픽셀당 $\sqrt{(\Delta R)^2 + (\Delta G)^2 + (\Delta B)^2}$, 평균/중앙값/95퍼센타일 통계 산출.
  - **CIEDE2000 (ΔE)** — 인지적 색차 기반. 무거우므로 옵션.
- 선택 지표는 그래프에서 **추가 라인**으로 토글 표시. 기본 라인(exact match)은 항상 노출.

---

## 저장소 스키마 (Storage Schema)

데이터를 두 계층으로 분리합니다. **메타데이터는 PostgreSQL**, **이진 이미지는 S3 호환 스토리지**.

### PostgreSQL (메타데이터)

```sql
-- Genesis Frame (단 1행만 존재)
CREATE TABLE genesis (
  seed_hex        TEXT PRIMARY KEY,
  created_at      TIMESTAMPTZ NOT NULL,
  prng_version    TEXT NOT NULL,
  image_object_key TEXT NOT NULL  -- 예: "genesis.png"
);

-- 시간별 갱신 frame
CREATE TABLE frames (
  id               BIGSERIAL PRIMARY KEY,
  frame_index      BIGINT UNIQUE NOT NULL,
  generated_at     TIMESTAMPTZ NOT NULL,
  seed_hex         TEXT NOT NULL,        -- (genesis_seed_hex, frame_index)에서 파생된 시드 또는 Genesis 참조
  prng_version     TEXT NOT NULL,
  similarity_pct   DOUBLE PRECISION,     -- exact match 기준
  match_count      INTEGER,              -- exact 매치 픽셀 수 (0~8294400)
  image_object_key TEXT,                 -- NULL = 보존 정책에 따라 PNG 폐기됨
  status           TEXT NOT NULL DEFAULT 'ok'  -- ok / gap / archived
);
CREATE INDEX idx_frames_generated_at ON frames(generated_at);
```

### S3 호환 오브젝트 스토리지 (이진 파일)

| 키 | 내용 | 비고 |
|---|---|---|
| `genesis.png` | Genesis Frame, 무손실 PNG | 약 23.7MB, 영구 보관 |
| `frames/{frame_index}.png` | 시간별 frame | 보존 정책에 따라 일정 기간 후 삭제 |

---

## 보존 및 비용 정책 (Retention & Cost Policy)

- **기본 보존 기간**: 최근 **30일치 PNG**만 원본 보관 (720 frames).
- 30일이 지난 frame은 PNG를 폐기하고 메타데이터(시드 포함)만 유지. 필요 시 시드로 재생성 가능.
- 30일치 스토리지 = 720 × 23.7MB ≈ **17 GB**. seed-only 메타데이터는 무시 가능 수준.
- Genesis Frame과 Genesis seed는 **무기한 보관**.
- 스토리지 알람 임계치: **50 GB** 초과 시 운영자 알림.
- 월 단위 스토리지 비용 보고서 자동 생성.

---

## 운영 (Operations)

### 인증 (Auth)

- **일반 사용자**: 인증 불필요. 모든 캔버스/그래프/API는 공개.
- **관리자**: 단일 매니저 계정 (이메일 magic link 기반).
  - 관리자 권한: 수동 frame 트리거, 보존 정책 override, archive/gap 상태 관리, 시스템 메트릭 조회.

### 모니터링 (Monitoring)

| 메트릭 | 의미 |
|---|---|
| `frame_generation_seconds` | 단일 frame 생성 소요 시간 (목표 < 5s) |
| `similarity_calc_seconds` | 유사도 계산 소요 시간 (목표 < 1s) |
| `storage_bytes_total` | 현재 S3 사용량 |
| `frames_skipped_total` | 정시 갱신 누락 누적 카운트 |

알람 조건:
- 정시(00분 기준) 후 5분 내에 새 frame이 생성되지 않음
- `storage_bytes_total > 50 GB`
- 유사도 계산 실패 (예외 발생)

### 장애 복구 (Failure Recovery)

- **Genesis seed 분실 = 프로젝트 종료.** → 매니저 환경 변수 + 별도 오프라인 백업 위치(예: 별도 클라우드 버킷 또는 비밀 관리 서비스), **반드시 2곳 이상**에 동시 보관.
- **정시 갱신 실패**: 다음 정시 직전까지 1회 자동 재시도. 실패 frame은 `frames` 테이블에 `status = 'gap'` 으로 기록 (시계열 연속성 유지).
- **DB 다운**: PNG 생성과 S3 업로드는 계속 진행. 메타데이터는 로컬 큐에 적재하고 DB 복구 시 일괄 INSERT.
- **S3 다운**: PNG 생성을 메모리에 보관, 5분간 재시도. 실패 시 `gap` 처리.

### 비용 관리 (Cost Management)

- 월 단위 스토리지 비용 리포트.
- `storage_bytes_total > 50 GB` 시 자동 알림 + 보존 기간을 7일로 일시 단축하는 비상 모드 제공.

---

## 프런트엔드 호환성 (Frontend Compatibility)

- **렌더링 default = WebGL.** 4K 텍스처와 zoom/pan 성능을 위해 채택. Canvas2D는 WebGL 미지원 환경에서의 fallback.
- **데스크톱**: 4K 원본 표시.
- **모바일/저사양**: 첫 로드 시 1280×720 다운샘플 미리보기 (약 1MB). zoom 인터랙션 시 영역별 4K 타일을 lazy load.
- **지원 브라우저**: Chrome / Edge / Firefox / Safari — 각각 최신 2개 메이저 버전.

---

## 기능 명세

### 1. 픽셀 캔버스 렌더링
- 4K UHD(3,840×2,160) 캔버스를 사용자 화면에 표시.
- **WebGL을 default로 채택**. Canvas2D는 fallback (§프런트엔드 호환성 참조).
- 사용자 확대/축소(zoom & pan) 인터랙션 지원.

### 2. 매시간 자동 갱신
- **서버 내부 APScheduler가 default**, 외부 cron은 fallback.
- 시드 기반 무작위 색 생성으로 결과 재현 가능성 확보 (§결정성과 시드 참조).
- 갱신 이력은 영구 저장 (§저장소 스키마 참조).

### 3. 유사도 계산
- Genesis Frame과 현재 Frame을 픽셀 단위 비교.
- **기본 지표는 Exact Match**, 선택 지표(Euclidean RGB / CIEDE2000)는 Phase 4 이후 토글로 제공 (§유사도 정의 참조).
- 결과는 `frames` 테이블의 `similarity_pct`, `match_count`, `generated_at`에 기록.

### 4. 변화율 그래프
- X축: 시간 (시 단위).
- Y축: 유사도 백분율.
- 꺾은선 그래프로 누적 추세 표시.
- 이론적 기댓값(약 5.96×10⁻⁸) 보조선과 비교 가능.

### 5. 히스토리 아카이브
- 30일 이내 frame은 PNG 직접 열람.
- 30일 이상 지난 frame은 시드로 on-demand 재생성하여 표시.
- 특정 시점의 픽셀 좌표 단위 검색 기능.

---

## 공개 API (Phase 5 사양 초안)

모든 응답은 JSON, 시각은 ISO 8601 UTC.

| 엔드포인트 | 설명 |
|---|---|
| `GET /api/genesis` | Genesis Frame 메타 (seed_hex, created_at, image URL) |
| `GET /api/frames/latest` | 가장 최근 frame 메타 |
| `GET /api/frames/{index}` | 특정 frame 메타 + PNG signed URL (PNG 폐기 후엔 재생성 트리거) |
| `GET /api/similarity?from=&to=` | 유사도 시계열 (그래프용 데이터) |

- Rate limit: IP당 60 req/min.
- 인증 불필요. 관리자 전용 엔드포인트는 별도 prefix(`/admin/...`)로 분리.

---

## 기술 스택 (예정)

| 영역 | 후보 |
|---|---|
| 백엔드 | Python (FastAPI) |
| 픽셀 생성 | NumPy (`Generator(PCG64)`) / Pillow (PNG 인코딩) |
| 저장소 | PostgreSQL (메타데이터) + S3 호환 스토리지 (이미지) |
| 스케줄러 | APScheduler (default) / cron (fallback) |
| 프런트엔드 | React / WebGL (default) / Canvas2D (fallback) / D3.js (그래프) |
| 모니터링 | Prometheus + Grafana (예정) |
| 배포 | Docker, Cloudflare 또는 AWS |

> 위 스택은 초기 구상이며 프로젝트 진행에 따라 변경될 수 있습니다.

---

## 로드맵

- [ ] **Phase 0** — 프로젝트 셋업, 시드 기반 4K 픽셀 생성기 프로토타입
  - DoD: 동일 시드 → 동일 SHA-256, 단일 frame 생성 < 5초.
- [ ] **Phase 1** — Genesis Frame 생성 및 영구 저장
  - DoD: Genesis가 DB+S3에 함께 기록되고 시드 재생성 검증 통과.
- [ ] **Phase 2** — 매시간 갱신 스케줄러 + 유사도 계산 파이프라인
  - DoD: 24시간 무누락 24 frames, 모든 frame에 `similarity_pct` 채워짐.
- [ ] **Phase 3** — 4K 캔버스 렌더링 웹 UI
  - DoD: 데스크톱 60fps zoom, 모바일 미리보기 < 2초 로드.
- [ ] **Phase 4** — 유사도 변화 그래프 대시보드
  - DoD: 30일치 데이터 1초 내 렌더, 옵션 지표 토글 가능.
- [ ] **Phase 5** — 히스토리 아카이브 및 공개 API
  - DoD: 4개 공개 엔드포인트가 OpenAPI 스펙으로 문서화 + 통합 테스트 통과.

자세한 완료 조건은 §완료 조건(Definition of Done) 참조.

---

## 완료 조건 (Definition of Done)

각 Phase는 아래 조건을 **모두** 만족해야 다음 Phase로 진행합니다.

### Phase 0 — 픽셀 생성기 프로토타입
- 동일한 `(seed_hex, frame_index)`로 두 번 실행 시 출력 PNG의 SHA-256이 비트 단위 일치.
- 단일 4K frame 생성 시간 < **5초** (개발 머신 기준).
- 단위 테스트로 위 두 조건이 자동 검증됨.

### Phase 1 — Genesis Frame 영속화
- Genesis seed가 환경 변수 + 별도 백업 2곳에 저장됨.
- `genesis` 테이블에 1행, S3에 `genesis.png` 객체 존재.
- 메타데이터에서 시드를 읽어 PNG를 재생성한 결과가 S3 객체와 SHA-256 일치.

### Phase 2 — 시간별 갱신 파이프라인
- 24시간 동안 cron이 누락 없이 24개 frame을 생성 (`status = 'ok'` 24행).
- 모든 frame에 `similarity_pct` 와 `match_count` 가 채워짐.
- 24시간 평균 `match_count` 가 **0~2** 범위 (이론적 기댓값 0.5에 부합).
- 의도적 DB/S3 장애 주입 시 적절한 `gap` 처리 또는 복구 동작 확인.

### Phase 3 — 4K 캔버스 UI
- Chrome / Safari / Firefox 데스크톱에서 4K 캔버스 zoom 인터랙션 60fps.
- 모바일(중급 안드로이드 기준) 첫 로드 < **2초**, 다운샘플 미리보기 표시.
- WebGL 미지원 환경에서 Canvas2D fallback 동작.

### Phase 4 — 유사도 그래프
- 최근 30일치 데이터 (720 포인트) 1초 이내 렌더.
- Exact match 라인이 default로 표시되고, Euclidean RGB / ΔE 옵션 지표 토글 가능.
- 이론적 기댓값 보조선 표시.

### Phase 5 — 아카이브 + 공개 API
- 4개 공개 엔드포인트(`/api/genesis`, `/api/frames/latest`, `/api/frames/{index}`, `/api/similarity`)가 OpenAPI 3.x 스펙으로 문서화.
- 각 엔드포인트에 통합 테스트 (200 응답 + 스키마 검증) 통과.
- Rate limit 60 req/min/IP 동작 확인.
- 30일 경과 frame 요청 시 시드 재생성 후 응답 (캐시 포함).

---

## 철학적 메모

> *"무한한 시간 속에서, 우연히도 같은 색을 가진 두 점은 서로를 알아볼 수 있을까?"*

이 사이트의 숫자는 차갑지만, 그 너머에 있는 질문은 따뜻합니다.
서로 다른 80억의 마음이 단 한 순간이라도 같은 색을 띨 수 있다면,
그것이 0.0000001%의 우연이라 할지라도 — 우리는 그것을 **의미**라고 부를 수 있을 것입니다.

---

## 라이선스

추후 결정 (MIT 검토 중).
