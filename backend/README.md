# Ninano ENSO API

## 실행

저장소 루트가 아니라 `backend` 디렉터리에서 실행합니다.

```bash
cd backend
uv sync
uv run uvicorn server.app:app --reload --host 127.0.0.1 --port 8000
```

- API 문서: <http://127.0.0.1:8000/docs>
- 상태 확인: <http://127.0.0.1:8000/api/v1/health>
- metadata: <http://127.0.0.1:8000/api/v1/enso/metadata>

기본 데이터 위치는 `../public_data/f32_packed_05deg`(0.5° 다운샘플)입니다. 0.25° 원본을 쓰려면 아래처럼 `NINANO_F32_ROOT`를 지정합니다. 별도 디스크나 서버의 데이터를 사용할 때는 다음처럼 지정합니다.

```bash
NINANO_F32_ROOT=/data/f32_packed \
  uv run uvicorn server.app:app --host 0.0.0.0 --port 8000
```

레이어 요청 예시:

```text
GET /api/v1/enso/layers/sst?sst_anomaly=1.2&wind_delta=3
GET /api/v1/enso/layers/wind?sst_anomaly=1.2&wind_delta=3
GET /api/v1/enso/layers/t2m?sst_anomaly=1.2&wind_delta=3
```

응답은 JSON이 아니라 원본 little-endian Float32 binary입니다. 현재 `wind` 파일은 격자점마다 `[u, v]`가 번갈아 저장된 interleaved 형식이고, `layout=planar`로 요청하면 서버가 `[u10m 전체][v10m 전체]` 순서로 바꿔 보냅니다.

```text
GET /api/v1/enso/layers/wind?sst_anomaly=1.2&wind_delta=3&layout=planar
```

원본이 이미 planar이면 wind 파일도 변환 없이 `FileResponse`로 그대로 전송합니다.

## 테스트

```bash
uv run pytest -q
```
