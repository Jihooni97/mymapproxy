# MapProxy Windows 폐쇄망 배포 가이드

## 전송할 파일

| 항목 | 경로 | 크기 |
|------|------|------|
| 캐시 타일 | `cache_data/fujairah/` | ~881MB |
| 설정 파일 | `mapproxy.yaml` | 소용량 |

---

## 1. 인터넷 되는 Windows PC에서 사전 준비

### Python 설치
1. [python.org](https://www.python.org/downloads/) 에서 Python 3.11+ 다운로드 및 설치
   - 설치 시 **"Add Python to PATH"** 체크 필수

### MapProxy 및 의존성 오프라인 패키지 수집
인터넷 되는 PC에서 실행:
```cmd
pip download MapProxy -d C:\mapproxy_packages
```
생성된 `C:\mapproxy_packages` 폴더를 USB/이동식 디스크에 복사.

---

## 2. 폐쇄망 VM으로 파일 전송

USB 또는 내부 네트워크로 다음 항목 전송:

```
C:\mapproxy\
├── mapproxy.yaml          ← 이 저장소의 파일
├── cache_data\
│   └── fujairah\          ← 타일 캐시 (881MB)
└── packages\              ← pip download로 수집한 패키지들
```

---

## 3. 폐쇄망 VM에서 설치

PowerShell 또는 CMD에서 실행:

```cmd
cd C:\mapproxy

:: 가상환경 생성
python -m venv venv

:: 가상환경 활성화
venv\Scripts\activate

:: 오프라인 설치
pip install --no-index --find-links=packages MapProxy
```

---

## 4. 서버 실행

```cmd
cd C:\mapproxy
venv\Scripts\activate
mapproxy-util serve-develop mapproxy.yaml --bind 0.0.0.0:8080
```

서버 확인: `http://localhost:8080/demo/`

### 백그라운드 실행 (PowerShell)
```powershell
Start-Process -NoNewWindow -FilePath "C:\mapproxy\venv\Scripts\mapproxy-util.exe" `
  -ArgumentList "serve-develop mapproxy.yaml --bind 0.0.0.0:8080" `
  -WorkingDirectory "C:\mapproxy" `
  -RedirectStandardOutput "C:\mapproxy\mapproxy.log"
```

---

## 5. Windows 시작 시 자동 실행 (선택)

`C:\mapproxy\start_mapproxy.bat` 파일 생성:
```bat
@echo off
cd /d C:\mapproxy
call venv\Scripts\activate
mapproxy-util serve-develop mapproxy.yaml --bind 0.0.0.0:8080
```

이 bat 파일을 Windows 작업 스케줄러에 등록하거나  
`C:\Users\{사용자}\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\` 에 복사.

---

## 6. mapproxy.yaml 경로 수정

Windows에서는 캐시 경로를 절대경로로 변경 권장:

```yaml
globals:
  cache:
    base_dir: C:/mapproxy/cache_data
    lock_dir: C:/mapproxy/cache_data/locks
```

캐시 디렉토리도 동일하게:
```yaml
caches:
  fujairah_cache:
    cache:
      type: file
      directory: C:/mapproxy/cache_data/fujairah
```

---

## 7. HMI 앱 연결

`.env.local` 설정:
```
NEXT_PUBLIC_MAP_TILE_URL=http://localhost:8080/wmts/my_layer/webmercator/{z}/{x}/{y}.png
```

VM IP로 외부 접근 시:
```
NEXT_PUBLIC_MAP_TILE_URL=http://{VM_IP}:8080/wmts/my_layer/webmercator/{z}/{x}/{y}.png
```

---

## 방화벽 포트 오픈 (필요 시)

```cmd
netsh advfirewall firewall add rule name="MapProxy" dir=in action=allow protocol=TCP localport=8080
```
