# S01 · Window · Clear · Present

> 상태: 🚧 진행 중  ·  실행: `Sandbox12.exe`  ·  테스트: `Tests.exe --d3d12`

DX12에서 "화면에 색 하나 칠하기". DX11에선 `ClearRenderTargetView` + `Present` 두 줄이던 것이, DX12에선 **CPU와 GPU가 같은 메모리를 언제 쓰는지 앱이 직접 맞춰야 하는 문제**가 된다.

## 하드웨어 관점
- GPU 앞에는 **명령 큐**(컨베이어 벨트)가 있다. `ExecuteCommandLists`는 상자를 벨트에 올리고 바로 반환 — GPU는 나중에 처리한다.
- **Fence**는 벨트 위에 꽂는 번호표. GPU가 그 지점을 지나가야 값이 바뀐다. 값이 단조 증가하므로 `completed >= v` 비교 하나로 "v 이전 작업 전부 완료"를 판정.
- 명령이 실제로 적히는 메모리는 **Command Allocator**. GPU가 읽는 중에 Reset하면 GPU가 쓰레기를 실행한다 → 프레임 슬롯마다 Allocator 하나 + 슬롯별 펜스 값.

```
CPU  | 기록 F0 | 기록 F1 | (F0 완료 대기) 기록 F2 | …
GPU            | 실행 F0 |        실행 F1        | 실행 F2
```

## 구현
| 파일 | 내용 |
|---|---|
| `RHI/RHIDevice` | Debug Layer → Factory → `EnumAdapterByGpuPreference(HIGH_PERFORMANCE)` (하이브리드 노트북에서 dGPU 선택) → Device, InfoQueue |
| `RHI/RHIQueue` | Command Queue + Fence + 이벤트, `Signal` / `WaitCPU` / `Flush` |
| `RHI/RHISwapChain` | FLIP_DISCARD, RTV 힙, 리사이즈, tearing |
| `RHI/DX12Renderer` | 프레임 슬롯 ×2 (Allocator + fence 값), `BeginFrame` / `EndFrame` / `Resize` |

직접 구현 대상: `Signal`, `WaitCPU`, `BeginFrame`, `EndFrame`, `Resize` (`RHI/M0_YourCode.cpp`)

## 검증
GPU 게이트 트릭 — `queue->Wait(gate, 1)`로 GPU 타임라인을 일부러 멈추고, 다른 스레드가 N ms 뒤 `gate->Signal(1)`. GPU가 언제 끝날지 CPU가 통제하므로 "기다려야 할 때 기다리는가 / 기다리지 말아야 할 때 안 기다리는가"를 타이밍으로 판정한다.

| 테스트 | 확인하는 것 | 잡아내는 실수 |
|---|---|---|
| Q1 | 펜스 값 1부터 단조 증가 | 값 재사용 |
| Q3 / Q4 | GPU를 150 ms 막으면 WaitCPU도 150 ms 대기, 그동안 CPU 시간 ≈ 0 | no-op 대기, busy-wait |
| F3 | 백버퍼 리드백 == 클리어 색 | 배리어 / RTV 오류 |
| F4 | 300프레임 동안 슬롯 재진입 시 이전 프레임 항상 완료 | 슬롯 대기 누락 → Allocator 리셋 충돌 → 디바이스 제거 |
| F5 | GPU가 막혀도 EndFrame은 즉시 반환, 같은 슬롯의 BeginFrame은 대기 | 매 프레임 Flush (병렬성 상실) |
| R2 | 리사이즈 후 백버퍼 크기·클리어 정상 | 백버퍼 재획득 누락 |
| E1 | Debug Layer ERROR 0 | 전부의 2차 안전망 |

**판별 못 하는 것:** Resize 전 Flush 누락 — 실측 결과 DXGI `ResizeBuffers`가 자기 큐를 내부에서 비운다. 그래도 Flush가 필요한 이유는 DXGI가 모르는 앱 소유 해상도 종속 리소스(깊이 버퍼 등) 때문 → S02에서 깊이 버퍼 재생성 테스트로 판별 예정.

## 결과
_직접 구현 완료 후 스크린샷 · 테스트 결과 추가_

## 관찰
- 창 모드에선 vsync off + `ALLOW_TEARING`이어도 144 FPS(모니터 주사율) 고정, in-flight 2 — DWM 합성 경로로 추정. PresentMon으로 Present 모드 확인 예정.

## UE5 대응
`D3D12Adapter.cpp`(디바이스/어댑터), `D3D12Submission.cpp`(큐 제출), `D3D12Viewport.cpp`(스왑체인/Present)
