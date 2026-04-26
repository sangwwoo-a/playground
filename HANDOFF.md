# 오늘의놀이터 — 인수인계 문서

**작성일**: 2026-04-26
**작성자**: Claude (이전 세션)
**다음 담당자**: 새 세션 또는 다른 협업자

---

## 1. 프로젝트 한 줄 요약

한국 부모들이 동네 놀이터에서 또래 친구를 익명으로 매칭해주는 모바일 웹앱. 카카오맵에 근처 놀이터 핀이 뜨고, 시간대별로 "갈게요" 약속을 등록하면 다른 부모가 본다.

- **개발자**: 상우 (1인 개발, 육아휴직 중, 사업자 등록 X)
- **현재 상태**: 베타 운영 중, 외부 테스터 사용 가능
- **배포**: https://playground-ten-ochre.vercel.app/
- **소유 도메인**: devkorea1m.com (아직 Vercel 연결 X)

---

## 2. 기술 스택

- **프론트엔드**: 단일 HTML 파일 (index.html, 833줄). 백엔드 없음.
- **지도**: 카카오맵 SDK (JavaScript 키 `298aeb2bbbab45bba43939a427b6b381` — 도메인 화이트리스트 보호됨, 채팅 노출 OK)
- **데이터**: 행안부 어린이놀이시설 API → playgrounds-geo.json (그룹화 + 카카오 지오코딩 완료, 9.74MB)
- **배포**: GitHub (sangwwoo-a/playground) → Vercel 자동 배포
- **로그인**: 현재 없음 (베타). 동의 4개 체크박스 + 닉네임/나이/성별 입력만.

---

## 3. 파일 구조

```
오늘의놀이터/
├── index.html               # 앱 본체 (배포 대상, 833줄)
├── playgrounds.json         # 행안부 원본 5.4MB (배포 대상, 37,956곳)
├── playgrounds-geo.json     # 그룹화+지오코딩된 9.74MB (배포 대상, 24,242그룹)
├── fetch-playgrounds.py     # 데이터 갱신용 (배포 X, .gitignore에 포함, API 키 보유)
├── geocode-playgrounds.py   # 지오코딩 스크립트 (배포 X, .gitignore에 포함)
├── geocode-cache.json       # 지오코딩 캐시 (배포 X, .gitignore에 포함)
├── CLAUDE.md                # Claude 작업 규칙 (HTML 편집 위험, 사고 패턴)
├── HANDOFF.md               # 이 문서
└── .gitignore               # API 키 파일 제외 설정
```

---

## 4. 데이터 처리 흐름

```
행안부 API
   ↓ fetch-playgrounds.py (페이지네이션 + 화이트리스트 필터)
playgrounds.json (37,956곳, 평탄 배열)
   ↓ geocode-playgrounds.py
   │   - 도로명 주소로 그룹화 (24,242그룹)
   │   - 카카오 Geocoding API → 정확 좌표 (89.9% 성공)
   │   - 실패한 10.1%는 행안부 좌표 중앙값으로 fallback
playgrounds-geo.json (그룹 단위, 앱이 직접 읽음)
```

### 행안부 API 핵심 파라미터 (틀리기 쉬움)
- `pageIndex` (NOT `pageNo`)
- `recordCountPerPage` (NOT `numOfRows`)
- 좌표 필드: `latCrtsVl`, `lotCrtsVl`
- 화이트리스트 카테고리: `A003`(도시공원), `A010`(주택단지), `A011`(학교) — 키즈카페·야영장·놀이제공영업소 제외
- 운영중 필터: `operYnCd=B001` + 폐쇄일자(`clsgYmd`) 없음 + 실외(`idrodrCd=O002`)

---

## 5. 핵심 알고리즘 — 그룹화

같은 도로명 주소의 놀이터를 한 핀으로 묶어요. 행안부 좌표가 부정확해서 한 단지 안 놀이터들이 따로 박히는 문제 해결용.

### 그룹 키 정규화 (`groupKey()` 함수)
```javascript
// 1. 끝의 (산곡동) 같은 행정동 괄호 제거
// 2. 도로명 토큰 찾기 (로/길/번길/대로로 끝남)
// 3. 도로명 다음 토큰의 부번지 정규화 (46-2 → 46)
// 4. 도로명 없으면 끝의 숫자 토큰 제거
```

**중요**: 도로명 토큰(로/길/번길/대로) 뒤가 아닌 숫자는 주소가 아니라 시설명 일부일 가능성 → 제거. 이전에 이거 안 해서 "산곡 우성4차아파트 40" 같은 잘못된 그룹이 생긴 적 있음.

### 그룹 단위 vs 개별 놀이터 단위
- **그룹**: 표시(UI)와 핀 단위
- **개별 놀이터**: 예약(약속) 단위 — 한 단지 안 여러 놀이터에 각각 약속 등록 가능
- 그룹 핀에 표시되는 카운트 = 그룹 안 모든 놀이터의 약속 합계
- 한 사람이 동시에 한 약속만 가능 (`findMyReservationAnywhere()` 함수가 다른 놀이터의 본인 예약 검사)

---

## 6. UI 흐름

1. **첫 진입**: 카카오맵 + 동의 4개 체크박스 모달 띄움
2. **동의 완료** → "동의하고 다음 단계로" 버튼 → 닉네임/나이/성별 입력
3. **시작하기** → 모달 닫히고 지도에 빨간 풍선들 표시
4. **풍선 클릭**:
   - 단일 놀이터(items.length===1) → 시간대 모달 바로
   - 그룹(items.length>=2) → 그룹 리스트 모달 → 항목 클릭 → 시간대 모달
5. **시간대 모달**: 7시~21시 표시, "갈게요" 버튼 → 시작/종료 시간 두 번 탭 → 등록

### 다른 부모에게 노출되는 정보 (안전 원칙)
- ✅ 닉네임 (본인 식별 안 되도록 사용자가 가명 입력)
- ✅ 아이 나이 (3~11세 이상)
- ✅ 아이 성별 (남아/여아/비공개)
- ❌ 부모 닉네임 첫 글자 같은 식별 정보 노출 안 함
- ❌ 아이 실명·사진·학교 입력 금지 (동의서에 명시)

---

## 7. 현재 상태 요약 (2026-04-26 기준)

### ✅ 완료된 것
- 카카오맵 + 풍선 핀 + 모달 흐름 정상 동작
- 행안부 데이터 37,956곳 → 24,242그룹으로 그룹화
- 카카오 지오코딩으로 좌표 89.9% 정확도 보정
- "한 사람 한 약속" 안전 규칙 구현
- 동의 4개 체크박스 게이트 + 베타 진입 흐름
- Vercel 배포 정상 (commit `38d2915`)
- HTML 깨짐(코드 누수) 사고 → git checkout으로 복원 후 재배포

### ⏳ 진행 중이었던 것 (다음 작업자가 이어받기)
- **풍선 핀 디자인 개선** — 현재 풍선 밑에 흰색 막대(flag-pole)+점(flag-dot)이 있어서 위치가 모호하게 보임. 사용자 요청: pole/dot 제거하고 풍선 자체가 위치를 가리키도록.

### ❌ 진행 중 사고 발생
- Claude(이전 세션)가 Edit 도구로 CSS 수정 시도 → 파일이 833줄에서 792줄로 잘림 (CLAUDE.md에 명시된 위험을 알면서도 발생)
- git checkout으로 복원 완료, 현재 833줄 정상 상태
- PowerShell `-replace`로 다시 시도 → CRLF/LF 줄바꿈 문제로 일부 치환 실패

### 📋 미완료 작업
1. **풍선 디자인 개선** (위 ⏳ 참고) — 가장 시급
2. 신고(report) 버튼 추가
3. devkorea1m.com 도메인을 Vercel에 연결
4. 카카오 로그인 정식 구현 (현재 베타에선 닉네임만, 정식 출시 시 카카오 OAuth 필요)
5. 메인 랜딩 사이트 (`devkorea1m.com`) 만들기 + 서브앱 rewrites
6. 개인정보처리방침 + 이용약관 작성
7. favicon + Open Graph 태그
8. playgrounds-geo.json 크기 최적화 (현재 9.74MB)

---

## 8. 풍선 디자인 개선 — 미완성 작업의 설계 (다음 담당자용)

### 현재 상태 (commit 38d2915)
```html
<!-- HTML -->
<div class="playground-flag">
  <div class="flag-bubble">3명</div>
  <div class="flag-pole"></div>     <!-- 흰색 세로 막대 14px -->
  <div class="flag-dot"></div>      <!-- 흰색 동그란 점 10px -->
</div>
```

```css
.playground-flag { position: relative; cursor: pointer; transform: translate(-50%, -100%); }
.flag-bubble { background: #ff5a5f; ... border: 2px solid #fff; }
.flag-pole { width: 2px; height: 14px; background: #fff; margin: 0 auto; }
.flag-dot { width: 10px; height: 10px; border-radius: 50%; background: #fff; ... }
```

```javascript
// renderMarkers() 안
el.innerHTML = '<div class="flag-bubble ...">3명</div><div class="flag-pole"></div><div class="flag-dot"></div>';

const overlay = new kakao.maps.CustomOverlay({
  position: new kakao.maps.LatLng(group.lat, group.lng),
  content: el,
  yAnchor: 1,    // 컨텐츠 바닥이 좌표
  xAnchor: 0.5,  // 가로 가운데
});
```

### 사용자 피드백
> 몇 명 참여하는지 나오는 아이콘 아래로 굳이 하얀 줄 내리니까 오히려 위치가 직관적이질 않아.

### 권장 변경 방향
1. `.flag-pole`, `.flag-dot` 제거
2. `.flag-bubble`에 작은 꼬리(▼) `::after` 추가 → 일반 지도핀처럼 풍선 자체가 좌표를 가리키게
3. `.playground-flag`의 `transform: translate(-50%, -100%)` 제거 (yAnchor: 1로 충분)
4. `.flag-bubble`에 그림자 살짝 추가해 가독성 ↑

### 권장 CSS (참고용 — 검증 후 적용)
```css
.playground-flag {
  position: relative;
  cursor: pointer;
  padding-bottom: 8px;
  filter: drop-shadow(0 2px 4px rgba(0,0,0,0.25));
}
.flag-bubble {
  background: #ff5a5f; color: #fff;
  padding: 6px 12px; border-radius: 14px;
  font-size: 13px; font-weight: 700;
  white-space: nowrap; min-width: 50px;
  text-align: center; border: 2px solid #fff;
  position: relative;
}
.flag-bubble::after {  /* 흰색 꼬리 외곽 */
  content: ''; position: absolute;
  left: 50%; bottom: -7px; transform: translateX(-50%);
  width: 0; height: 0;
  border-left: 7px solid transparent;
  border-right: 7px solid transparent;
  border-top: 8px solid #fff;
}
.flag-bubble::before {  /* 색상 채움 꼬리 */
  content: ''; position: absolute;
  left: 50%; bottom: -3px; transform: translateX(-50%);
  width: 0; height: 0;
  border-left: 5px solid transparent;
  border-right: 5px solid transparent;
  border-top: 6px solid #ff5a5f;
  z-index: 1;
}
.flag-bubble.few::before { border-top-color: #4a90e2; }
.flag-bubble.empty::before { border-top-color: #aaa; }
.flag-bubble.lots::before { border-top-color: #ff3b30; }
```

### 권장 JS 변경
```javascript
// 한 줄만 변경
el.innerHTML = '<div class="flag-bubble ' + cls + '">' + (total === 0 ? '0명' : total + '명') + groupBadge + '</div>';
// (flag-pole, flag-dot 제거)
```

### ⚠️ 작업 시 절대 주의 — Edit 도구 쓰지 말 것
- 이전 사고 2회: Edit 도구가 큰 string 교체 시 파일 끝부분 잘림
- 권장 방법:
  1. PowerShell `-replace` (CRLF/LF 차이 주의 — 이 파일은 CRLF)
  2. 또는 사용자에게 메모장으로 직접 수정 부탁

---

## 9. Claude 작업 시 주의사항 (사고 기록)

### 사고 1: Edit 도구로 큰 string 교체 → 파일 잘림
- **빈도**: 이미 3번 이상 발생
- **증상**: 파일 끝부분이 잘려서 `</script>`, `</body>`, `</html>` 사라짐
- **재발 메커니즘**: Edit 도구가 300줄 이상 또는 5000자 이상 교체할 때 잘림
- **대응**: CLAUDE.md에 "Edit 큰 string 교체 금지" 규칙 명시. 그래도 재발함.

### 사고 2: cat append로 잘린 파일에 코드 이어 붙임 → 코드 누수
- **증상**: HTML body에 JavaScript 코드 텍스트 노출 (`s.forEach(it => ...` 같은 게 화면에 보임)
- **원인**: 잘린 파일에 새 함수 정의 또 추가 → 첫 `</html>` 뒤에 코드 텍스트 잔존
- **대응**: PowerShell로 첫 `</html>` 위치 찾아 그 이후 잘라냄. 이번에 한 번 더 발생해서 `38d2915` 커밋으로 수정.

### 사고 3: PowerShell Get-Content가 UTF-8 BOM 없는 파일을 CP949로 잘못 읽음
- **증상**: 한국어가 `?ㅻ뒛?섎??댄꽣`처럼 깨져 보임 — 실제 파일은 멀쩡
- **대응**: 항상 `-Encoding UTF8` 명시

### Claude에게 부탁할 때 쓸 만한 패턴
1. "파일 직접 수정하지 말고, PowerShell 명령어만 만들어줘" — 사용자가 검토 후 실행
2. "수정 전 백업 + 수정 후 5가지 검증(줄 수, `</html>` 1개, 코드 누수 검사, 끝부분, 함수 중복)"
3. "변경 후 git diff 보여주고 commit 전 사용자 승인 받기"

---

## 10. 의사결정 기록

| 결정 사항 | 선택 | 이유 |
|---|---|---|
| 지도 SDK | 카카오맵 | 한국 위치 데이터 정확, 무료 한도 충분 |
| 데이터 소스 | 행안부 어린이놀이시설 API | 8만 곳 등록, 단지 안 놀이터까지 포함 |
| 배포 | GitHub Pages X, Vercel O | Vercel이 한글 도메인 raw 파일 처리 더 잘함 |
| 로그인 | 베타엔 없음 | 카카오 비즈 앱 전환 X (개인사업자 등록 안 함) |
| 익명성 | 닉네임만 노출, 아이 나이/성별만 다른 부모에게 보임 | 안전 + 매칭 가능 균형 |
| 그룹화 키 | 도로명 주소 정규화 | 같은 단지 놀이터를 한 핀으로 묶기 |
| 좌표 정확도 | 카카오 Geocoding 우선, 행안부 중앙값 fallback | 행안부 좌표 부정확 (89.9% 카카오 보정) |

---

## 11. 보안

### 노출돼도 OK
- 카카오 JavaScript 키 `298aeb2bbbab45bba43939a427b6b381` (도메인 화이트리스트 보호)

### 절대 노출 X (`.gitignore` 처리됨)
- 행안부 OpenAPI 인증키 (fetch-playgrounds.py 안)
- 카카오 REST 키 (geocode-playgrounds.py 안)
- 두 키 모두 채팅엔 노출된 적 있음 — 보안 권고는 했지만 폐기 권장

---

## 12. 다음 담당자 시작 가이드

### 1단계: 현재 상태 파악
```powershell
cd "C:\Users\sangw\OneDrive\바탕 화면\오늘의놀이터"
(Get-Content index.html -Encoding UTF8).Count   # 833 나와야 정상
git status                                       # main, up to date
git log --oneline -5
```

### 2단계: 배포본 확인
- 브라우저로 https://playground-ten-ochre.vercel.app/ 시크릿 창 접속
- 동의 4개 → 닉네임/나이/성별 입력 → 지도에 빨간 풍선 떠야 함
- 풍선 클릭 → 모달 → "갈게요" → 시간 두 번 탭 → 등록 동작 확인

### 3단계: 풍선 디자인 작업 시작 (8번 섹션 참고)
- ⚠️ Edit 도구 쓰지 말 것
- PowerShell로 작업하거나 메모장으로 직접 수정
- 변경 후 검증 5단계 (CLAUDE.md 참고) 통과해야 commit

### 4단계: commit + push
```powershell
git add index.html
git status
git commit -m "Improve flag pin design - remove pole/dot, add tail"
git push
```

### 5단계: Vercel 자동 재배포 1~2분 대기 후 시크릿 창에서 확인

---

## 13. 연락처 / 권한

- **GitHub repo**: https://github.com/sangwwoo-a/playground (sangwwoo-a 소유)
- **Vercel 프로젝트**: playground-ten-ochre
- **Kakao Developers**: 사용자 계정으로 로그인 (앱 ID 본인만 접근)

---

이 문서가 도움이 되길. 다시 한번 사고에 대해 죄송합니다.
— 이전 세션 Claude
