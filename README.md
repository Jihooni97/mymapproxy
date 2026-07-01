# mymapproxy

MapProxy 기반 오프라인 지도 타일 서버.  
타일을 미리 캐싱해 폐쇄망 환경에서 인터넷 없이 지도를 제공한다.

---

## 구성 

| 항목 | 내용 |
|------|------|
| 캐시 지역 | 두바이-푸자이라 (54.8~56.6°E, 24.8~25.6°N) |
| 줌 레벨 | z6 ~ z16 |
| 타일 수 | 70,979 |
| 캐시 크기 | ~881MB |

---

## 동작 방식

```
클라이언트 → MapProxy → 캐시 확인
                          ├── 캐시 있음 → 타일 반환
                          └── 캐시 없음 → 투명 타일 반환 (외부 요청 없음)
```

캐시된 구역은 위성 이미지 표시, 미캐싱 구역은 빈 타일. 완전 오프라인 동작.

---

## 설치 및 실행

### Mac / Linux

```bash
# 1. 가상환경 생성 및 설치
python3 -m venv .venv
.venv/bin/pip install MapProxy

# 2. 서버 실행 (포트 8080)
.venv/bin/mapproxy-util serve-develop mapproxy.yaml --bind 0.0.0.0:8080

# 3. 백그라운드 실행
.venv/bin/mapproxy-util serve-develop mapproxy.yaml --bind 0.0.0.0:8080 > /tmp/mapproxy.log 2>&1 &
```

### Windows

```cmd
:: 1. 가상환경 생성 및 설치
python -m venv venv
venv\Scripts\activate
pip install MapProxy

:: 2. 서버 실행 (포트 8080)
mapproxy-util serve-develop mapproxy.yaml --bind 0.0.0.0:8080
```

Python 설치 시 **"Add Python to PATH"** 체크 필수.

백그라운드 실행 (PowerShell):
```powershell
Start-Process -NoNewWindow -FilePath "venv\Scripts\mapproxy-util.exe" `
  -ArgumentList "serve-develop mapproxy.yaml --bind 0.0.0.0:8080" `
  -WorkingDirectory (Get-Location) `
  -RedirectStandardOutput "mapproxy.log"
```

> 폐쇄망(인터넷 없는) 환경 배포는 `WINDOWS_DEPLOY.md` 참고

---

## 엔드포인트

| 용도 | URL |
|------|-----|
| 데모 페이지 | `http://localhost:8080/demo/` |
| 타일 (HMI 연결용) | `http://localhost:8080/wmts/my_layer/webmercator/{z}/{x}/{y}.png` |
| WMS | `http://localhost:8080/service?SERVICE=WMS&...` |

VM에서 외부 접근 시 `localhost` → VM IP 변경.

---

## HMI 연결 설정

`frontend/.env.local`:
```
NEXT_PUBLIC_MAP_TILE_URL=http://localhost:8080/wmts/my_layer/webmercator/{z}/{x}/{y}.png
```

---

## 추가 지역 캐싱

### 1. seed.yaml 수정

```yaml
coverages:
  my_area:
    bbox: [경도_min, 위도_min, 경도_max, 위도_max]  # EPSG:4326
    srs: 'EPSG:4326'
```

### 2. 타일 수 사전 계산 (선택)

```python
import math

def tile_count(lon_min, lon_max, lat_min, lat_max, z_min, z_max):
    total = 0
    for z in range(z_min, z_max + 1):
        x0 = int((lon_min + 180) / 360 * 2**z)
        x1 = int((lon_max + 180) / 360 * 2**z)
        y0 = int((1 - math.log(math.tan(math.radians(lat_max)) + 1/math.cos(math.radians(lat_max)))/math.pi)/2 * 2**z)
        y1 = int((1 - math.log(math.tan(math.radians(lat_min)) + 1/math.cos(math.radians(lat_min)))/math.pi)/2 * 2**z)
        total += (x1-x0+1) * (y1-y0+1)
    return total
```

### 3. 시딩 실행

```bash
# -c: 동시 요청 수 (기본 2, 최대 4 권장)
.venv/bin/mapproxy-seed -f mapproxy.yaml -s seed.yaml --seed uae_seed -c 4
```

### 4. 진행 상황 확인

```bash
tail -f /tmp/mapproxy-seed.log
```

---

## 줌 레벨 기준

| 줌 레벨 | 표시 단위 |
|---------|---------|
| z6 ~ z8 | 국가 수준 |
| z10 ~ z12 | 도시 수준 |
| z14 ~ z15 | 구역/도로 수준 |
| z16 | 건물 윤곽 수준 |
| z17+ | 건물 상세 (타일 수 급증) |

> z17은 z16 대비 4배 타일 수. 필요한 경우에만 포함 권장.

---

## 설정 파일 가이드

### mapproxy.yaml

MapProxy 서버의 핵심 설정. 서비스 노출 방식, 레이어, 캐시 구조, 타일 소스를 정의한다.

#### `services` — 외부에 노출할 프로토콜

```yaml
services:
  demo:       # 브라우저 확인용 데모 페이지 (/demo/)
  wms:        # WMS 프로토콜 (GIS 클라이언트용)
    md:
      title: MyMapProxy
      abstract: 서버 설명
  wmts:       # WMTS 프로토콜 (웹 지도 클라이언트용, 권장)
  tms:        # TMS 프로토콜 (슬래시리 등 구형 클라이언트용)
```

필요 없는 프로토콜은 삭제해도 무방.

#### `layers` — 클라이언트에 노출되는 레이어

```yaml
layers:
  - name: my_layer        # URL에 사용되는 레이어 이름
    title: My Satellite   # 사람이 읽는 표시 이름
    sources: [fujairah_cache]  # 이 레이어가 읽는 캐시 이름
```

레이어를 여러 개 정의하면 여러 구역/스타일을 독립 제공 가능.

#### `caches` — 타일 저장 방식

```yaml
caches:
  fujairah_cache:           # 캐시 이름 (layers에서 참조)
    grids: [webmercator]    # 사용할 그리드 (좌표계 + 줌 레벨 정의)
    sources: [my_source]    # 타일을 가져올 소스 이름
    cache:
      type: file            # 파일 시스템에 저장 (기본값)
      directory: ./cache_data/fujairah  # 저장 경로
```

`type`은 `file` 외에 `mbtiles`, `s3`, `redis` 등 지원.

#### `sources` — 타일을 가져오는 원본 서버

```yaml
sources:
  my_source:
    type: tile              # 타일 URL 방식
    url: http://127.0.0.1:1/%(tms_path)s.png  # 타일 서버 URL
    grid: webmercator
    on_error:
      other:
        response: transparent  # 오류 시 투명 타일 반환
        cache: false           # 오류 타일은 캐싱 안 함
```

`url` 패턴 변수:
- `%(tms_path)s` → `z/x/y` 형태
- `%(x)s`, `%(y)s`, `%(z)s` → 개별 좌표

`type`은 `tile` 외에 `wms`, `mapserver` 등 지원.

#### `grids` — 좌표계 및 줌 레벨 정의

```yaml
grids:
  webmercator:
    base: GLOBAL_WEBMERCATOR  # 표준 웹 메르카토르 (EPSG:3857)
    num_levels: 23            # 줌 레벨 0~22 지원
```

내장 기본값: `GLOBAL_WEBMERCATOR` (웹 지도 표준), `GLOBAL_GEODETIC` (WGS84).

#### `globals` — 전역 설정

```yaml
globals:
  cache:
    base_dir: ./cache_data      # 캐시 루트 디렉토리
    lock_dir: ./cache_data/locks  # 동시 요청 락 파일 위치
  http:
    client_timeout: 60          # 소스 서버 요청 타임아웃 (초)
```

---

### seed.yaml

타일을 미리 받아 캐시에 저장하는 시딩(seeding) 작업을 정의한다.  
`mapproxy-seed` 명령으로 실행하며, 오프라인 배포 전 사전 캐싱에 사용.

#### `seeds` — 시딩 작업 정의

```yaml
seeds:
  fujairah_seed:                # 작업 이름 (--seed 옵션에 전달)
    caches: [fujairah_cache]    # mapproxy.yaml의 캐시 이름
    grids: [webmercator]        # 사용할 그리드
    levels:
      from: 21                  # 시작 줌 레벨
      to: 21                    # 종료 줌 레벨 (범위 지정 가능)
    coverages: [fujairah]       # 시딩 영역 이름
```

`levels` 대신 리스트 형태도 가능:
```yaml
levels:
  list: [6, 7, 8, 14, 15, 16]  # 특정 줌 레벨만 선택
```

#### `coverages` — 시딩 영역 정의

```yaml
coverages:
  fujairah:
    bbox: [56.3197, 25.1931, 56.3554, 25.2163]  # [경도_min, 위도_min, 경도_max, 위도_max]
    srs: 'EPSG:4326'  # bbox 좌표계 (위경도면 EPSG:4326)
```

bbox 대신 GeoJSON/Shapefile 파일 경로 사용도 가능:
```yaml
coverages:
  my_area:
    datasource: ./my_area.geojson
    srs: 'EPSG:4326'
```

#### 시딩 실행

```bash
# 특정 시드 작업 실행
.venv/bin/mapproxy-seed -f mapproxy.yaml -s seed.yaml --seed fujairah_seed -c 4

# 전체 시드 작업 실행
.venv/bin/mapproxy-seed -f mapproxy.yaml -s seed.yaml -c 4
```

`-c`: 동시 다운로드 스레드 수. 서버 부하 고려해 2~4 권장.

#### 시딩 진행 확인

```bash
# 실시간 로그 확인
tail -f /tmp/mapproxy-seed.log

# 캐시 디렉토리 파일 수 확인
find ./cache_data -name "*.png" | wc -l
```
