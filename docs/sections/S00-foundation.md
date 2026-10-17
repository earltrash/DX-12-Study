# S00 · Foundation — DX11 엔진 & UE5 스타일 에디터

> 상태: ✅  ·  태그: `s00-foundation`

DX12로 넘어가기 전, 출발점이 되는 DX11 엔진(DTEngine)을 **검증 가능한 상태**로 만들고 에디터를 UE5 방식으로 정리한 단계.

![editor](../images/s00-ue5-editor.png)

## 엔진 기반
- Unity 스타일 GameObject / Component, Scene 직렬화 (JSON 텍스트 `.scene` + CBOR 바이너리 `.scnb`, 같은 스키마)
- Assimp FBX 임포트, CPU 애니메이션 (스키닝 가중치 보존, 메시 공간 스키닝 `v · invBind · global · meshGlobalInv`)
- Undo/Redo (서브트리 스냅샷, GO id 보존), ImGuizmo 기즈모
- 규약: 왼손 좌표계, row-vector 행렬, CPU에서 Transpose 업로드 → HLSL 기본 column_major (GPU에서 실측 확인)

## UE5 스타일 에디터
- **에디터 카메라는 씬 밖** — UE의 `FEditorViewportClient`처럼 에디터가 소유. 씬 저장/로드와 무관하고, Play 중엔 씬의 Main Camera로 렌더
- UE 5.7 소스 수치 그대로: 마우스 감도 0.2°/px, 속도 테이블 `{0.033 … 32}`, RMB+휠 ±10%, 휠 돌리 0.96 m/노치, 가속 20000 cm/s² · 감쇠 10/s
- 비행 물리는 `dv/dt = a − k·v`를 **해석해**로 적분 → 프레임레이트 무관 (UE의 프레임 단위 적분은 60 fps에서 종단속도가 약 17% 낮음)
- 포커스 거리 `d = r / sin θ` (구 전체가 시야 원뿔 안에 들어오는 조건)
- 레이아웃: 메뉴 / 툴바 / 뷰포트 / Outliner / Details / 상태바(Content Drawer, Output Log, 콘솔 명령) — 순수 함수 `ComputeEditorLayout`으로 계산, 씬은 뷰포트 rect에만 렌더

## 검증
| 영역 | 테스트 | 결과 |
|---|---|---|
| 모델 | FBX 6종 임포트, 바인드 포즈 CPU 스키닝 == 원본 메시 | ✅ |
| 셰이더 | 컴파일 / 리플렉션 / VS↔PS 링키지 | ✅ |
| 직렬화 | JSON·CBOR 왕복이 원본 값을 모두 보존 | ✅ |
| 에디터 카메라 (C1–C11) | UE 기본값, 종단속도 20 m/s, 60/240 fps 결과 동일, 궤도 거리 보존, 포커스 | ✅ 33/33 |
| 레이아웃 (L1–L4) | 7개 해상도에서 패널 겹침 없음 + 면적 합 = 화면 | ✅ 17/17 |

전체 **213 PASS / 0 FAIL** (WARN 4 = 원본 에셋의 텍스처 파일 누락 등)

## 배운 점 · 함정
- 한글 경로 셰이더 `#include` — `CP_ACP` 변환은 실패, UTF-8로 처리해야 함
- 플립 모델 스왑체인 + tearing 플래그 없으면 vsync off여도 DWM vblank에 묶여 60→30 fps로 양자화
