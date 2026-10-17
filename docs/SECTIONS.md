# Monster Engine — Section 상세 계획 (S00 ~ S37)

> 확정: 2026-10-17 · 38 Sections · 8 Phases
> 각 Section은 **설명 → 목적 → 기대(배우는 것 / 관찰할 현상) → 결과(산출물 + 검증 기준) → 참고** 순서.
> ⚡ = DX11에서 구현해 본 기능의 DX12 이식 · 🆕 = 새로 배우는 기능
> 참고의 [약어]는 맨 아래 **자료 목록** 참조.

## 공통 완료 조건 (Definition of Done)
1. `Tests.exe` 해당 Section 테스트 **FAIL 0**, 전체 회귀 FAIL 0
2. 테스트 동안 **D3D12 Debug Layer ERROR/CORRUPTION 0건** (해당 시 GBV 포함)
3. 핵심 실수에 대한 **뮤테이션 확인** — 일부러 버그를 넣어 테스트가 실제로 FAIL하는지. 판별 못 하는 경우는 문서에 명시
4. **S03 이후**: GPU 타이밍 수치를 기록 (비교가 있는 Section은 표로)
5. `docs/sections/SXX-*.md` 작성 + 스크린샷 + git 태그 `sXX-name`
6. UE5 대응 코드 위치 기록

## 한눈에 보기

| Phase | Sections |
|---|---|
| 0 · Foundation | S00 |
| 1 · Core | S01 Clear/Present · S02 Triangle · S03 Timing & Debug · **S04 Compute 기초** · S05 Constants/Instancing/Upload · S06 Textures/Descriptors · S07 DX11 이식 · S08 Shadow |
| 2 · Compute | S09 groupshared · S10 Reduction/Atomics · S11 Wave/Scan · S12 Utilities · S13 Compute Skinning · S14 Particles · S15 Culling/Indirect · S16 Clustered Lighting |
| 3 · Geometry | S17 Geometry Shader · S18 Tessellation · S19 Mesh/Amplification |
| 4 · Architecture | S20 Memory · S21 Bindless/PSO Cache · S22 Render Graph · S23 Deferred · S24 Async/Multithread |
| 5 · Screen-Space | S25 SSAO · S26 SSR |
| 6 · Ray Tracing | S27 DXR 기초 · S28 Dynamic AS · S29 RT Shadow/AO · S30 RT Reflections · S31 Path Tracer · S32 ReSTIR DI · S33 DDGI · S34 SER/OMM |
| 7 · Modern | S35 Visibility Buffer · S36 TAA/Upscaling · S37 Work Graphs |

**순서 원칙**
- Core에서 DX12가 DX11과 다른 부분(동기화, 메모리, 바인딩)을 먼저 익힌다.
- Compute 기초를 S04로 당긴 이유: Root Signature와 UAV만 있으면 되고, 이후 모든 Section에서 도구로 쓴다 [K].
- Architecture는 패스가 충분히 많아져 수동 관리가 고통스러워진 **뒤에** 한다. 그래야 추상화가 왜 필요한지 체감한다.
- Screen-Space를 RT 직전에 두는 이유: 같은 장면에서 래스터 기법 ↔ RT ↔ Path Tracer 레퍼런스를 비교하기 위해서다 [J].

---

# Phase 0 · Foundation

## S00 · Foundation — DX11 엔진 & UE5 스타일 에디터 ⚡ ✅
- **설명**: DTEngine(DX11)을 Monster Engine으로 옮기고, 자동 테스트로 기존 기능을 검증하고, 에디터를 UE5 방식으로 정리했다.
- **목적**: DX12 이식의 **비교 기준(레퍼런스 이미지·수치)**과 회귀 테스트망 확보.
- **기대**: 컴포넌트/씬 직렬화/FBX/스키닝이 수치로 맞는지 확인. UE 5.7 소스의 카메라 수치와 물리를 이해.
- **결과**: Tests 213 PASS / 0 FAIL. 씬 밖 에디터 카메라, UE5 레이아웃, Output Log. 비행 물리는 해석해로 적분해 프레임레이트와 무관하게 만듦.
- **참고**: UE 5.7 `EditorViewportClient.cpp`, `CameraController.cpp`

---

# Phase 1 · Core — DX11 드라이버가 숨기던 것들

## S01 · Window · Clear · Present 🆕 🚧
- **설명**: Device / Queue / SwapChain / Fence로 화면을 한 색으로 칠하고 Present한다. 프레임 2개를 동시에 비행시킨다.
- **목적**: CPU와 GPU가 **서로 다른 타임라인**에서 돈다는 것과, 공유 메모리(Command Allocator)의 수명을 앱이 책임진다는 것을 체득.
- **기대**:
  - Fence 값을 큐의 "번호표"로 이해: 단조 증가하므로 `>=` 비교 하나로 그 이전 작업 전부의 완료를 판정한다.
  - 슬롯 대기를 빼면 Allocator를 GPU가 쓰는 중에 Reset하게 되고, 그 결과 디바이스가 제거되는 것을 직접 본다.
  - 창 모드 Present가 DWM에 묶이는 현상(144 FPS 고정)을 관찰.
- **결과**:
  - `RHIDevice` / `RHIQueue` / `RHISwapChain` / `DX12Renderer`, `Sandbox12`.
  - 검증 — GPU 게이트(`queue->Wait`로 GPU를 의도적으로 멈춤) 타이밍 테스트로 다음을 확인한다:
    - WaitCPU가 실제로 기다린다 (Q3).
    - 대기 중 CPU 시간이 ≈0, 즉 busy-wait가 아니다 (Q4).
    - EndFrame은 기다리지 않는다 (F5).
    - 슬롯 재진입 시점에는 그 슬롯의 이전 프레임이 이미 끝나 있다 (F4).
    - 백버퍼 리드백 값이 클리어 색과 같다 (F3).
  - **판별 불가**: Resize 전 Flush 누락. DXGI가 내부에서 큐를 비우기 때문 → S02에서 판별.
- **참고**: [B] L1, [C] HelloWindow·D3D12Fullscreen, [D] Work submission, [E] I~II, [F], [G1] 1·10강 · UE5 `D3D12Submission.cpp`, `D3D12Viewport.cpp`

## S02 · Hello Triangle 🆕
- **설명**: 삼각형(→ 큐브)을 그린다. 다음을 모두 처음 다룬다:
  - Root Signature, PSO, DXC(SM 6.x) 셰이더 컴파일
  - VB/IB를 Upload Heap → Default Heap으로 복사 + 배리어
  - 깊이 버퍼(DSV)
  - Agility SDK 설정 (`D3D12SDKVersion` export)
- **목적**: DX11의 "상태를 그때그때 세팅"이 DX12에선 **PSO 하나로 미리 컴파일**된다는 점과 그 이유(셰이더 재컴파일 히칭 제거)를 이해. 리소스가 **어느 메모리(시스템 vs VRAM)에 있는지**를 앱이 고른다는 것을 이해.
- **기대**:
  - Upload Heap(CPU 쓰기 가능, PCIe 너머)에서 바로 그리기 vs Default Heap(VRAM)으로 복사 후 그리기의 차이 ([G1] 2~3강과 같은 실험).
  - COPY_DEST → VERTEX_AND_CONSTANT_BUFFER 전이가 왜 필요한지.
- **결과**:
  - `RHIBuffer`, `PSO` 빌더, DXC 컴파일 파이프라인, 깊이 버퍼.
  - 검증:
    - 픽셀 리드백: 삼각형 내부 = 정점 색 보간값, 외부 = 클리어 색.
    - 겹친 두 삼각형은 그리는 순서와 무관하게 앞쪽이 보인다 (깊이 테스트).
    - **S01에서 미룬 판별**: Resize 시 깊이 버퍼도 재생성 → Flush를 빼면 Debug Layer 에러가 나는지 뮤테이션으로 확인.
- **참고**: [B] L2, [C] HelloTriangle, [E] IV~V, [F], [G1] 2·3·8강, [H] 3·5·6·8강

## S03 · GPU Timing & Debug 🆕
- **설명**: Timestamp Query로 패스별 GPU 시간을 재고, PIX 마커/캡처, DRED(디바이스 제거 시 브레드크럼·페이지 폴트 정보)를 붙인다.
- **목적**: 이후 모든 Section을 **수치로 비교**하기 위한 측정 도구. 디바이스 제거가 났을 때 "어느 패스에서"를 알 수 있게.
- **기대**:
  - GPU 시간은 CPU 시계와 별개다: `GetTimestampFrequency`, `ResolveQueryData`, 결과는 몇 프레임 뒤에 읽어야 함.
  - Present 대기와 GPU 작업 시간을 분리해서 보기 (S01의 144 FPS 원인 확인, PresentMon 병행).
- **결과**:
  - `GpuProfiler`(스코프 타이머), 에디터 오버레이에 패스별 ms, PIX 이벤트, DRED 로그 출력.
  - 검증:
    - 작업량을 2배로 하면 측정 시간도 ≈2배가 된다 (선형성).
    - `ID3D12Device5::RemoveDevice`로 강제 제거 시 DRED 출력에 마지막 패스 마커가 남는다.
- **참고**: [C] PredicationQueries, [D] Counters/queries · Debugging

## S04 · Compute 기초 🆕 (Core로 당김)
- **설명**:
  - Compute Shader로 텍스처를 처리한다: 반전 / 그레이스케일 / 그라디언트 생성.
  - 다룰 것: Dispatch 수 계산, `SV_DispatchThreadID / GroupID / GroupThreadID`, `[numthreads]`, UAV(처음으로 shader-visible 디스크립터 힙 사용), UAV 배리어.
  - 결과를 백버퍼로 복사하거나 화면에 그린다.
- **목적**: GPU를 "그리는 기계"가 아니라 **스레드 그룹 단위로 일하는 병렬 프로세서**로 보는 출발점. 같은 작업을 Pixel Shader와 Compute로 각각 해서 비교.
- **기대**:
  - 스레드 그룹은 SM/CU 하나에 올라가고, 32(NVIDIA)·32/64(AMD) 레인 단위 웨이브로 실행된다.
  - 그룹 크기(8×8 / 16×16 / 32×1 / 64×1)에 따른 점유율·성능 차이를 S03으로 측정.
  - 이미지 크기가 그룹 크기의 배수가 아닐 때의 경계 처리.
- **결과**:
  - Compute PSO / Root Sig, 최소 디스크립터 힙.
  - 검증:
    - 결과 리드백이 CPU 레퍼런스와 **비트 단위 일치**.
    - 1000×777처럼 배수가 아닌 크기에서 경계 픽셀이 정상.
    - 의존하는 두 Dispatch 사이에 UAV 배리어를 빼는 실험. 하드웨어 타이밍에 따라 재현이 불안정할 수 있어 GBV로 확인.
  - 그룹 크기별 타이밍 표.
- **참고**: [A] 13장, [K] 2장 "Drawing with compute", [L] GPU 아키텍처(SIMT, 다이버전스)

## S05 · Constants · Instancing · Upload Ring 🆕
- **설명**:
  - 프레임별 **선형 업로드 할당기**(링 버퍼)로 상수를 올린다. CBV는 256바이트 정렬.
  - Root Constant / Root CBV / Descriptor Table 세 방식 비교.
  - StructuredBuffer + `SV_InstanceID`로 인스턴싱.
  - **GPU Upload Heap**(ReBAR — CPU가 VRAM에 직접 쓰기) vs Upload + Copy 비교.
- **목적**: DX11 `Map(WRITE_DISCARD)`가 뒤에서 하던 **버퍼 리네이밍**을 직접 구현. GPU가 아직 읽는 메모리를 덮어쓰지 않게 만든다.
- **기대**:
  - 링이 한 바퀴 돌아 in-flight 프레임 영역을 침범하면 화면이 깜빡이는 현상을 일부러 만들어 본다.
  - 오브젝트 1만 개를 드로우콜 1만 번 vs 인스턴싱 1번 → CPU 시간 차이.
  - ReBAR 직접 쓰기의 지연·대역폭 특성 (쓰기 결합 메모리라 CPU 읽기는 극도로 느림).
- **결과**:
  - `UploadRing`, 인스턴스 버퍼, 1만 큐브 장면.
  - 검증:
    - GPU 게이트로 GPU를 멈춘 상태에서 링이 in-flight 영역을 절대 덮지 않는다 (패턴을 쓰고 리드백으로 확인).
    - 정렬 assert.
  - 방식별 CPU/GPU 시간 표.
- **참고**: [B] L3 동적 버퍼, [C] HelloConstBuffers, [G1] 5강, [G2] 10강 GPU Upload Heaps, [A] 16장 인스턴싱, [M]

## S06 · Textures · Descriptor Allocator 🆕
- **설명**:
  - 텍스처 업로드: `GetCopyableFootprints`, RowPitch 256바이트 정렬, `CopyTextureRegion`.
  - DDS/PNG 로드 (DirectXTex), sRGB 포맷, Static Sampler.
  - **디스크립터 할당기**: CPU 전용 staging 힙(영구 보관) + shader-visible 링 힙(프레임별), `CopyDescriptorsSimple`.
- **목적**: DX11 `PSSetShaderResources(t0)` 뒤에서 드라이버가 만들던 **디스크립터 테이블**을 직접 설계. 디스크립터는 GPU가 읽는 메모리 위의 32~64바이트 구조체다.
- **기대**:
  - shader-visible 힙은 하나만 바인딩할 수 있고 크기가 제한된다(Tier에 따라) → 왜 링/풀 구조가 필요한지.
  - sRGB: 저장 값 188 ↔ 선형 0.5. 샘플러가 디코딩한다.
  - Mip은 이 단계에선 DDS에 들어 있는 것을 사용 (S12에서 Compute로 생성).
- **결과**:
  - `DescriptorAllocator`(CPU staging + GPU 링), `TextureLoader12`, 텍스처 큐브.
  - 검증:
    - 텍셀 리드백 일치.
    - sRGB 변환값 검증.
    - GPU 게이트 상태에서 디스크립터 링이 in-flight 테이블을 덮지 않는다.
- **참고**: [B] L3·L4, [C] HelloTexture, [E] III·IV, [G1] 4·6·7·9강, [H] 7강

## S07 · DX11 기능 이식 — PBR · HDR · Skinning (+ Blending · Stencil · MSAA) ⚡
- **설명**: DX11에서 만든 기능을 DX12로 옮긴다.
  - Cook-Torrance PBR, 노멀맵, IBL(이 단계에선 미리 계산된 맵을 로드)
  - HDR RT(R16G16B16A16_FLOAT) + 톤매핑 + Bloom
  - VS 스키닝, 정적 애니메이션
  - 추가로 블렌딩(투명, 뒤→앞 정렬), 스텐실(아웃라인 / 거울), MSAA 4x + 명시적 Resolve
- **목적**: 셰이더는 거의 그대로이고 **바인딩·상태·배리어만 다르다**는 걸 확인. RT → SRV, DSV → SRV 전이 패턴을 손에 익힘.
- **기대**:
  - DX11 상태 객체(Blend/DepthStencil/Raster)가 PSO 필드로 흡수된다. 그래서 투명/불투명마다 PSO가 따로 필요하다.
  - MSAA: DX12에선 `ResolveSubresource`를 직접 호출해야 한다. HDR에선 톤매핑 전 resolve vs 후 resolve 결과 차이를 관찰.
- **결과**:
  - DX11 Client와 같은 장면을 렌더.
  - 검증:
    - **DX11 결과와 픽셀 비교** (최대 오차·PSNR 기준).
    - 스키닝 결과가 기존 CPU 스키닝 테스트와 일치.
    - MSAA 엣지 픽셀이 부분 커버리지 값을 가진다.
- **참고**: [A] 8~11·18·19·23장, [G1] 11강 멀티 머티리얼, [M] Helios·Delt06

## S08 · Shadow Mapping (CSM) 🆕
- **설명**:
  - Cascaded Shadow Maps: 실용 분할(로그/균등 혼합), 캐스케이드별 직교 투영.
  - **텍셀 스냅**으로 반짝임(shimmering) 제거.
  - PCF, 비교 샘플러, Slope-scaled Depth Bias.
- **목적**: 섀도 알고리즘을 처음 제대로 구현. **S29 RT 섀도와 비교할 기준선**.
- **기대**:
  - Acne / Peter-panning이 바이어스와 어떻게 맞바뀌는지.
  - 카메라가 서브텍셀만큼 움직일 때 스냅이 없으면 가장자리가 떨린다.
- **결과**:
  - CSM 4단계 + 디버그 뷰 (캐스케이드 색칠).
  - 검증:
    - 캐스케이드 경계가 각 절두체 조각을 완전히 포함한다 (CPU 수학 테스트).
    - 서브텍셀 이동 시 그림자맵 원점이 텍셀 격자에 고정된다.
    - 알려진 그림자 지점의 픽셀 확인.
- **참고**: [A] 20장, [M]

---

# Phase 2 · Compute Shader — GPU 하드웨어 모델 → 응용

## S09 · groupshared & 동기화 🆕
- **설명**: 분리형 가우시안 블러. 타일 + 경계(apron)를 groupshared에 적재 → `GroupMemoryBarrierWithGroupSync` → 계산.
- **목적**: SM 안의 **온칩 공유 메모리**(L1과 같은 물리 SRAM)를 직접 관리. 전역 메모리 왕복을 줄이는 원리.
- **기대**:
  - 공유 메모리 없는 버전 vs 있는 버전의 대역폭·시간 차이.
  - 32 뱅크 × 4바이트 구조에서 생기는 **뱅크 충돌**을 패딩으로 해소.
  - 그룹당 공유 메모리 사용량이 늘면 동시에 올라가는 그룹 수(점유율)가 줄어든다.
- **결과**:
  - Blur 패스 (S07 Bloom에 적용).
  - 검증: CPU 레퍼런스와 ε 이내.
  - 반경 4/8/16 × 순진한 구현 / 공유 메모리 / 패딩 버전의 타이밍 표.
- **참고**: [A] 13장 블러, [L] GPU 아키텍처

## S10 · Reduction & Atomics 🆕
- **설명**: 휘도 히스토그램(256 bin)을 만들고, 가중 평균 → **자동 노출**(시간 적응)에 쓴다. `InterlockedAdd`.
- **목적**: 수백만 스레드가 하나의 값을 만드는 **리덕션** 패턴과 원자 연산의 경합 비용 이해.
- **기대**:
  - 전역 원자 연산만 쓰는 버전 vs 그룹 로컬 히스토그램(groupshared) 후 병합 버전의 경합 차이.
  - 결과를 CPU로 읽지 않고 GPU 안에서 노출값까지 끝내기.
- **결과**:
  - Auto Exposure (S07 HDR에 연결).
  - 검증:
    - 히스토그램 bin 합 == 픽셀 수.
    - 평균 휘도가 CPU 레퍼런스와 일치.
  - 경합 버전별 타이밍.
- **참고**: [M] Delt06 auto exposure

## S11 · Wave Intrinsics & Prefix Sum 🆕
- **설명**:
  - `WaveGetLaneCount`, `WaveActiveSum`, `WavePrefixSum`, `WaveActiveBallot`.
  - 다단계 병렬 스캔(1M 원소), Stream Compaction.
- **목적**: 그룹보다 더 작은 실행 단위인 **웨이브(워프)** 안의 레인끼리 레지스터로 직접 통신하는 방법. 이후 컬링·파티클·클러스터 라이팅의 핵심 부품.
- **기대**:
  - 웨이브 크기는 하드웨어마다 다르다 (NVIDIA 32, AMD 32/64) → 코드가 그 가정에 의존하지 않게.
  - S10 리덕션을 웨이브 연산으로 다시 짜서 groupshared 버전과 비교.
- **결과**:
  - `GpuScan`, `GpuCompact` 유틸.
  - 검증:
    - 스캔/컴팩션 결과가 CPU와 정확히 일치 (크기 1, 31, 32, 33, 1M 경계 케이스).
    - 레인 수 조회값 기록.
- **참고**: [C] HLSL SM 6.0 Wave, [L]

## S12 · Compute Utilities — Mip 생성 · IBL 사전 계산 🆕
- **설명**:
  - Compute로 Mip 체인 생성. sRGB 텍스처는 UAV를 만들 수 없으므로 선형 처리.
  - IBL 사전 계산: Irradiance, GGX Prefiltered Env(중요도 샘플링), BRDF LUT.
- **목적**: DX12엔 DX11 `GenerateMips`가 없다. S07에서 로드해 쓰던 IBL 맵을 엔진이 직접 만들게 한다.
- **기대**:
  - 홀수 크기 Mip의 다운샘플 가중치.
  - Prefilter의 샘플 수와 노이즈/시간 트레이드오프.
- **결과**:
  - `MipGenerator`, `IblBaker`.
  - 검증:
    - Mip 평균이 CPU 박스 필터와 일치.
    - BRDF LUT가 레퍼런스 값과 일치.
    - S07의 미리 계산된 맵과 이미지 비교.
- **참고**: [B] L4 Compute Mip, [M] Helios compute mip·IBL

## S13 · Compute Skinning 🆕
- **설명**: 스키닝을 VS에서 Compute로 옮긴다. 결과 정점 버퍼(UAV → VB 전이)를 메인 패스, 섀도 패스, 이후 RT BLAS(S28)에서 재사용.
- **목적**: "정점당 한 번 계산하고 여러 번 쓰기" 구조. RT 동적 메시의 전제.
- **기대**: 패스가 늘어날수록 VS 스키닝은 같은 계산을 반복한다. 패스 수에 따른 시간 차이.
- **결과**:
  - Skinning CS + 공유 정점 버퍼.
  - 검증: 기존 CPU 스키닝 결과와 최대 오차 비교 (Tests의 바인드 포즈 테스트 재사용).
  - 패스 수(1/2/4) × VS / CS 타이밍.
- **참고**: [G3] 16강 변형 메시, [M] Delt06 스키닝

## S14 · GPU Particles 🆕
- **설명**:
  - Emit / Simulate / Compact를 Compute로 처리 (Append/Consume 또는 카운터 버퍼).
  - Indirect Draw 인자 생성.
  - 투명 정렬은 Bitonic Sort, 깊이 버퍼 충돌은 선택.
- **목적**: CPU가 개수조차 모르는 상태에서 GPU가 스스로 그릴 양을 정하는 구조. S15의 예행연습.
- **기대**: 파티클 100만 개, Bitonic 정렬 비용(n log²n).
- **결과**:
  - 파티클 시스템 (에디터 컴포넌트).
  - 검증:
    - 생존 + 사망 == 최대 개수 (보존).
    - 정렬 결과 단조성 검사.
    - 시드를 고정하면 결과가 결정적.
- **참고**: [C] nBodyGravity, [M]

## S15 · GPU Culling & ExecuteIndirect 🆕
- **설명**:
  - 오브젝트 바운드를 Compute로 절두체 컬링 → Hi-Z 오클루전(2-phase).
  - Command Signature + Count Buffer로 `ExecuteIndirect`.
- **목적**: **CPU가 드로우를 내리지 않는** 렌더링. GPU-driven의 시작.
- **기대**: 오브젝트 10만 개에서 CPU 드로우 vs Indirect 비교. 오클루전 컬링이 이전 프레임 깊이에 의존해 생기는 문제를 2-phase로 해결.
- **결과**:
  - Culling 패스 + Indirect 렌더러.
  - 검증:
    - 절두체 컬링 결과가 CPU 결과와 같은 집합.
    - 드로우 카운트 리드백.
    - 이미지가 컬링 끈 버전과 일치.
- **참고**: [C] ExecuteIndirect·PredicationQueries, [A] 16장, [K] GPU Driven, [M] Delt06 Hi-Z

## S16 · Tiled / Clustered Lighting 🆕
- **설명**: 화면을 froxel(예: 16×9×24)로 나누고 Compute로 라이트 → 클러스터를 배정. 라이트 인덱스 리스트를 만들어 Forward+ 셰이딩. 히트맵 디버그 뷰.
- **목적**: 라이트 수백~수천 개를 픽셀당 관련 라이트만 계산하게.
- **기대**: 깊이 분할(지수 vs 선형)에 따른 클러스터당 라이트 수 분포. S11 스캔으로 리스트를 압축.
- **결과**:
  - 라이트 1000개 장면.
  - 검증: 각 클러스터의 라이트 리스트가 CPU 전수 검사 결과를 **빠짐없이 포함** (false negative 0).
  - 라이트 수별 타이밍.
- **참고**: Olsson 2012 (Clustered Shading), [M]

---

# Phase 3 · Geometry Pipeline

## S17 · Geometry Shader 🆕
- **설명**:
  - 노멀/탄젠트 시각화, 단일 패스 와이어프레임(무게중심 좌표)
  - 퍼 셸 / 포인트 스프라이트 확장
  - `SV_RenderTargetArrayIndex`로 큐브맵 6면 단일 패스 렌더
- **목적**: 레거시 GS를 직접 써 보고 **왜 느린지**(출력 순서 보장, 가변 증폭의 버퍼링 비용)를 측정으로 확인. S19와 비교하기 위한 기준선.
- **기대**: 증폭률이 커질수록 성능이 비선형으로 떨어진다. 같은 효과를 인스턴싱 / VS로 바꾼 버전과 비교.
- **결과**:
  - GS 데모 4종.
  - 검증: Pipeline Statistics Query(GSInvocations, GSPrimitives)가 기대값과 일치.
  - GS vs 대체 구현 타이밍 표.
- **참고**: [A] 12장

## S18 · Tessellation 🆕
- **설명**: Hull(패치 상수, 테셀레이션 계수) → 고정 테셀레이터 → Domain(디스플레이스먼트). 화면 공간 엣지 길이 기반 LOD 지형.
- **목적**: 하드웨어 테셀레이터 단계 이해 + **크랙 없는 LOD**.
- **기대**: 공유 엣지의 계수가 다르면 크랙이 생긴다. 계수에 따른 삼각형 수 / 시간.
- **결과**:
  - 테셀레이션 지형.
  - 검증:
    - 공유 엣지 계수 일치 (CPU 검사).
    - Pipeline Statistics의 HS/DS 호출 수.
    - 미리 세분한 메시와 시간 비교.
- **참고**: [A] 14장

## S19 · Mesh & Amplification Shader 🆕
- **설명**:
  - Meshlet 빌드 (meshoptimizer, 64 정점 / 124 삼각형)
  - Amplification 단계에서 메쉬렛 절두체 + 법선 콘 컬링 → Mesh Shader 출력
  - FL 12_2 필요
- **목적**: GS와 고정 정점 파이프라인을 대체한 **컴퓨트형 지오메트리 파이프라인**. S17과 같은 효과를 비교하고, S35 Visibility Buffer의 기반을 만든다.
- **기대**: 메쉬렛 크기와 컬링률·성능. 정점 재사용(캐시)이 앱 손으로 넘어온다.
- **결과**:
  - Meshlet 렌더러.
  - 검증:
    - 정점 파이프라인 결과와 이미지 일치.
    - 컬링된 메쉬렛 수 리드백.
  - **GS(S17) vs 정점 파이프라인 vs Mesh Shader 타이밍 표**.
- **참고**: [C] D3D12 Mesh Shaders, [M] Delt06 Meshlets, Karis *Nanite* (SIGGRAPH 2021)

---

# Phase 4 · Engine Architecture

## S20 · Memory Management 🆕
- **설명**:
  - Heap 타입: DEFAULT / UPLOAD / READBACK / GPU_UPLOAD.
  - Committed vs Placed vs Reserved 리소스, Resource Heap Tier.
  - **앨리어싱**(같은 메모리를 여러 임시 리소스가 공유) + Aliasing Barrier.
  - D3D12MA 통합.
  - `QueryVideoMemoryInfo` 예산, Residency(`MakeResident` / `Evict`).
  - (선택) Tiled 리소스 텍스처 스트리밍.
- **목적**: VRAM을 앱이 직접 관리. S22 Render Graph의 트랜지언트 메모리 앨리어싱 전제.
- **기대**:
  - 작은 리소스를 Committed로 만들면 64KB 정렬 때문에 낭비가 생긴다 [C SmallResources].
  - 예산 초과 시 OS가 페이징하면서 프레임이 급락하는 것을 관찰.
- **결과**:
  - D3D12MA 기반 할당기, 메모리 통계 패널.
  - 검증:
    - 앨리어싱된 리소스들의 결과가 정확 (Aliasing Barrier를 빼면 FAIL하는지 뮤테이션).
    - 종료 시 할당 누수 0.
    - `ReportLiveDeviceObjects` 0.
- **참고**: [C] ReservedResources·Residency·SmallResources, [D] Memory management, [E] III, [G4], [M] Helios D3D12MA

## S21 · Bindless & PSO Cache 🆕
- **설명**:
  - SM 6.6 `ResourceDescriptorHeap[index]`, 전역 Root Signature 하나, 머티리얼은 인덱스로.
  - `ID3D12PipelineLibrary` PSO 캐시 + 셰이더 핫리로드.
- **목적**: 디스크립터 테이블 관리(S06)를 **인덱스 하나**로 단순화. PSO 생성 히칭을 캐시로 제거.
- **기대**:
  - 비균일 인덱스(`NonUniformResourceIndex`)가 필요한 경우.
  - 캐시 히트 시 PSO 생성 시간 감소량.
- **결과**:
  - Bindless 경로.
  - 검증:
    - 기존 바인딩 경로와 이미지 일치.
    - 캐시 히트 시 생성 시간 측정.
    - 핫리로드 후 결과가 정상.
- **참고**: [C] PipelineStateCache·DynamicIndexing, [M] Helios bindless, [A] 15장 동적 인덱싱

## S22 · Render Graph 🆕
- **설명**: 패스가 읽고 쓰는 리소스를 선언하면 그래프가 순서를 정하고, **배리어를 자동으로 삽입**하고, 트랜지언트 리소스를 앨리어싱(S20)하고, 안 쓰는 패스를 제거한다. 그래프 시각화 포함.
- **목적**: S01~S21에서 손으로 넣던 배리어를 자동화. UE5 RDG를 읽을 수 있는 수준.
- **기대**: 수동 배리어 대비 실수 0. 트랜지언트 메모리 절감량. (선택) Enhanced Barriers로 전환.
- **결과**:
  - `RenderGraph`, 기존 패스 이식.
  - 검증:
    - 알려진 그래프에 대해 생성된 배리어 목록이 기대값과 일치.
    - 손으로 짠 버전과 이미지 일치.
    - 메모리 사용량 비교.
- **참고**: O'Donnell *FrameGraph* (GDC 2017), [K] framegraph, [M] Delt06 RenderGraph · UE5 RDG

## S23 · Deferred Shading ⚡
- **설명**: G-Buffer(옥타헤드럴 노멀 인코딩, 러프니스/메탈릭 패킹) + 라이트 패스(S16 클러스터 재사용). Forward+와 비교.
- **목적**: Forward / Deferred의 대역폭 vs 셰이딩 비용 트레이드오프를 측정으로 확인. RT(S27~)와 SSR/SSAO의 입력 버퍼 마련.
- **기대**: 해상도에 따라 G-Buffer 대역폭이 커지는 것, MSAA와의 궁합 문제.
- **결과**:
  - Deferred 경로.
  - 검증: Forward 결과와 이미지 오차 기준 이내.
  - 해상도별 Forward+ vs Deferred 타이밍.
- **참고**: [M] Helios·Delt06, [J] 래스터 G-Buffer

## S24 · Async Compute & 멀티스레드 기록 🆕
- **설명**:
  - Compute 큐 추가, 큐 간 Fence.
  - 리소스의 큐 소유권 (COMMON 상태 프로모션/디케이).
  - SSAO / 파티클을 섀도 패스와 **동시에** 실행.
  - 스레드별 CommandList / Allocator 풀로 병렬 기록.
- **목적**: GPU의 그래픽 엔진과 컴퓨트 엔진을 동시에 쓰고, CPU 코어도 병렬로 쓰기.
- **기대**:
  - 겹쳐도 빨라지지 않는 경우: 두 작업이 같은 유닛(ALU / 대역폭)을 다투면 이득이 없다.
  - 기록 스레드 수에 따른 CPU 시간 변화.
- **결과**:
  - 다중 큐 스케줄링 (Render Graph 확장).
  - 검증:
    - 두 큐의 타임스탬프 구간이 실제로 겹친다.
    - 단일 큐 버전과 결과 일치.
  - 프레임 시간 비교.
- **참고**: [C] nBodyGravity·Multithreading, [D] Multi-engine sync, [G2] 6~9강

---

# Phase 5 · Screen-Space (RT 비교 기준선)

## S25 · SSAO (GTAO) 🆕
- **설명**: 깊이 + 노멀 기반 수평선(horizon) AO, 공간/시간 디노이즈.
- **목적**: S29 RTAO와 비교할 기준선. 화면 공간 정보만으로는 볼 수 없는 것(화면 밖, 가려진 면)의 한계 이해.
- **기대**: 화면 가장자리·얇은 물체에서 생기는 오차.
- **결과**:
  - GTAO 패스.
  - 검증:
    - 평면만 있는 장면에서 AO = 1 (가림 없음).
    - 알려진 코너 형상에서 기대값 범위.
  - 타이밍.
- **참고**: [A] 21장, [M]

## S26 · SSR 🆕
- **설명**: Hi-Z 기반 화면 공간 레이 마칭, 실패 시 IBL로 폴백, 러프니스에 따른 블러.
- **목적**: S30 RT 반사와 비교할 기준선.
- **기대**: 화면 밖 / 가려진 물체가 반사에서 사라지는 아티팩트.
- **결과**:
  - SSR 패스.
  - 검증:
    - 거울 평면에서 알려진 물체가 기대 픽셀에 반사된다.
    - 화면 가장자리에서 폴백이 동작.
- **참고**: [M] Delt06 SSLR

---

# Phase 6 · Ray Tracing

## S27 · DXR 기초 🆕
- **설명**:
  - BLAS / TLAS 빌드, State Object.
  - Shader Table 레이아웃: 레코드 정렬, 셰이더 식별자 32바이트.
  - RayGen / Miss / ClosestHit.
  - **Any Hit**(알파 테스트)와 **Intersection Shader**(절차적 구체).
- **목적**: 래스터와 완전히 다른 실행 모델(가속 구조 순회 + 셰이더 테이블 디스패치) 이해.
- **기대**:
  - SBT 인덱싱 공식 (`InstanceContributionToHitGroupIndex` + 지오메트리 인덱스 × 스트라이드 …)에서 실수하기 쉬운 지점.
  - Any Hit 비용.
- **결과**:
  - DXR 렌더러 (1차 가시성).
  - 검증:
    - RT 1차 가시성 이미지(인스턴스 ID / 깊이)가 래스터 결과와 일치.
    - 알파 테스트 잎이 래스터의 알파 테스트와 일치.
- **참고**: [C'] Hello World·Simple Lighting·Procedural Geometry, [G3] 1~7·12·13강, [J] NVIDIA Part 1~2

## S28 · Dynamic Acceleration Structures 🆕
- **설명**:
  - 움직이는 강체는 TLAS 갱신(update vs rebuild).
  - **스키닝 메시**는 S13 출력으로 BLAS refit(`ALLOW_UPDATE`) vs rebuild.
  - Compaction.
- **목적**: 실제 게임(애니메이션 캐릭터가 있는 FPS)에서 RT를 쓰기 위한 필수 단계.
- **기대**: refit을 반복하면 BVH 품질이 떨어져 트레이스 시간이 점점 늘어난다 → 주기적 rebuild 정책이 필요.
- **결과**:
  - 동적 AS 매니저.
  - 검증: 애니메이션 캐릭터의 RT 그림자 실루엣이 래스터 섀도와 일치.
  - 프레임 경과에 따른 refit vs rebuild 트레이스 시간 그래프.
- **참고**: [G3] 14~16강, [J] NVIDIA "Going Further"(TLAS 갱신)

## S29 · RT Shadow & AO 🆕
- **설명**: Inline Ray Query(Compute / PS 안에서 `RayQuery`)로 면광원 소프트 섀도와 RTAO.
- **목적**: **S08 CSM / S25 GTAO와 같은 장면에서 비교**.
- **기대**: RayGen 파이프라인과 Inline Ray Query의 쓰임 차이. 레이 수(spp)와 노이즈.
- **결과**:
  - RT Shadow / RTAO.
  - 검증:
    - 하드 섀도 RT 결과가 섀도맵과 엣지 픽셀을 제외하고 일치.
    - AO가 Path Tracer(S31) 레퍼런스에 수렴.
  - CSM vs RT, GTAO vs RTAO 이미지·타이밍 비교.
- **참고**: [C'] Real-Time Denoised AO, [G3] 10강, [J] AO·그림자 레이 튜토리얼

## S30 · RT Reflections & Denoising 🆕
- **설명**: 러프니스 기반 반사 레이 + 시공간 디노이저 (SVGF 계열: 시간 누적 + 분산 유도 공간 필터).
- **목적**: **S26 SSR과 비교**. 1 spp 신호를 쓸 수 있는 이미지로 만드는 디노이징 이해.
- **기대**: 고스팅과 반응성(시간 누적 길이)의 트레이드오프.
- **결과**:
  - RT 반사 + 디노이저.
  - 검증: Path Tracer 레퍼런스 대비 RMSE (디노이즈 전 / 후).
  - SSR vs RT 비교 이미지.
- **참고**: [G3] 9·11강, [J] 시간 누적, Schied *SVGF* (HPG 2017)

## S31 · Path Tracer (레퍼런스) 🆕
- **설명**: 프레임 누적 패스 트레이서. GGX 중요도 샘플링, Next Event Estimation, MIS, Russian Roulette.
- **목적**: 이후 모든 실시간 기법의 **정답 이미지**.
- **기대**: 노이즈가 1/√N으로 줄어드는 것. MIS가 없을 때 생기는 파이어플라이.
- **결과**:
  - 레퍼런스 모드 (에디터 토글).
  - 검증:
    - **White Furnace Test**: 에너지 보존, 균일 환경에서 흰 물체가 사라져 보여야 함.
    - 수렴 곡선이 1/√N.
- **참고**: [J] 확산 GI·GGX GI, PBR Book, [L] "엔진 + PBR + 패스트레이서" 포트폴리오

## S32 · ReSTIR DI 🆕
- **설명**: 수많은 라이트(S16의 1000개)에 대한 직접광. RIS(Resampled Importance Sampling) + 시간 / 공간 재사용.
- **목적**: 라이트가 많아도 픽셀당 섀도 레이 몇 개로 끝내기.
- **기대**: 편향 / 비편향 버전 차이, 재사용으로 노이즈가 급감하는 모습.
- **결과**:
  - ReSTIR DI 패스.
  - 검증: 비편향 버전의 평균이 전수 계산과 노이즈 범위 안에서 일치.
  - spp 대비 노이즈 비교.
- **참고**: Bitterli *ReSTIR* (SIGGRAPH 2020)

## S33 · DDGI 🆕
- **설명**: 프로브 격자, 프로브당 레이 → 옥타헤드럴 Irradiance / 거리 아틀라스, 체비셰프 가시성으로 빛샘 억제.
- **목적**: 실시간 간접광(GI). Path Tracer와 비교.
- **기대**: 프로브 간격과 빛샘, 시간 수렴 속도.
- **결과**:
  - DDGI 시스템 + 프로브 시각화.
  - 검증:
    - Path Tracer 대비 간접광 오차.
    - 빛샘 테스트 장면 (얇은 벽).
- **참고**: Majercik *DDGI* (JCGT 2019), [J] 확산 GI

## S34 · DXR 1.2 — SER & OMM 🆕
- **설명**:
  - **SER**(Shader Execution Reordering): 히트 결과로 스레드를 재정렬해 다이버전스를 감소.
  - **OMM**(Opacity Micromaps): 알파 테스트 Any Hit 호출을 줄임.
  - SM 6.9 / 최신 Agility SDK 필요.
- **목적**: 최신 하드웨어(Blackwell)의 RT 가속 기능 사용.
- **기대**: 재질이 다양한 장면 / 잎이 많은 장면에서의 이득 측정.
- **결과**:
  - SER / OMM 경로.
  - 검증: 이미지가 변하지 않는다.
  - On / Off 타이밍 비교.
- **참고**: [C'] Hello SER·Sakura Forest·Opacity Micromaps

---

# Phase 7 · Modern GPU-Driven

## S35 · Visibility Buffer 🆕
- **설명**:
  - 래스터 단계에선 삼각형 ID + 인스턴스 ID만 기록하고, 재질 해석 패스에서 무게중심 좌표와 도함수를 재구성해 셰이딩.
  - S19 메쉬렛과 S15 컬링 위에 구축.
- **목적**: G-Buffer 대역폭 없이 셰이딩. Nanite 계열 구조 이해.
- **기대**: 재질 수가 많을 때 재질 해석 비용, 도함수 재구성 오차.
- **결과**:
  - Visibility Buffer 경로.
  - 검증: Deferred(S23)와 이미지 일치.
  - 대역폭 / 타이밍 비교.
- **참고**: The Forge, Karis *Nanite*, [M]

## S36 · TAA & Temporal Upscaling 🆕
- **설명**:
  - Halton 지터, 모션 벡터 (스키닝 / 동적 오브젝트 포함).
  - 히스토리 재투영, 주변 클램핑.
  - 이후 오픈소스 FSR 통합 또는 자체 업스케일.
- **목적**: 현대 렌더러의 기본 AA / 업스케일. RT 디노이저(S30)와 같은 시간 누적 원리.
- **기대**: 고스팅 / 블러 / 디스오클루전 처리.
- **결과**:
  - TAA + 업스케일러.
  - 검증:
    - 정적 장면이 슈퍼샘플 레퍼런스에 수렴 (RMSE).
    - 움직이는 물체 고스팅 테스트.
- **참고**: Karis *High Quality Temporal Supersampling* (SIGGRAPH 2014), [M] Delt06 TAA

## S37 · Work Graphs 🆕
- **설명**: GPU가 스스로 작업을 생성·분배하는 노드 그래프 (SM 6.8). S15 컬링이나 S14 파티클을 Work Graph로 다시 작성.
- **목적**: ExecuteIndirect 이후의 GPU-driven 모델.
- **기대**: 중간 버퍼 / 왕복 Dispatch가 사라지는 효과, 레코드 크기와 성능.
- **결과**:
  - Work Graph 버전.
  - 검증: 기존 결과와 일치.
  - ExecuteIndirect 버전과 타이밍 비교.
- **참고**: [C] Work Graphs 샘플·GPUOpen Work Graphs

---

## 자료 목록
| 약어 | 자료 |
|---|---|
| A | Frank Luna, *Introduction to 3D Game Programming with DirectX 12* (2016) — [목차](https://www.degruyterbrill.com/document/doi/10.1515/9781683922902/html) |
| B | Jeremiah van Oosten, [*Learning DirectX 12* (3dgep)](https://www.3dgep.com/category/graphics-programming/directx/directx-12/) L1~L4 |
| C | Microsoft [DirectX-Graphics-Samples](https://github.com/microsoft/DirectX-Graphics-Samples) · [Working samples](https://learn.microsoft.com/en-us/windows/win32/direct3d12/working-samples) |
| C' | 〃 [D3D12Raytracing](https://raw.githubusercontent.com/microsoft/DirectX-Graphics-Samples/master/Samples/Desktop/D3D12Raytracing/readme.md) |
| D | Microsoft Learn [Direct3D 12 Programming Guide](https://learn.microsoft.com/en-us/windows/win32/direct3d12/directx-12-programming-guide) |
| E | Ohjurot [DX12-YT](https://github.com/Ohjurot/DX12-YT) (YouTube) |
| F | Alex Tardif [*Using D3D12 in 2022*](https://alextardif.com/DX12Tutorial.html) |
| G1 | megayuchi [D3D12 프로그래밍 기초편](https://www.inflearn.com/course/d3d12-%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D-%EA%B8%B0%EC%B4%88) (인프런) |
| G2 | megayuchi [D3D12 프로그래밍 기초 플러스](https://www.inflearn.com/course/d3d12-프로그래밍-기초-플러스) |
| G3 | megayuchi [DirectX Raytracing Programming](https://www.inflearn.com/course/directx-raytracing-programming) |
| G4 | megayuchi D3D Tiled Resources를 이용한 텍스처 스트리밍 ([DirectX12 강의 목록](https://www.inflearn.com/tag-curation/skill/directx12)) |
| H | Rookiss [MMORPG Part2: 게임 수학과 DirectX12](https://www.inflearn.com/course/%EC%96%B8%EB%A6%AC%EC%96%BC-3d-mmorpg-2?cid=326418) |
| I | 얌얌코딩 [게임 엔진 만들기 — DX12 전환 PART2](https://www.inflearn.com/course/게임-엔진-만들기-directx12) |
| J | NVIDIA [DX12 Raytracing Tutorial](https://developer.nvidia.com/rtx/raytracing/dxr/dx12-raytracing-tutorial-part-1) · Chris Wyman [Getting Started with RTX](https://github.com/asdlei99/gettingstartedwithrtxraytracing) |
| K | [vkguide.dev](https://vkguide.dev/) (Vulkan, 커리큘럼 구성 참고) |
| L | [GraphicsProgrammingRoadmap](https://github.com/j-2k/GraphicsProgrammingRoadmap) · [그래픽스 프로그래머 로드맵 2026](https://braindetox.kr/en/posts/graphics_programmer_learning_2026.html) |
| M | 공개 DX12 렌더러 [Helios](https://github.com/rtarun9/Helios) · [Delt06 dx12-renderer](https://github.com/Delt06/dx12-renderer) |
| 논문/발표 | O'Donnell *FrameGraph* (GDC 2017) · Olsson *Clustered Shading* (HPG 2012) · Karis *Nanite* (SIGGRAPH 2021) · Schied *SVGF* (HPG 2017) · Bitterli *ReSTIR* (SIGGRAPH 2020) · Majercik *DDGI* (JCGT 2019) · Karis *TAA* (SIGGRAPH 2014) · [PBR Book](https://pbr-book.org) |
