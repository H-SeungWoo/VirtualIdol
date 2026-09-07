# Stage On · 실시간 가상 공연장

**버추얼 공연자와 관객이 아바타로 만나 음원·이모지·피버로 상호작용하는 Unreal Engine 5 기반 공연 플랫폼입니다.**

[▶ 프로젝트 시연 영상](https://www.youtube.com/watch?v=JKt8BfGZm_k) · [▶ 성과공유회 영상 · 2분 39초부터](https://youtu.be/7AOZLXVn0M4?t=159)

| 항목 | 내용 |
|---|---|
| 개발 기간 | 2024.10.07–2024.12.19 · 메타버스 아카데미 최종 프로젝트 |
| 팀 규모 | 7명 · 기획 1 / 클라이언트 개발 3 / TA 1 / 백엔드 1 / AI 1 |
| 담당 | **한승우 · 클라이언트 개발** — 관객 상호작용, 공연 상태 처리, 멀티클라이언트 음원 재생, UMG UI |
| 성과 | 메타버스 아카데미 3기 최종성과공유회 **대상 · 과학기술정보통신부 장관상** |
| 핵심 기술 | UE 5.4, C++, Blueprint, UMG, Listen Server, RPC, HTTP/JSON, MediaPlayer |

## 먼저 볼 구현 3가지

| 구현 | 해결한 일 | 코드 바로가기 |
|---|---|---|
| **음원 전달·재생** | 음원 객체 전달을 HTTP 파일 다운로드와 로컬 MediaPlayer 재생으로 전환 | [목록 요청·다운로드](Source/VirtualIdol/Private/KMK/HttpActor_KMK.cpp#L1000) → [재생 액터](Source/VirtualIdol/Private/HSW_AudioLoadingActor.cpp#L28) |
| **공연 진행·관객 반응** | Listen Server에서 공연 카운트다운과 관객 반응을 각 참여자에게 전달 | [GameMode](Source/VirtualIdol/Private/HSW/HSW_AuditoriumGameMode.cpp) → [GameState](Source/VirtualIdol/Private/HSW/HSW_GameState_Auditorium.cpp#L88) |
| **공연자 기능 분리** | 캐릭터에 결합된 공연자 기능을 Actor Component로 분리해 모션 연동 플러그인의 Pawn에 적용 | [컴포넌트 선언](Source/VirtualIdol/Public/KMK/Virtual_KMK.h#L11) · [구현](Source/VirtualIdol/Private/KMK/Virtual_KMK.cpp) |

아래는 팀 프로젝트에서 한승우가 담당한 기능입니다. 공용 파일에도 작성한 함수가 있으며, 파일 전체와 개별 함수의 기여 범위를 구분합니다.

## 1. 음원 전달·재생 — 파일 준비와 재생 시작의 분리

### 문제

공연자가 준비한 WAV 파일을 런타임 음원 객체로 생성해 Multicast RPC로 전달하려 했으나, 다른 클라이언트에서는 재생되지 않았습니다. 로그와 객체 복제 방식을 조사하며 **객체 참조를 넘기는 것과 수신 클라이언트에 실제 음원 데이터를 준비하는 것은 별개**라는 점을 파악했습니다.

### 구현

백엔드 개발자와 공연별 음원 조회·다운로드 규격을 협의했습니다. 공연 ID에 대해 곡 ID·제목·다운로드 URL을 JSON 배열로 받고, **음원 파일 준비와 재생 시작 전달을 분리**했습니다.

```text
[음원 파일 준비]
공연 ID → HTTP/JSON 조회 → WAV 다운로드 → Saved/Music 저장

[재생 시작]
로컬 음원 목록 → 곡 선택 → 재생 액터 생성
                                ↓
                   Multicast RPC로 재생 시작 전달
                                ↓
                   MediaPlayer로 로컬 파일 재생
```

`SetWavFiles`가 저장된 파일을 곡 목록으로 정리하고, `CreateAudioActor`가 재생 액터를 생성합니다. 재생 액터는 파일 경로를 복제하고 Multicast RPC로 재생 시작을 전달합니다.

### 핵심 코드

아래 코드는 실제 구현에서 로그를 생략하고 서식을 정리한 발췌입니다.

**음원 준비 — 다운로드 응답의 바이트를 파일로 저장합니다.**

```cpp
// OnReqMusic의 WAV 다운로드 완료 콜백 내부
if (bWasWavSuccessful && WavResponse.IsValid() && WavResponse->GetResponseCode() == 200)
{
    FString FileName = FString::Printf(
        TEXT("%d_%d_%s.wav"), ConcertId, SongId, *Title);
    SaveWavToFile(FileName, WavResponse->GetContent());
}
```

[원본 코드 · OnReqMusic / SaveWavToFile](Source/VirtualIdol/Private/KMK/HttpActor_KMK.cpp#L1085)

**재생 — Multicast RPC 함수에서 로컬 파일을 열어 재생합니다.** MediaPlayer 생성 이후의 처리입니다.

```cpp
// MultiRPC_PlayWavFile_Implementation 내부
if (MediaPlayer && MediaPlayer->OpenFile(SongFilePath))
{
    if (MediaSoundComp)
    {
        MediaSoundComp->SetMediaPlayer(MediaPlayer);
        MediaPlayer->Play();
    }
}
```

[원본 코드 · MultiRPC_PlayWavFile_Implementation](Source/VirtualIdol/Private/HSW_AudioLoadingActor.cpp#L69)

**결과:** 프로젝트 개발 당시 멀티플레이 환경에서 여러 클라이언트의 음원 재생을 확인했습니다. 지원하는 WAV 음원을 서버에 추가하면 음원 때문에 클라이언트를 다시 빌드·배포할 필요가 없도록 구성했습니다.

이 구현은 재생 시작을 전달하는 구조입니다. 모든 클라이언트의 다운로드 완료를 기다리는 준비 확인이나 재생 위치의 시각 보정은 포함하지 않습니다.

## 2. 공연 진행·관객 반응

- Listen Server 환경에서 GameMode와 GameState를 이용해 공연 카운트다운을 공연자·관객의 표시 경로로 전달했습니다.
- 관객 캐릭터의 이모지와 피버 관련 입력을 RPC로 전달하고, UMG UI와 효과에 연결했습니다.
- 피버 기여도를 Dynamic Material의 밝기 파라미터에 연결해 관객이 자신의 참여를 시각적으로 확인하도록 구현했습니다.
- FSM 기반 관객 NPC와 C++·Blueprint 기반 상호작용을 구현했습니다.

**공연 진행 — GameMode가 GameState의 카운트다운 RPC를 호출합니다.**

```cpp
void AHSW_AuditoriumGameMode::BroadcastCountDown()
{
    AHSW_GameState_Auditorium* gs = GetGameState<AHSW_GameState_Auditorium>();
    if (gs)
    {
        gs->MultiRPC_ShowCountDown();
    }
}
```

[원본 코드 · BroadcastCountDown](Source/VirtualIdol/Private/HSW/HSW_AuditoriumGameMode.cpp#L95) · [수신 처리 · GameState](Source/VirtualIdol/Private/HSW/HSW_GameState_Auditorium.cpp#L97)

**관객 반응 — 피버 입력으로 변경된 밝기 값을 머티리얼의 발광 파라미터에 반영합니다.** 로그를 생략한 실제 함수입니다.

```cpp
void AHSW_ThirdPersonCharacter::MulticastRPCBrightness_Implementation(int index)
{
    FeverDynamicMat->SetScalarParameterValue(
        TEXT("jswEmissivePower-A"), FeverBright);
}
```

[원본 코드 · 밝기 반영](Source/VirtualIdol/Private/HSW/HSW_ThirdPersonCharacter.cpp#L682) · [입력과 밝기 값 변경](Source/VirtualIdol/Private/HSW/HSW_ThirdPersonCharacter.cpp#L589)

## 3. 공연자 기능 분리

기존 캐릭터 클래스에 결합된 공연자 기능을 Actor Component로 분리해, 모션 연동 플러그인의 Pawn에도 적용할 수 있도록 리팩터링했습니다. 저장소의 `UVirtual_KMK`는 `UActorComponent`를 상속하며 공연자 기능과 음원 목록·재생 요청을 포함합니다.

**공연자 컴포넌트는 소유 Actor에서 필요한 메시를 찾아 연결합니다.** 아래는 `BeginPlay`의 시작 부분으로, 이후 초기화 코드는 생략했습니다.

```cpp
void UVirtual_KMK::BeginPlay()
{
    Super::BeginPlay();
    meshComp = GetOwner()->FindComponentByTag<USkeletalMeshComponent>(FName(TEXT("Mesh")));
    // 이후 초기화 코드 생략
}
```

[원본 코드 · 소유 Actor의 메시 연결](Source/VirtualIdol/Private/KMK/Virtual_KMK.cpp#L39) · [UActorComponent 상속 선언](Source/VirtualIdol/Public/KMK/Virtual_KMK.h#L11)

실제 Pawn 부착 설정은 Blueprint 에셋에도 포함됩니다.

## 담당 역할과 코드 안내

| 기능 | 살펴볼 파일·함수 |
|---|---|
| 음원 API 연동 | [HttpActor_KMK.cpp](Source/VirtualIdol/Private/KMK/HttpActor_KMK.cpp#L1000) — `ReqMusic`, `OnReqMusic`, `SaveWavToFile` |
| 저장 음원 목록·재생 요청 | [Virtual_KMK.cpp](Source/VirtualIdol/Private/KMK/Virtual_KMK.cpp#L316) — `SetWavFiles`, `CreateAudioActor` |
| 음원 파일 재생 | [HSW_AudioLoadingActor.cpp](Source/VirtualIdol/Private/HSW_AudioLoadingActor.cpp#L64) — `ServerRPC_PlayWaveFile`, `MultiRPC_PlayWavFile` |
| 곡 선택 UI | [HSW_SongUnit.cpp](Source/VirtualIdol/Private/HSW/HSW_SongUnit.cpp) |
| 이모지·피버·밝기 연동 | [HSW_ThirdPersonCharacter.cpp](Source/VirtualIdol/Private/HSW/HSW_ThirdPersonCharacter.cpp#L517) — 이모지 RPC, `OnMyFeverGauge`, 피버·밝기 갱신 |
| 관객 효과음·현장음 | [HSW_AudioActor.cpp](Source/VirtualIdol/Private/HSW/HSW_AudioActor.cpp) — AudioComponent 기반 재생·볼륨 제어 |

`HttpActor_KMK.cpp`의 위 3개 함수와 `Virtual_KMK.cpp`의 위 2개 함수는 한승우가 작성했습니다. `HSW_AudioActor`는 관객 효과음용이며, 다운로드한 WAV의 재생 경로는 `HSW_AudioLoadingActor`에서 확인할 수 있습니다.

## 저장소 구조

```text
VirtualIdol.uproject                # Unreal Engine 5.4 프로젝트
Source/VirtualIdol/
  Private/HSW/                     # 관객 캐릭터·UI·공연 진행·효과음
  Private/HSW_AudioLoadingActor.cpp # 로컬 WAV 재생 액터
  Private/KMK/                     # HTTP·공연자 등 팀 공동 작업 파일
  Public/                          # 헤더·RPC·컴포넌트 선언
Content/                           # 맵·Blueprint·UI·모델 등 에셋
Config/                            # 엔진·입력·온라인 설정
Plugins/                           # 프로젝트 플러그인
```

## 실행 환경과 확인 방법

- 프로젝트 기준 버전: **Unreal Engine 5.4** (`VirtualIdol.uproject` 기준)
- Windows C++ 빌드 환경과 프로젝트에서 활성화한 플러그인이 필요합니다. VRM4U, ReMoCapp, OnlineSubsystemSteam, LiveLink 등이 사용됩니다.
- 전체 저장소와 에셋을 준비한 뒤 `VirtualIdol.uproject`에서 프로젝트 파일을 생성하고, `VirtualIdolEditor`를 빌드해 에디터에서 엽니다. 플러그인의 설치·버전·이용 조건은 각 배포처 기준을 확인해야 합니다.
- 로그인·공연 조회·음원 다운로드·온라인 세션은 당시 백엔드 및 서비스 설정에 의존합니다. 현재 외부 서버의 가용성과 새 환경에서의 전체 실행은 확인되지 않았으므로, **동작 확인은 상단 시연 영상과 아래 발표 자료를 먼저 참고해 주세요.**

## 팀 발표 자료

<details>
<summary>발표 슬라이드 22장 펼쳐보기</summary>

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_1](https://github.com/user-attachments/assets/132f6e1e-225c-4e7d-a6a9-bc4da91977b8)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_2](https://github.com/user-attachments/assets/12d4f4a4-1956-4deb-8050-6debedde69be)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_3](https://github.com/user-attachments/assets/f041a8be-3c25-413c-a12b-ab1280252e79)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_4](https://github.com/user-attachments/assets/c01c61c2-e1a9-4f78-bd45-5a7d08381323)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_5](https://github.com/user-attachments/assets/adb46326-eed5-46af-b050-4d0466d283ed)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_6](https://github.com/user-attachments/assets/923a7edb-4f03-4e2d-bd99-b6c4e17a12d6)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_7](https://github.com/user-attachments/assets/35e35821-8611-4133-96f5-3bd284e5a61b)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_8](https://github.com/user-attachments/assets/a12b437c-936b-42ed-9e33-9a182c725a46)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_9](https://github.com/user-attachments/assets/49945add-35f5-432a-9af5-521b585c7497)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_10](https://github.com/user-attachments/assets/c1ffcfa9-1ebf-4be1-ae5b-b69a853de422)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_11](https://github.com/user-attachments/assets/e58ba677-7c72-40ce-9fd8-64e71bf0848a)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_12](https://github.com/user-attachments/assets/dae510e2-bd8f-4ad9-ac5d-e90ce84a7db9)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_13](https://github.com/user-attachments/assets/9a001570-b459-4e38-9ac0-082ccd999964)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_14](https://github.com/user-attachments/assets/06b410be-0066-41fe-8c6d-3dab1e65d992)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_15](https://github.com/user-attachments/assets/973d636a-3455-4221-91dc-3aed04b0a2b4)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_16](https://github.com/user-attachments/assets/e9fa4975-37b0-4b17-bc1a-1c6a471fd7bf)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_17](https://github.com/user-attachments/assets/f263df7a-1fd9-4304-89b9-a377e6895bc4)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_18](https://github.com/user-attachments/assets/0bcea3c4-ae86-49ef-93b8-f877dd2fa5c0)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_19](https://github.com/user-attachments/assets/b7a20c29-e3ed-4a59-8b6e-338defeb722c)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_20](https://github.com/user-attachments/assets/acb0d94b-4c24-4af8-95d1-02ff4aadb21a)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_21](https://github.com/user-attachments/assets/4f70a5c5-8587-4895-96d8-a740f49e05c7)

![1-28_차원의 저편에서 만난 최애와의 잊을 수 없는 무대입니다만 _베타발표PDF_22](https://github.com/user-attachments/assets/d9fbb95c-55a1-4642-85bb-8fa00b00dbf6)

</details>
