# 손인형뽑기 (Hand Claw Machine) — 통합 제작 & 배포 지시서
> **최신 통합본 (v2, 2026-06-21).** 이전 5개 수정 + 확률 집기 + 실제 기계 외형/자동 배출까지 모두 반영한 "최종 목표 상태" 스펙.
> 새 Claude Code 세션에서 이 파일을 읽히고 **"이 파일대로 만들어줘"** 한 줄이면 됨.
> 진행: **caveman + cavecrew** (긴 구현은 서브에이전트에 위임하고 요약만 회수 → 메인 컨텍스트 절약).

---

## 0. 작업 규칙
- caveman + cavecrew로 진행.
- 결과물: **단일 정적 파일 `index.html` 하나.** 빌드 도구·번들러·npm 설치 없음.
- 모든 라이브러리는 CDN(importmap/script)로만 로드.

## 1. 목표
웹캠으로 손을 인식해 조종하는 **3D 인형뽑기**. 실제 인형뽑기 기계 외형의 유리 케이스 안에서,
손을 움직이면 집게가 따라오고, 손을 쥐면(핀치) 집게가 내려가 인형을 집고, 손을 펴면
집게가 **자동으로 배출구로 이동해 인형을 떨어뜨린다.** 카메라가 안 되면 마우스/터치 폴백.

## 2. 기술 스택 (전부 CDN)
- `three@0.160` — ESM importmap
- `cannon-es@0.20` — 인형 더미 물리
- `@mediapipe/hands` + `@mediapipe/camera_utils` — 손 랜드마크. **카메라 시작 시 lazy load.**
- ⚠️ `getUserMedia`/MediaPipe는 **HTTPS 필수** → GitHub Pages에서 정상. 로컬은 `localhost`만 허용.

---

## 3. 기계 외형 (실제 인형뽑기 기계)
- 직육면체 **캐비닛**: 빨간 지붕(top) + 흰/빨강 프레임 모서리 + **4면 반투명 유리벽** + 바닥.
- 윗면은 집게가 다니도록 **열려 있되**, 4면 유리벽으로 둘러싸여 인형이 절대 밖으로 못 나감.
- 전면 좌측 하단에 **상품 배출구(chute)** + 그 아래 수거함(밖에서 보이게).
- 장식: 전면 하단 컨트롤 패널에 둥근 버튼 3개(장식용), 빨간 띠 트림.
- 케이스 내부가 잘 보이도록 조명/시점 조정. 위에서 살짝 내려다보는 고정 시점.

## 4. 집게(claw) 모양
- 컴팩트한 허브(작은 원통/구) + **프롱 3개가 아래로 늘어뜨려져** 끝이 안쪽으로 모이는 크레인 집게.
- 각 프롱 = 위쪽 직선 + 아래쪽 안쪽으로 꺾인 갈고리(2 segment). 피벗은 허브 상단.
- open/close = 피벗 회전 각도: **열림 ≈ 28°(적당히만), 닫힘 ≈ 5°(끝이 거의 한 점)**.
- 중앙에 떠 있는 흰색 사각형/판 같은 잔여물 없을 것. 금색 메탈 재질.
- 천장에서 케이블(가는 cylinder)로 연결, 하강/상승 시 케이블 길이 동기화.

## 5. 인형(plush)
- 둥근 인형 **~26개**: 캔디색 랜덤, 작은 검은 눈 2개, 일부는 귀. 반지름 0.30~0.46.
- **스폰/더미는 집게 도달 범위(`clawX`/`clawZ`) 안**, 그리고 **chute 칸막이 영역은 제외**한 메인 영역에만.

---

## 6. 케이스 밀폐 + 배출구 분리 (인형이 배출구로 굴러 들어가는 문제 해결)
- 인형 더미는 유리 케이스(4벽 + 바닥)로 **완전 밀폐** → 굴러서 절대 못 나감.
- **배출구(chute)는 케이스 내부 한 코너에 '칸막이 벽'으로 분리**하고, 그 코너 바닥에 **구멍** → 아래 수거함으로 떨어지게.
- 칸막이 높이(`dividerH`): 인형이 굴러서는 **절대 못 넘되**, **집게는 위로 넘어갈 수 있는** 높이.
- 결과: 자유롭게 굴러다니는 인형은 chute에 못 들어가고, **오직 집게가 들어 옮길 때만** 들어감.
- 물리 콜라이더는 시각 벽과 정확히 일치. ball↔ground/wall restitution 낮게(안 튀게).

## 7. 조작
- **시작 오버레이**: "카메라로 시작" / "마우스·터치로 시작". 우상단 모드 토글 버튼.
- **카메라 모드**
  - 손 위치(landmark 9, x는 셀피 미러)로 집게 XZ 이동.
  - 좌우(x): `handXRange`를 `clawX` 전체로 정규화. 앞뒤(z): `handYRange`(중앙 밴드)를 `clawZ` 전체로 **확장 매핑** + 클램프 (앞뒤 인식 약한 문제 해결). z 방향이 직관과 맞는지 점검(필요시 반전).
  - 핀치 = `dist(thumbTip[4], indexTip[8]) / handSize < pinchThreshold`, `handSize = dist(wrist[0], lm[9])`.
  - 떨림 방지용 저역통과 스무딩(`smooth`).
- **마우스/터치 모드**: 드래그로 이동(레이캐스트 평면 좌표), 누르면 잡기 시작·떼면 release.
- **카메라 권한 거부/로드 실패 시 자동으로 마우스 모드 폴백** + 토스트.

## 8. 집게 하강/상승 속도
- 내려가 인형 잡는 데 **약 1초**, 다시 올라오는 데 **약 1초**.
- descend/ascend를 **시간 기반 보간**(`descendTime`, `ascendTime`)으로.

## 9. 잡기 로직 (확률적 집기 + 자동 배출) — state machine
`idle → descend(핀치 시작) → close(인형 선택 + 등급 굴림) → ascend → hold(핀치 유지·손 따라옴) → (release) deliver(자동 chute 이동) → drop → idle`

### 9-1. 확률적 집기 + 가변 악력 (실제 기계가 악력 조절하는 그 수법)
- close에서 반경 내 가장 가까운 인형 1개 선택 후 **등급**을 굴림:
  - **FIRM**(확실): 끝까지 안 놓침.
  - **WEAK**(헐거움): 잡지만 ascend/hold/**deliver 이동 중** 매 프레임 `weakSlipPerSec`로 슬립 판정 → 떨어짐.
  - **TEASE**(약올림): 살짝 들었다가 `teaseDropDelay` 후 곧 놓침.
- **pity**(사장 자비): 연속 실패(미배달)가 `pityThreshold` 넘으면 다음 집기는 FIRM 강제 후 카운터 리셋.
- 집은 인형 물리 바디는 sleep, 들고 있는 동안 매 프레임 집게 밑으로 위치 동기화.
- WEAK/TEASE는 프롱을 완전히 닫지 말고 살짝 열린 채로(헐겁게 물린 느낌). 슬립 순간 프롱 탁 벌리고 인형 흔들리며 떨어지게.

### 9-2. 자동 배출 (실제 기계처럼)
- hold 상태에서 손을 펴면(release) → 그 자리에서 놓지 말고, **집게가 자동으로 배출구(chute) 위로 이동** → 도착하면 스스로 인형을 놓음.
- `deliver` 이동 중에는 **사용자 입력(손/마우스) 무시**.
- chute에 떨어진 인형은 구멍으로 빠져 수거함에 → **점수 +1**.
- 점수는 **chute 통과 시에만** +1. 배달 성공하면 연속실패 카운터 0 리셋. 슬립/약올림으로 케이스 바닥에 떨어지면 점수 없음 + 연속실패 +1.

## 10. HUD
- 좌상단 "담은 인형 N" 카운터, 하단 조작 힌트, 중앙 상단 토스트.
- 카메라 모드 시 우하단 작은 웹캠 프리뷰(미러). 다크 글래스모피즘.

---

## 11. 튜닝 상수 (파일 상단 `CFG` 객체로 모을 것)
```
// 유리 케이스 내부
caseX:[-2.5, 2.5], caseZ:[-2.0, 2.0], caseH:3.6,
// 집게 도달 범위 (케이스 안쪽, 인형 반경 고려)
clawX:[-2.2, 2.2], clawZ:[-1.7, 1.7],
restY:3.2, grabY:0.5, topY:4.6,
descendTime:1.0, ascendTime:1.0,        // 하강/상승 각 ~1초
grabRadius:0.6,
// 배출구(코너 칸막이 + 바닥 구멍)
chuteX:-1.7, chuteZ:1.4, chuteHalf:0.6, dividerH:0.85,
// 인형 스폰(집게 도달 범위 안, chute 코너 제외)
spawnX:[-1.9, 1.9], spawnZ:[-1.5, 0.9], nBalls:26,
// 카메라 손 인식
pinchThreshold:0.55, handXRange:[0.1, 0.9], handYRange:[0.25, 0.78], smooth:0.25,
// 확률 집기
grabFirmRate:0.35, grabWeakRate:0.40,   // TEASE = 나머지(0.25)
weakSlipPerSec:0.6, teaseDropDelay:0.35, pityThreshold:4
```

## 12. 기타
- 모바일 세로 비율 대응, `devicePixelRatio` 2 캡, `touch-action:none`.
- 파일명은 **반드시 `index.html`** (루트 URL 직링크).
- 마우스/카메라 모드 둘 다 위 모든 동작 동일 적용.

---

## 13. 배포 — GitHub Pages
- 계정: **perteacher** (가명). 새 **public** 레포 `claw-machine`.
- 최종 URL: **https://perteacher.github.io/claw-machine/**
- ⚠️ 인증(`gh auth`)·로그인은 **사용자가 직접**. CC는 자격증명/비밀번호 입력하지 말 것.

### A. gh CLI 경로
```bash
git init
git add index.html
git commit -m "feat: hand claw machine"
gh repo create perteacher/claw-machine --public --source=. --remote=origin --push
gh api -X POST repos/perteacher/claw-machine/pages \
  -f "source[branch]=main" -f "source[path]=/" 2>/dev/null \
  || echo "→ Settings>Pages에서 수동 활성화 필요"
```

### B. 수동 경로
1. GitHub에서 public 레포 `claw-machine` 생성
2. `index.html` 푸시
3. Settings → Pages → Source: **Deploy from a branch → main / (root)** → Save
4. 1~2분 뒤 https://perteacher.github.io/claw-machine/ 확인

### C. 배포 후 체크
- [ ] https로 열리고 카메라 권한 요청이 뜸
- [ ] 모바일 세로에서 레이아웃·터치 정상
- [ ] three / cannon-es / mediapipe CDN 로드 OK (콘솔 에러 없음)
- [ ] 인형이 케이스 밖/배출구로 굴러나가지 않음
- [ ] 집게 하강·상승 각 ~1초, release 시 자동 배출 동작
- [ ] 확률 집기(FIRM/WEAK/TEASE) + pity 동작

---

## 14. (선택) 다음 단계 메모
- **앱인토스**: 카메라 테스트 검증부터. Pages 안정화 후 Toss 웹뷰 카메라 권한·세로비율 호환성 점검.
- 확장 후보: 제한시간/라이프, 효과음·햅틱, 인형 희귀도, 난이도 프리셋(쉬움/현실).
- 블로그(perteacher)용: 제작 과정 + CC 프롬프트로 dev-diary 1편.
