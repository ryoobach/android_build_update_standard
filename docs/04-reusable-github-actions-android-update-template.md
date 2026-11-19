# 재사용 가능한 Android GitHub Actions 업데이트 표준

이 문서는 Easy Health에서 확인한 **고정 서명 키 복원 → APK 서명 검증 → 버전 검증 → 무삭제 업데이트 확인** 절차를 다른 Android 프로젝트에도 적용하기 위한 공통 작업 지침입니다.

## 적용 원칙

- 기존 설치 앱을 유지하는 업데이트가 목표라면, 새 빌드를 만들기 전에 기존 설치본의 서명 인증서 SHA-256을 먼저 확보합니다.
- 기존 앱과 동일한 개인 키를 확보하지 못했다면 빌드 파이프라인을 고쳐도 기존 설치본의 무삭제 업데이트를 보장할 수 없습니다.
- 프로젝트별 applicationId, versionCode, versionName, 서명 지문은 반드시 해당 프로젝트에서 확인합니다. Easy Health의 값이나 지문을 다른 앱에 복사하지 않습니다.
- debug keystore 캐시 방식은 Easy Health의 실제 구현을 재현하는 예시입니다. 공개 저장소·상용 배포·Play 배포에서는 전용 release keystore를 GitHub Actions secrets 또는 승인된 비밀 저장소에서 공급하는 방식을 우선합니다.

## 표준 파이프라인 순서

1. **환경 준비** — JDK/Android SDK/Gradle 버전을 고정하고 workflow 실행 정보를 기록합니다.
2. **기존 서명 키 복원** — 프로젝트 전용 키를 안전한 저장소에서 복원합니다. 캐시를 사용하는 경우 프로젝트별 고정 cache key를 사용합니다.
3. **키 누락 시 중단** — 키가 없으면 임의로 새 키를 생성하지 말고 즉시 실패시킵니다. 새 키 생성은 기존 설치본과 서명이 달라질 수 있습니다.
4. **키 인증서 검사** — keytool로 인증서 SHA-256을 계산하고 프로젝트별 승인된 지문과 일치하는지 검사합니다.
5. **앱 식별자·버전 검사** — Gradle 설정에서 applicationId, versionCode, versionName을 확인합니다. applicationId는 기존 설치본과 같아야 하고 versionCode는 설치본보다 커야 합니다.
6. **APK 빌드** — 검증된 키만 사용해 APK를 서명합니다.
7. **APK 자체 검사** — apksigner verify --print-certs 및 Android SDK 도구로 서명, applicationId, versionCode를 다시 확인합니다.
8. **키와 APK 지문 대조** — 최종 APK 인증서 SHA-256이 복원한 키의 지문 및 프로젝트 기준값과 모두 같은지 검사합니다.
9. **설치 검증** — 기존 앱이 설치된 테스트 기기에서 덮어쓰기 업데이트를 실행하고 앱 데이터·설정이 유지되는지 확인합니다.
10. **산출물 게시** — 모든 검사가 통과한 APK만 artifact/release로 게시합니다. 실패한 빌드는 배포하지 않습니다.

## GitHub Actions 캐시 사용 시 주의

Easy Health에서 확인한 예시는 `actions/cache@v4`, `~/.android/debug.keystore`, 프로젝트 고정 cache key를 사용합니다. 캐시는 키를 다음 실행에서 재사용하는 편의 수단이지, 영구 백업이나 비밀 저장소가 아닙니다. 캐시 만료·삭제에 대비해 승인된 안전한 백업을 별도로 유지해야 합니다. 캐시가 사라졌다면 새 키를 만들어 진행하지 말고 백업에서 기존 키를 복구합니다.

## 필수 검증 예시

프로젝트별 실제 값을 변수로 지정해 검사합니다. 아래 값은 예시 자리표시자이며 그대로 사용하면 안 됩니다.

```bash
# 인증서 SHA-256 확인
keytool -list -v -keystore "$KEYSTORE_PATH" -alias "$KEY_ALIAS"

# APK 서명 검증 및 인증서 확인
"$ANDROID_HOME/build-tools/$BUILD_TOOLS_VERSION/apksigner" verify --verbose --print-certs "$APK_PATH"

# applicationId / versionCode / versionName은
# Gradle 설정 및 aapt dump badging 결과를 대조해 확인
```

CI는 위 출력값을 파싱해 프로젝트별 승인값과 자동 비교하고, 하나라도 불일치하면 실패해야 합니다.

## Easy Health 사례와 Easy Navi 적용 시 주의

Easy Health workflow에서 확인한 캐시 복원·지문 검증 흐름은 참고 구현입니다. 하지만 Easy Navi의 과거 빌드 artifact 세 개는 서로 다른 인증서 지문으로 서명된 것이 확인됐습니다. 따라서 Easy Health의 새 고정 키를 Easy Navi에 적용하는 것만으로는 과거 설치본의 무삭제 업데이트가 보장되지 않습니다. Easy Navi는 기존 설치본과 일치하는 원래 키를 먼저 찾아야 하며, 확인 전에는 배포용 업데이트 APK를 성공 처리하지 않습니다.

## 배포 승인 체크리스트

- [ ] 기존 설치본의 applicationId 확인
- [ ] 기존 설치본의 서명 인증서 SHA-256 확인
- [ ] 기존 개인 키의 안전한 복원 경로 확인
- [ ] CI에서 키 누락 시 실패하는지 확인
- [ ] 키 인증서 지문과 승인 지문 일치
- [ ] applicationId 일치 및 versionCode 증가
- [ ] 최종 APK 서명 검증 및 지문 일치
- [ ] 기존 설치 앱을 삭제하지 않고 업데이트 성공
- [ ] 사용자 데이터·설정 유지
- [ ] 검증 통과한 APK만 배포

## 오류 기록 규칙

실패한 단계, 원본 오류 메시지, 확인된 원인, 수정한 workflow/Gradle 파일, 커밋, 빌드 run, APK 지문, 실제 업데이트 테스트 결과를 `docs/03-android-build-update-error-log.md`에 누적합니다. 해결되지 않은 사항은 반드시 '조사 중' 또는 '미검증'으로 표시합니다.
