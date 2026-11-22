# 기술 문서

데이터 형식, 로딩·캐시 동작, 렌더링, 배포에 관한 세부 내용입니다. 프로젝트 소개는 [README](../README.md)를 보세요.

## 현재 데이터 로딩 방식

프런트엔드는 먼저 FastAPI의 변수별 레이어 API를 호출합니다. 개발 중 Vite는 `/api` 요청을 `http://127.0.0.1:8000`으로 프록시합니다.

```text
/api/v1/enso/metadata
/api/v1/enso/layers/{layer}?sst_anomaly=1.2&wind_delta=3
```

FastAPI가 실행되지 않을 때는 `vite.config.ts`의 개발 서버 middleware가 저장소의 `public_data`를 다음 URL로 제공합니다.

```text
/public_data/f32_packed_05deg/manifest.json
/public_data/f32_packed_05deg/sst/{file}.f32
/public_data/f32_packed_05deg/t2m/{file}.f32
/public_data/f32_packed_05deg/tpf/{file}.f32
/public_data/f32_packed_05deg/msl/{file}.f32
/public_data/f32_packed_05deg/tcwv/{file}.f32
/public_data/f32_packed_05deg/wind/{file}.f32
```

데이터 로딩 우선순위는 다음과 같습니다.

```text
FastAPI 변수별 packed F32
        │ 실패
        ▼
Vite 정적 packed F32
        │ 실패
        ▼
브라우저 preview 데이터
```

preview는 UI 확인을 위한 가상 데이터이며 실제 cBottle 결과가 아닙니다.

## Packed F32 데이터 계약

### 파일명

모든 변수 폴더가 같은 파일명을 사용합니다.

```text
sst_{sst_anomaly}_wind_{wind_delta}.f32
```

예시는 다음과 같습니다.

```text
sst_0.0_wind_0.f32
sst_2.4_wind_5.f32
sst_-2.0_wind_-5.f32
```

SST 23단계와 wind 11단계 조합으로 변수 폴더마다 총 253개 조건을 가집니다.

### 격자

격자 크기와 순서는 `manifest.json`을 기준으로 하므로 프런트 코드에 `LATSIZE`, `LONSIZE`를 고정하지 않습니다.

| 항목 | 0.25° 원본 (`f32_packed`) | 0.5° 웹용 (`f32_packed_05deg`) |
| --- | --- | --- |
| 위도 | `90.0 → -90.0`, 721개 | `90.0 → -90.0`, 361개 |
| 경도 | `0.0 → 359.75`, 1,440개 | `0.0 → 359.5`, 720개 |
| dtype | little-endian `float32` | little-endian `float32` |
| scalar 파일 크기 | 4,152,960 bytes | 1,039,680 bytes |

0.5° 격자점은 0.25° 격자점의 부분집합이므로 `backend/tools/downsample_to_05deg.py`는 보간 없이 stride-2로 골라냅니다. 육지의 `NaN` 구조도 그대로 유지됩니다.

### Scalar 파일

`sst`, `t2m`, `tpf`, `msl`, `tcwv`는 각각 하나의 2차원 필드입니다.

```text
layout: [lat, lon]
index = latIndex * lonSize + lonIndex
```

### Wind 파일

wind 파일 하나에는 `u10m`, `v10m` 두 성분이 들어 있습니다. 저장 순서는 `manifest.json`의 `wind.layout`을 따릅니다.

현재 데이터는 격자점마다 u, v가 번갈아 저장된 interleaved 형식입니다.

```text
layout: [lat, lon, component]
index = (latIndex * lonSize + lonIndex) * 2
u10m  = values[index]
v10m  = values[index + 1]
```

`backend/tools/convert_wind_to_planar.py`로 변환하면 `u10m` 전체 격자 뒤에 `v10m` 전체 격자가 오는 planar 형식이 됩니다.

```text
layout: [component, lat, lon]
index = latIndex * lonSize + lonIndex
u10m  = values[index]
v10m  = values[fieldSize + index]
```

프런트엔드는 두 형식을 모두 읽습니다. FastAPI는 `layout=planar` 요청을 받으면 interleaved 파일을 planar로 바꿔 보내고, 최근 4개를 메모리에 캐시합니다.

### 원시값 변환

F32에는 원시 모델 단위가 들어 있으므로 브라우저에서 표시 단위로 변환합니다.

| 변수 | 입력 | 화면 변환 |
| --- | --- | --- |
| `sst` | K | `K - 273.15` → °C |
| `t2m` | K | `K - 273.15` → °C |
| `tpf` | kg m⁻² s⁻¹ 상당 flux | `max(0, value × 86400)` → mm/day |
| `msl` | Pa | `Pa / 100` → hPa |
| `tcwv` | kg/m² | 음수만 0으로 제한 |
| `u10m`, `v10m` | m/s | 변환 없음 |

`NaN`은 데이터 없음으로 취급하며 지도에서 투명하게 표시합니다.

## 요청과 캐시 동작

- 최초 로딩 시 `(SST 0.0, wind 0)`의 `sst`와 기본 표시 레이어를 먼저 읽고 wind를 이어서 읽습니다.
- 표시 레이어를 바꾸면 해당 scalar 폴더의 파일만 추가로 읽습니다.
- 실험 슬라이더를 움직이면 해당 조건의 요청을 즉시 시작합니다.
- 슬라이더를 연속해서 움직이면 이전 요청은 `AbortController`로 취소합니다.
- 조건 변경 시 선택 scalar와 SST가 도착하면 WebGL 지도를 즉시 바꾸고, wind가 도착하면 GPU 입자를 시작합니다.
- 파싱한 필드는 최대 96 MiB 범위에서 LRU 방식으로 메모리에 캐시합니다.

## 화면 렌더링

표시 가능한 색상 레이어는 다음 5개입니다.

| 변수 | 의미 | 표시 단위 |
| --- | --- | --- |
| `sst` | 해수면 온도 | °C |
| `t2m` | 지표 2m 공기 온도 | °C |
| `tpf` | 강수량 | mm/day |
| `msl` | 평균 해면 기압 | hPa |
| `tcwv` | 대기 기둥 수증기량 | kg/m² |

바람 입자는 각 위치의 벡터를 다음과 같이 사용합니다.

```text
speed = sqrt(u10m² + v10m²)
longitude += u10m × timeStep / cos(latitude)
latitude  += v10m × timeStep
```

이 계산은 흐름을 직관적으로 보여주기 위한 화면 애니메이션이며, 물리 시간 적분 결과로 사용하지 않습니다.

평면 지도는 33%에서 `360° × 180°` 전체 범위를 표시하고 모든 확대 단계에서 경도:위도 범위를 2:1로 유지합니다.

## 배포 시 주의사항

`npm run build`는 `apps/web/dist`에 프런트엔드만 생성합니다. 대용량 `public_data`는 dist에 복사하지 않습니다.

운영 환경에서는 Nginx, CDN, Object Storage 또는 백엔드가 `f32_packed_05deg` 디렉터리를 정적 파일로 제공해야 합니다. 데이터 주소가 프런트와 다르면 빌드 전에 설정합니다.

```bash
export VITE_F32_BASE_URL=https://data.example.com/f32_packed_05deg
npm run build
```

해당 주소 바로 아래에는 `manifest.json`, `sst/`, `t2m/`, `tpf/`, `msl/`, `tcwv/`, `wind/`가 있어야 합니다. 다른 origin을 사용하면 데이터 서버의 CORS 설정도 필요합니다.

FastAPI 통합 frame을 fallback으로 사용할 경우 다음 주소를 설정합니다.

```bash
export VITE_API_BASE_URL=https://api.example.com/api/v1
```

## Python 환경

Python/Earth2Studio 작업은 Windows Python이 아니라 Ubuntu/WSL 환경을 기준으로 합니다. 프로젝트는 Python 3.13을 사용합니다.

```bash
wsl -d Ubuntu-24.04
cd /mnt/d/ninano/ninano
uv sync
```

프런트엔드와 로컬 packed F32 확인에는 NVIDIA GPU가 필요하지 않습니다. 실제 Earth2Studio/cBottle 모델 실행에 필요한 장치는 사용하는 모델과 실행 설정에 따라 별도로 확인해야 합니다.

## 자동 생성 파일

- `apps/web/node_modules/`: `npm install`로 설치한 패키지
- `apps/web/dist/`: `npm run build` 결과
- `apps/web/tsconfig.app.tsbuildinfo`: TypeScript 증분 빌드 캐시
- `apps/web/package-lock.json`: 재현 가능한 npm 설치를 위한 잠금 파일

`node_modules`, `dist`, `tsbuildinfo`는 직접 수정하지 않습니다.
