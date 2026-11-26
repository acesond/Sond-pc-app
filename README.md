# 🖥️ PC 원격제어 · 버튼식 웹앱 (미리보기)

폰에서 **버튼 한 번**으로 집 Windows PC(`acesond/Sond-pickle`)를 조작하는 단일 파일 웹앱입니다.
빌드 없이 `index.html` 하나로 동작하고, 코인 웹앱과 같은 다크 테마를 씁니다.

- **배포 주소: <https://acesond.github.io/Sond-pc-app/>** (GitHub Pages, `main` / `/`)
- 폰 브라우저에서 위 주소를 열고 **홈 화면에 추가**하면 앱처럼 뜹니다. 토큰은 사이트별 localStorage라 배포 주소에서 한 번만 다시 넣으면 됩니다.
- 미리보기(로컬 검증): `preview-390.png` (390×844), `preview-390-buttons.png`, `preview-390-capture.png`, `preview-390-full.png` / 라이브: `live-390.png`

## 쓰는 순서

1. GitHub에서 **Fine-grained token** 발급 → <https://github.com/settings/personal-access-tokens/new>
   - Repository access: **Only select repositories** → `acesond/Sond-pickle`
   - Permissions → Repository permissions → **Actions: Read and write**
2. 앱의 `🔑 GitHub 토큰` 칸에 붙여넣고 **저장** → **연결 확인**
   - 배지가 `연결됨`이 되고 워크플로 `PC Remote Commander · active`가 표시되면 준비 완료
3. 아래 버튼을 누르면 `pc-commander.yml`이 `workflow_dispatch`로 실행되고, 앱이 실행 상태를 자동으로 따라갑니다.

토큰은 **브라우저 localStorage에만** 저장되며 코드에는 들어 있지 않습니다. 요청은 `api.github.com`에만 나갑니다.
(토큰을 지우면 `삭제` 버튼, 저장 위치는 `localStorage` 키 `sondpc.pat.v1`)

## 빠른 제어 버튼과 실제 명령

워크플로는 `shell: cmd`입니다. 모든 경로는 실제 PC 값(`%LOCALAPPDATA%\Programs\Python\Python312\python.exe`,
`%USERPROFILE%\pc-tools\...`)을 그대로 씁니다.

| 버튼 | 보내는 명령 |
|---|---|
| 📸 현재 화면 캡처 | `"%LOCALAPPDATA%\Programs\Python\Python312\python.exe" "%USERPROFILE%\pc-tools\capture.py"` |
| 🪟 열린 창 목록 | `powershell -NoProfile -Command "Get-Process \| Where-Object {$_.MainWindowTitle} \| Select-Object ProcessName,MainWindowTitle \| Format-Table -AutoSize"` |
| 🔌 러너 상태 | `tasklist /FI "IMAGENAME eq Runner.Listener.exe" & tasklist /FI "IMAGENAME eq Runner.Worker.exe"` |
| 🌐 크롬 열기 | `start "" chrome` |
| 📂 탐색기 열기 | `start "" explorer.exe "%USERPROFILE%"` |
| 💬 카카오톡 종료 *(확인 팝업)* | `taskkill /IM KakaoTalk.exe /F` — 자동 로그인이 꺼져 있어 다시 켜면 로그인 화면 |
| 🧹 PowerShell·터미널 닫기 *(확인 팝업)* | `taskkill /IM powershell.exe /F & taskkill /IM Terminal.exe /F & taskkill /IM WindowsTerminal.exe /F` (러너·cmd는 건드리지 않음) |
| 🔒 PC 화면 잠금 *(확인 팝업)* | `rundll32.exe user32.dll,LockWorkStation` |

`⌨️ 직접 명령` 칸에 cmd 명령을 적고 **실행**하면 그대로 전송됩니다. `Ctrl/Cmd + Enter`로도 실행됩니다.

## 실행 상태 추적

- `POST /repos/acesond/Sond-pickle/actions/workflows/pc-commander.yml/dispatches` 로 전송(`ref: main`, `inputs.command`)
- 보내기 전의 최신 런 ID를 기억해 두고 `GET /actions/runs?event=workflow_dispatch` 를 3초 간격으로 폴링
  → `큐 대기 → 실행 중 → 완료(성공/실패)` 배지, 소요 시간, `GitHub에서 보기` 링크 표시
- 같은 앱에서 동시에 두 명령이 나가지 않도록 막습니다(러너 1대·PC 화면 1개).
- 20초 넘게 `큐 대기`면 "PC가 꺼졌거나 러너가 멈췄을 수 있습니다" 안내를 띄웁니다.

## 화면 캡처가 보이는 원리와 한계

`pc-commander.yml`은 **아티팩트를 올리지 않습니다.** `capture.py`가 화면을 1280px JPEG로 줄여
`===START_SCREEN_CHUNK=== … ===END_SCREEN_CHUNK===` 사이에 **base64로 로그에 출력**하는 구조입니다.
그래서 앱은:

1. 캡처 런이 성공하면 `GET /actions/runs/{id}/logs`(로그 zip)를 받아 **JSZip**으로 열고, 로그 안의 청크를 찾아
   `data:image/jpeg;base64,…`로 바로 화면에 띄웁니다. (`INFO:1920x1080`도 함께 읽어 배율 계산)
2. 이 요청이 브라우저 보안(CORS)에서 막히면(로그 zip은 저장소 리다이렉트라 헤더가 없을 수 있음)
   **로그 붙여넣기** 칸에 `===START_SCREEN_CHUNK===` 구간을 붙여 넣어 같은 방식으로 이미지를 만듭니다.
   실패 시 `GitHub에서 보기` 링크로 실행 페이지를 엽니다.

캡처 이미지를 누르면 그 지점이 PC 좌표로 환산됩니다(`이미지 x × 화면폭/1280`, 기본 1.5배 — `화면 폭` 칸에서 조정).
`🖱 클릭` → 확인 팝업 → `click.py <x> <y> 1` 전송.

## 파일

- `index.html` — 앱 전체(스타일·스크립트 포함, 외부 의존은 JSZip CDN 하나)
- `README.md` — 이 문서

로컬 개발 폴더(`downloads/pc-remote-webapp/`) 기준 검증 자료:

- `preview-390*.png`, `live-390.png` — 390×844 검증 스크린샷(로컬 / 배포 라이브)
- `../../tests/pc_remote_webapp_test.py` — Playwright 검증(GitHub API 모킹), 결과 `../../tests/report.json`
- `../../tests/live_pc_remote_check.py` — 배포 주소 390×844 라이브 검증, 결과 `../../tests/live_report.json`
- `../../tests/fixtures/real_capture_log.txt` — 실제 러너 로그(2026-11-24 캡처 런) 파싱 검증용

## 배포

- 저장소: `acesond/Sond-pc-app` (Public)
- GitHub Pages: Settings → Pages → Source `Deploy from a branch` → Branch `main` / `/ (root)` → Save
- 배포 주소: <https://acesond.github.io/Sond-pc-app/>
- 수정은 `index.html`을 같은 `main` 브랜치에 푸시하면 1분 안에 자동 반영됩니다.
