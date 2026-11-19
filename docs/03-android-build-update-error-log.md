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