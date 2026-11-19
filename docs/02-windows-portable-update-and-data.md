# Windows 포터블 앱 업데이트 및 사용자 데이터 보존 기준

## 1. Easy Save Audio & Video에서 확인한 실제 방식

조사 저장소: https://github.com/ryoobach/Easy_Save_Audio

확인 파일:
- .github/workflows/build-windows.yml
- app_v2.py
- README.md

Easy Save는 Android APK처럼 signing certificate를 비교해 업데이트하는 앱이 아닙니다. Windows x64용 PyInstaller onedir 포터블 패키지 ZIP을 빌드하고 배포합니다.

### 1.1 빌드 패키지 방식

Workflow에서:
- PyInstaller onedir 방식으로 빌드합니다.
- 실행 파일과 _internal 런타임 폴더를 하나의 앱 디렉터리에 둡니다.
- 실행 파일과 _internal 파일이 ZIP에 모두 들어 있는지 검증합니다.
- 버전이 포함된 ZIP 파일명으로 artifact를 생성합니다.

확인된 예시 버전은 workflow에서 2.1.5로 검증하며, 패키지 이름은 다음 구조입니다.

Easy_Save_Audio_Video_v2.1.5_Windows_x64_Onedir.zip

### 1.2 사용자 데이터 분리

app_v2.py는 사용자 데이터 경로를 다음과 같이 분리합니다.

%APPDATA%\EasySaveAudio

이 경로 아래에 settings.json, 작업 기록, 임시 작업 데이터를 보관합니다.

설정 파일에는 마지막 MP3 저장 폴더, 마지막 동영상 저장 폴더, 작업 이력이 저장됩니다. 1.x 설정/기록을 2.x에서 읽어 이전 MP3 폴더와 기록을 유지하는 마이그레이션 코드도 확인했습니다.

따라서 프로그램 실행 폴더를 새 onedir 패키지로 교체하더라도 사용자 설정과 작업 이력은 별도 경로에 남도록 설계되어 있습니다.

## 2. Easy Save 방식의 업데이트 의미

이 방식은 설치 프로그램의 자동 업데이트 기능이 아닙니다. 현재 확인된 workflow는 버전 ZIP을 빌드하고 배포하는 역할입니다.

실제 업데이트 시에는:
1. 앱을 완전히 종료합니다.
2. 새 버전 ZIP을 별도 임시 폴더에 압축 해제합니다.
3. 기존 프로그램 폴더의 사용자 데이터가 아닌 앱 실행 파일 및 런타임 파일을 새 버전으로 교체합니다.
4. %APPDATA%\EasySaveAudio는 삭제하거나 덮어쓰지 않습니다.
5. 새 버전을 실행해 설정과 작업 기록이 유지됐는지 확인합니다.

앱 폴더 전체를 교체하는 경우에도 사용자 데이터가 앱 폴더 안에 저장되어 있지 않은지 확인해야 합니다.

## 3. 다른 Windows 앱에 적용할 기준

- 사용자 설정·이력·다운로드 폴더를 실행 파일 디렉터리 밖에 둡니다.
- 업데이트 ZIP에는 실행 파일과 필요한 런타임 파일을 모두 포함합니다.
- 패키지 안에 launcher와 runtime 폴더가 있는지 자동 검증합니다.
- 업데이트는 앱 프로세스를 종료한 뒤 수행합니다.
- 기존 사용자 데이터 폴더는 보존하고, 설정 형식이 바뀌면 명시적 마이그레이션을 작성합니다.
- 버전 문자열을 단일 소스에서 읽고 artifact 파일명과 workflow 검증값을 일치시킵니다.
- 업데이트 실패 시 기존 앱 폴더를 복구할 수 있도록 교체 전에 임시 백업 또는 staging 디렉터리를 사용합니다.

## 4. Android 방식과 혼동하지 말 것

Easy Save의 폴더 교체 방식은 Android APK의 서명 문제를 해결하지 않습니다.

- Android: applicationId + 동일 서명 인증서 + 증가한 versionCode가 필수입니다.
- Windows portable: 앱 파일 교체와 사용자 데이터 경로 분리가 핵심입니다.

## 참고 자료

- https://github.com/ryoobach/Easy_Save_Audio/blob/main/.github/workflows/build-windows.yml
- https://github.com/ryoobach/Easy_Save_Audio/blob/main/app_v2.py
