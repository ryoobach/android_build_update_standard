# Android 서명 및 무삭제 업데이트 기준

## 1. 기존 Easy Health에서 확인한 실제 방식

조사 저장소: https://github.com/ryoobach/Easy-Health

확인 파일:
- .github/workflows/android.yml
- app/build.gradle.kts

### 1.1 서명 키를 고정하는 방법

GitHub Actions workflow는 actions/cache@v4를 사용합니다.

- cache path: ~/.android/debug.keystore
- 고정 cache key: easy-health-debug-signing-key-v1
- 캐시가 정확히 복원되지 않으면 빌드를 중단합니다.
- cache miss 상태에서 새 debug keystore를 자동 생성해 배포하지 않습니다.
- keytool로 keystore 인증서 SHA-256을 출력합니다.

### 1.2 Gradle에서 같은 키 사용

Easy Health의 app/build.gradle.kts는 debug signing config를 명시적으로 고정합니다.

- 파일: ~/.android/debug.keystore
- storePassword: android
- keyAlias: androiddebugkey
- keyPassword: android
- debug build type이 해당 signing config를 사용하도록 지정합니다.

### 1.3 인증서와 앱 버전 검증

Workflow는 applicationId, versionCode, versionName, keystore 인증서 SHA-256, 최종 APK 인증서 SHA-256을 확인합니다.

확인된 workflow에 기록된 기대 SHA-256:
e71f6e2cba41d3caef823be573636aaa1065e84022bcd7031cb4531b476b0354

APK의 서명 지문과 keystore의 지문이 기대 지문과 모두 일치해야 통과합니다. 이 값은 Easy Health 전용이며 다른 앱에 복사하면 안 됩니다.

### 1.4 조사 중 발견한 버전 설정 불일치

조사 시점의 파일 내용:
- workflow의 검증 기대값: versionCode 36, versionName 1.36
- 현재 app/build.gradle.kts: versionCode 37, versionName 1.37

따라서 두 값은 현재 서로 일치하지 않습니다. Easy Health workflow를 수정하거나 재사용할 때는 Gradle 실제 버전과 workflow 검증값을 동기화해야 합니다.

## 2. Android 앱 무삭제 업데이트의 필수 조건

기존 설치본을 삭제하지 않고 APK를 업데이트하려면 모두 만족해야 합니다.

1. applicationId가 기존 설치 앱과 동일해야 합니다.
2. 새 APK가 기존 설치본과 동일한 서명 인증서로 서명되어야 합니다.
3. 새 versionCode가 설치된 앱보다 커야 합니다.
4. APK가 정상 빌드되고 Android 설치 검증을 통과해야 합니다.

versionName은 사용자에게 보이는 버전 문자열이며, 업데이트 가능 여부를 결정하는 핵심 숫자는 versionCode입니다.

## 3. 새 Android 프로젝트에 적용할 절차

1. 기존 설치 APK 또는 기존 빌드 산출물에서 서명 인증서 지문을 먼저 확인합니다.
2. 과거 GitHub Actions cache, 보관 keystore, GitHub secrets, 안전한 백업 등에서 기존 개인 키를 복구할 수 있는지 확인합니다.
3. 기존 설치본과 동일한 키가 확보된 경우에만 고정 서명 설정을 적용합니다.
4. 기존 키를 복원하는 단계가 빌드보다 먼저 실행되도록 합니다.
5. 키가 없거나 지문이 다르면 빌드를 실패 처리합니다. 새 키를 조용히 생성하지 않습니다.
6. APK의 applicationId, versionCode, versionName, 서명 SHA-256을 검증합니다.
7. 기존 앱이 설치된 테스트 기기에서 삭제 없이 업데이트 설치를 실제로 확인합니다.

## 4. 중요한 보안 및 복구 주의사항

- GitHub Actions cache는 암호화된 비밀 저장소가 아닙니다. GitHub 문서는 cache에 access token이나 로그인 자격 증명 같은 민감 정보를 저장하지 말라고 경고합니다. 공개 저장소에서는 cache 접근 범위를 특히 주의해야 합니다.
- Easy Health의 현재 방식은 기존 Android debug keystore를 재사용하는 프로젝트별 구현 사례입니다. Google Play 배포용 production signing key를 cache에 넣는 일반 보안 표준으로 확대 적용하면 안 됩니다.
- production signing key는 GitHub Actions secrets 또는 별도의 안전한 비밀 저장소를 통해 공급하고 접근 권한을 최소화해야 합니다.
- cache key를 바꾸면 기존 캐시를 찾지 못할 수 있습니다. 서명 키를 교체하는 것과 cache key를 바꾸는 것을 같은 작업으로 취급하지 않습니다.
- cache가 없어졌다고 새 키를 만들어 기존 앱을 업데이트할 수 있는 것은 아닙니다. 기존 키를 복구하지 못하면 동일 서명 업데이트는 불가능합니다.
- keystore 원문, 비밀번호, 개인 키는 이 저장소에 절대 커밋하지 않습니다.

## 5. Easy Navi 적용 시 체크리스트

- [ ] 기존 Easy Navi 설치본의 applicationId 확인
- [ ] 기존 설치본의 인증서 SHA-256 확인
- [ ] Easy Health 방식과 Easy Save 방식의 차이 확인
- [ ] 과거 Easy Navi 빌드의 동일 서명 키 복구 여부 확인
- [ ] 동일 키 복구가 확인된 뒤 고정 서명 적용
- [ ] versionCode 증가 확인
- [ ] 최종 APK의 applicationId / versionCode / SHA-256 검증
- [ ] 기존 앱 삭제 없이 업데이트 설치 테스트
- [ ] 검증 완료 후에만 APK 전달

## 참고 자료

- https://github.com/ryoobach/Easy-Health/blob/main/.github/workflows/android.yml
- https://github.com/ryoobach/Easy-Health/blob/main/app/build.gradle.kts
- https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching
