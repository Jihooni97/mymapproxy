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

### Windows 폐쇄망

`WINDOWS_DEPLOY.md` 참고

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
