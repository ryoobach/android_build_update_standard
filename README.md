# Android / App Update Standard

이 저장소는 Ryoo jaesang & 최피디가 관리하는 앱의 빌드·서명·업데이트 방식을 기록하고 재사용하기 위한 기준 저장소입니다.

## 확인한 기존 프로젝트

1. **Easy Health (Android)** — 고정된 Android debug signing key를 GitHub Actions cache에서 복원하고, 인증서 지문을 검증한 뒤 APK를 빌드합니다. 동일 서명을 유지해 기존 앱을 삭제하지 않고 업데이트하는 방식입니다.
2. **Easy Save Audio & Video (Windows)** — APK 서명 방식이 아니라 PyInstaller onedir ZIP 배포 방식입니다. 실행 파일과 런타임 폴더를 교체하고, 사용자 설정·작업 기록은 앱 폴더 밖의 %APPDATA%\EasySaveAudio에 보관합니다.

## 문서

- [Android 서명 및 무삭제 업데이트 기준](docs/01-android-signing-and-in-place-update.md)
- [Windows 포터블 앱 업데이트 및 사용자 데이터 보존 기준](docs/02-windows-portable-update-and-data.md)

## 핵심 원칙

- Android는 applicationId와 서명 인증서가 기존 설치본과 같고 versionCode가 더 높아야 무삭제 업데이트가 가능합니다.
- 서명 키를 잃거나 임의로 새 키를 만들면 기존 앱에 업데이트할 수 없습니다.
- Windows 포터블 앱은 실행 파일만 덮어쓰는 것으로 끝내지 않고, 런타임 파일과 사용자 데이터 저장 위치를 분리해 보존합니다.
- 앱별 실제 설정을 확인한 뒤 적용합니다. 문서의 예시 버전·지문·경로를 다른 앱에 그대로 복사하지 않습니다.
- 비밀번호, 개인 키 원문, keystore 파일, GitHub secret 값은 이 저장소에 올리지 않습니다.

## 조사 대상 저장소

- https://github.com/ryoobach/Easy-Health
- https://github.com/ryoobach/Easy_Save_Audio
