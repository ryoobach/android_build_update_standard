# Android 빌드·업데이트 오류 해결 기록

이 문서는 실제 발생한 오류와 검증된 해결 방법을 누적하는 작업 일지입니다. 추측한 해결책은 미검증으로 표시하고, 실제 빌드·설치 테스트가 끝나기 전에는 해결 완료로 기록하지 않습니다.

## 기록 규칙
각 사례마다 발생 날짜와 프로젝트/버전, 오류 메시지와 실패 단계, 확인된 원인과 추정 사항의 구분, 조사 파일 및 커밋, 적용한 수정, 빌드 결과와 APK 서명 정보, 기존 설치본 무삭제 업데이트 테스트, 재발 방지 체크리스트를 남깁니다.

## 사례 001 — Easy Navi release 서명 secrets 누락

- 프로젝트: https://github.com/ryoobach/Easy_Navi_Starter
- 확인 시점: 2026-11-19
- 증상: GitHub Actions run 25 및 26이 Prepare persistent release signing key 단계에서 실패.
- 확인된 원인: workflow가 요구하는 ANDROID_KEYSTORE_BASE64, ANDROID_KEYSTORE_PASSWORD, ANDROID_KEY_ALIAS, ANDROID_KEY_PASSWORD secrets가 비어 있었음.
- 현재 코드에서 확인한 설정: applicationId com.ryoobach.easynavistarter, versionCode 7, versionName 0.2.5.
- 과거 성공 빌드 run 20은 debug-fallback 방식으로 APK를 만들었고 versionCode 5 / versionName 0.2.3으로 확인됨.
- 주의: 과거 fallback 빌드의 서명 개인 키가 현재도 보존되어 있는지는 아직 확인되지 않았음. GitHub-hosted runner에서 매번 새 debug keystore를 만들었다면 실행 간 인증서가 달라질 수 있음.
- 미검증 사항: 과거 Easy Navi APK들의 인증서 지문 일치 여부, 현재 휴대폰 설치본의 지문과 일치 여부, 기존 앱 무삭제 업데이트 성공 여부.
- 해결 상태: 조사 중 — 해결 완료 아님.

### 다음 조치
1. 과거 성공 APK 각각의 서명 인증서 SHA-256을 추출·비교한다.
2. 기존 설치본과 일치하는 키를 저장소 기록, 안전한 백업, 기존 secrets 또는 과거 artifact에서 복구할 수 있는지 확인한다.
3. 동일 키가 확보되면 고정 서명 키 복원 절차를 적용한다.
4. 키 복원이 안 되면 새 키로 기존 앱을 업데이트할 수 있다고 주장하지 않는다.
5. 실제 설치 기기에서 삭제 없는 업데이트를 검증한다.

## 재발 방지 규칙
- Android 앱마다 기존 설치본의 서명 인증서 지문을 기준값으로 확보한 뒤 빌드 파이프라인을 변경합니다.
- 서명 키가 없거나 지문이 다르면 배포용 빌드를 실패 처리합니다. 임의의 새 키로 대체하지 않습니다.
- actions/cache는 키 재사용 수단일 뿐 유일한 백업으로 간주하지 않습니다. 캐시 만료·삭제 가능성을 고려해 안전한 백업 또는 비밀 저장소를 별도로 둡니다.
- 공개 저장소에 keystore, 비밀번호, 개인 키를 커밋하지 않습니다.
- workflow 검증용 버전 값과 Gradle의 실제 versionCode/versionName을 자동 대조합니다.
- APK가 생성됐다고 성공으로 처리하지 않습니다. 서명 지문과 기존 설치본의 무삭제 업데이트까지 확인해야 완료입니다.
- 프로젝트마다 debug/release 서명 정책이 다르므로 다른 앱 설정을 무비판적으로 복사하지 않습니다.

## 향후 오류 기록 템플릿
### 사례 NNN — 오류 제목
- 발생 날짜:
- 프로젝트 / 버전:
- 오류 메시지 / 실패 단계:
- 확인된 원인:
- 추정 원인:
- 조사한 파일 / 커밋:
- 적용한 수정:
- 빌드 결과:
- 서명 지문 / applicationId / versionCode:
- 기존 앱 무삭제 업데이트 테스트:
- 해결 상태: 미조사 / 조사 중 / 수정했으나 미검증 / 검증 완료
- 재발 방지:
- 다른 프로젝트 적용 시 주의사항:
## 사례 001 추가 조사 — 과거 Easy Navi APK의 서명 지문 비교

2026-11-19에 GitHub Actions의 과거 성공 artifact 3개를 실제 다운로드해 APK v2 서명 블록에서 인증서 DER을 추출하고 SHA-256을 계산했습니다.

| Workflow run | Artifact / 앱 버전 | 인증서 SHA-256 |
|---|---|---|
| 44460907735 | run 5 / app-debug.apk | 3d46abf3d890ca55ac249e492574a11430af2f765eaf39774e443062e57110f4 |
| 44462024398 | run 6 / app-debug.apk | 26466b1e0f0e0f27a413983882f0d88cff87cd68f32ad2bb4e3a8c7f473428a8 |
| 44469736454 | run 20 / v0.2.3 debug-fallback | 4ee6e76cef49049097d0fe0b87ba567642ac8734f5feb47dd6e93754eec4d323 |

### 확인된 결론
- 과거 세 APK의 서명 인증서 지문은 모두 서로 다릅니다.
- 따라서 과거 빌드들은 하나의 고정 debug keystore를 공유하지 않았습니다.
- APK 안에서 인증서 공개 정보는 추출할 수 있지만, 업데이트 서명에 필요한 개인 키는 복구할 수 없습니다.
- Easy Health의 캐시 복원 방식을 Easy Navi에 그대로 적용해 새 키를 만들면, 기존 Easy Navi 설치본과 서명이 달라져 업데이트가 거부될 수 있습니다.
- 기존 설치본의 버전과 인증서가 어느 artifact에 해당하는지 아직 사용자 기기에서 확인하지 못했습니다.
- 기존 키의 안전한 백업 또는 기존 secrets가 발견되지 않는 한, 동일 서명으로 무삭제 업데이트할 수 있는지는 미확정입니다.

### 현재 상태 및 안전 조치
- 해결 상태: 조사 중 — 빌드·배포 보류.
- 사용자에게 설치된 앱의 버전을 확인해 해당 artifact와 대조해야 합니다.
- 기존 키와 동일하다는 근거 없이 새 고정 키로 release APK를 빌드하거나 업데이트 가능하다고 안내하지 않습니다.
- 새 고정 키로 빌드하면 향후 업데이트 일관성은 확보할 수 있지만, 과거 설치본을 무삭제 업데이트할 수 있다는 뜻은 아닙니다.

## 사례 002 — Easy Navi Starter 0.2.5 빌드 차단 및 무삭제 업데이트 보호

- 확인 날짜: 2026-11-19
- 프로젝트: https://github.com/ryoobach/Easy_Navi_Starter
- 현재 소스 버전: applicationId `com.ryoobach.easynavistarter`, versionCode 7, versionName 0.2.5.
- 최신 workflow run: 44514987453 (run 26), 결과 failure.
- 확인된 workflow 실패 원인: `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD` 네 가지 persistent release signing secrets가 설정되지 않아 서명 키 준비 단계에서 중단됨.
- 과거 artifact에서 확인된 인증서 SHA-256은 run 5 / run 6 / run 20이 서로 모두 달랐음(사례 001 참조).
- 따라서 과거 빌드 중 어느 키가 현재 사용자 기기에 설치된 앱의 키인지 확인되지 않았고, 과거 개인 키를 복원할 근거도 아직 없음.
- Android 공식 업데이트 조건상 applicationId가 같아도 서명 인증서가 다르면 업데이트가 거부됨. versionCode만 올리거나 새 고정 키를 생성하는 것으로 이 문제를 해결할 수 없음.
- 안전 조치: 기존 앱을 삭제하지 않도록 새 키로 서명한 APK의 배포 및 업데이트 가능 안내를 보류함.
- 현재 상태: 차단 — 기존 설치본과 일치하는 서명 키 복구가 필요함.
- 다음 조치: 기존 signing keystore의 안전한 백업 또는 과거 빌드 환경에서 사용한 키를 찾는다. 복구 후 지문 일치, versionCode 증가, 기존 앱 무삭제 설치 테스트를 모두 통과한 APK만 배포한다.


## 사례 003 — Easy Navi Starter 0.4.3도 기기 보호 차단으로 설치 실패

- 확인 날짜: 2026-11-20
- 프로젝트: https://github.com/ryoobach/Easy_Navi_Starter
- 테스트 버전: 0.4.3 (versionCode 25로 설정)
- 테스트 기기 결과: 사용자가 실제 Android 기기에서 설치를 시도했으나, 기존과 동일하게 기기 보호 차단 안내가 표시되어 설치하지 못했다고 보고함.
- 진단 목적 수정: `app/src/main/AndroidManifest.xml`에서 `.PowerMenuAccessibilityService` 서비스 선언을 제거한 진단 빌드.
- 관련 커밋: `23b92315728e3ae7cb174f9a8ff7a12d3b63ac58` (versionCode/versionName 변경), `146e646638ac918699110b6189305c0fc3698c4d` (접근성 서비스 선언 제거).
- GitHub Actions run: [44686060202](https://github.com/ryoobach/Easy_Navi_Starter/actions/runs/44686060202), 2026-11-20 완료, conclusion=success.
- 빌드 결과: workflow의 APK 빌드 및 서명 검증 단계 성공. 이것은 기기 설치 성공이나 Play Protect 통과를 의미하지 않음.
- 확인된 결론: 접근성 서비스 선언을 제거한 것만으로는 사용자 기기의 설치 차단이 해소되지 않았음. 접근성 서비스가 유일한 차단 원인이라는 가설은 지지되지 않음.
- 미확인 사항: 실제 차단 주체가 Google Play Protect인지, Samsung Auto Blocker/기기 보호 기능인지, 다른 설치 정책인지; 정확한 차단 상세 사유; 해당 APK의 인증서가 현재 설치본과 일치하는지.
- 현재 상태: 수정했으나 미검증 / 차단 지속. 원인 확인 전 권한이나 코드를 무작정 제거하지 않음.

### 다음 진단 순서
1. 휴대폰에 표시되는 차단 화면의 정확한 제목·문구와 상세 정보/앱 검사 결과를 확인해 차단 주체를 구분한다. 이미 해제한 Auto Blocker를 다시 끄라고 반복 안내하지 않는다.
2. 0.4.3 APK의 인증서 SHA-256, applicationId, versionCode를 workflow 로그에서 확보하고 기준값과 대조한다.
3. AndroidManifest.xml, 요청 권한, 서비스·리시버, 네트워크/외부 코드, 의존성 변경 이력을 이전 정상 APK와 비교한다.
4. 공식 Android Emulator의 AVD에서 동일 APK의 신규 설치와 업데이트 설치를 각각 시험한다. 에뮬레이터 설치 성공은 실기기 보안 차단 해제의 증거로 취급하지 않는다.
5. 차단 상세 사유가 확보되기 전에는 Play Protect 또는 제조사 보안 기능을 우회하는 방법을 안내하지 않는다. 오탐이 의심되면 Google Play Protect 앱 검토/이의 제기 절차 등 공식 경로를 검토한다.
6. 실기기 설치 성공, 앱 실행, 업데이트 보존 여부를 별도로 기록한다.

### 재발 방지 규칙 추가
- Android 앱 배포 전 공식 Android Emulator(AVD)에서 대상 API 수준별 APK 설치·실행 테스트를 수행한다.
- 최소한 신규 설치, 동일 패키지 업데이트, 앱 실행, 권한 요청, 재실행을 검사하고 결과를 보관한다.
- 실기기에서만 나타나는 제조사 보안/Play Protect 차단은 별도 실기기 테스트 항목으로 관리한다.
- CI 빌드 성공, APK 서명 검증 성공, 에뮬레이터 설치 성공, 실기기 설치 성공을 서로 다른 상태로 기록한다. 어느 하나도 다른 항목의 통과로 간주하지 않는다.
- 차단 원인이 확인되지 않은 상태에서 서비스·권한을 제거하는 진단 빌드는 임시 진단용으로만 사용하며, 제품 기능 손실 여부를 명시한다.
