# DJ CLUBMIX 음원 신청 시스템

웹 + Android(Capacitor) 소스 프로젝트. **설정 전에는 서비스가 작동하지 않습니다.** Firebase 프로젝트, 배포, Android 빌드, 카카오 비즈메시지 공급자 계약이 필요합니다.

## 구현 기능
- 신청자: 곡명, 아티스트, YouTube 링크, 닉네임, 선택적 카카오 수신 번호/동의 입력
- 서버 저장: Cloud Functions를 통한 Firestore 저장
- 신청자: 36자리 비밀 조회 코드로 5초 간격 상태 자동 갱신 (정확한 의미의 push 실시간 스트리밍이 아닌 준실시간 폴링)
- 관리자: Firebase 이메일/비밀번호 로그인, Firestore 실시간 목록, 접수/승인/거절 상태 변경
- 카카오 알림톡: 상태 변경 시 외부 비즈메시지 공급자 호출 연동 코드 포함 (공급자별 API 페이로드 맞춤 수정 필요)
- 관리자 목록은 최근 200건까지만 표시. 운영 시 페이지네이션 추가 권장.

## 설치 및 배포
1. Node.js 22 설치. 프로젝트에서 `npm install`, `cd functions && npm install && cd ..`.
2. Firebase 프로젝트 생성, Firestore(프로덕션 모드), Authentication > Email/Password, Functions, Hosting, App Check 활성화. Functions 사용에는 Blaze 요금제가 필요할 수 있습니다.
3. Firebase 콘솔에서 웹 앱 등록 후 `web/firebase-config.js` 교체. App Check reCAPTCHA Enterprise 사이트 키도 입력하고 도메인 등록.
4. Authentication에서 **관리자 계정을 수동 생성**. 그 계정의 UID를 `firestore.rules`의 `REPLACE_ADMIN_UID`에 입력. UID는 공개된 프런트엔드에 넣지 않습니다.
5. `npm exec firebase login` 및 `npm exec firebase use -- --add` 또는 `npm exec firebase deploy -- --project YOUR_PROJECT_ID`.
6. `npm run deploy`로 Hosting, Functions, Firestore rules 배포. Firebase 콘솔에서 필요한 복합 색인 요청이 표시되면 생성.
7. 공개된 웹 주소 `https://PROJECT_ID.web.app` 확인. Android는 `capacitor.config.json`의 `server.url`을 해당 HTTPS 주소로 교체 후 `npx cap add android`, `npx cap sync android`, `npx cap open android`를 실행하여 Android Studio에서 APK/AAB 빌드. Android 래퍼는 웹 서버 연결이 필요합니다.

## 카카오 알림톡 연동 (추가 작업 필수)
- 카카오 비즈니스 채널, 알림톡 발신 프로필 및 승인 템플릿, 비즈메시지 제공업체 계약/인증이 필요합니다.
- Firebase Functions secrets `ALIMTALK_ENDPOINT`, `ALIMTALK_API_KEY`를 설정하세요 (`firebase functions:secrets:set ...`).
- `functions/index.js`의 notifyStatus POST body는 **공통 예시 형식**이며 공급업체의 실제 API 규격에 맞춰 수정해야 전송됩니다. 승인된 템플릿 코드로 `CLUBMIX_APPROVED`/`CLUBMIX_REJECTED`를 교체하세요.
- 현재 전송 실패 자동 재시도/중복 방지 및 배달 상태 기록은 미구현입니다. 상용 운영 전 큐와 멱등 키, 실패 로그를 추가하세요.

## 보안 및 운영 주의
- 비밀 조회 코드를 가진 사람은 해당 신청 내용을 볼 수 있습니다. 링크 공유 주의. 코드 분실 시 복구 기능은 없습니다.
- App Check는 자동화 남용 완화 수단이지 완전한 스팸 방지는 아닙니다. 대규모 운영 전 IP/계정 기반 rate limit, 신고, 데이터 보존·삭제 정책, 개인정보 처리방침을 마련하세요.
- 번호는 Firestore에 저장되며 관리자에게 노출됩니다. 개인정보 처리방침 및 보존기간·삭제 절차 마련 필수.
- Cloud Functions 생성 API는 관리자 인증 없이 누구나 호출할 수 있습니다. App Check 활성화 및 남용 방어가 중요합니다.
- Firestore 보안 규칙은 관리자 외 직접 읽기/쓰기 차단. 공개 신청은 callable Function을 통해서만 처리합니다.
