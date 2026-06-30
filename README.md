# mymapproxy

MapProxy 기반 오프라인 지도 타일 서버.  
Google 위성 타일을 캐싱해 폐쇄망 환경에서 제공한다.

## 구성

| 항목 | 내용 |
|------|------|
| 소스 | Google Satellite |
| 캐시 지역 | 두바이-푸자이라 (54.8~56.6°E, 24.8~25.6°N) |
| 줌 레벨 | z6 ~ z16 |
| 타일 수 | 70,979 |
| 캐시 크기 | ~881MB |

## 실행 (Mac / Linux)

```bash
# 의존성 설치
python3 -m venv .venv
.venv/bin/pip install MapProxy

# 서버 실행 (포트 8080)
.venv/bin/mapproxy-util serve-develop mapproxy.yaml --bind 0.0.0.0:8080
```

## 실행 (Windows 폐쇄망)

`WINDOWS_DEPLOY.md` 참고

## HMI 연결 URL

```
http://localhost:8080/wmts/google/webmercator/{z}/{x}/{y}.png
```

VM에서 접근 시 `localhost` 를 VM IP로 변경.

## 추가 지역 캐싱

`seed.yaml` 의 `bbox` 수정 후:

```bash
.venv/bin/mapproxy-seed -f mapproxy.yaml -s seed.yaml --seed dubai_fujairah_seed -c 4
```
