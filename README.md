### 네크로맨서 | 2026.01. ~ 2026.04.

- 개요: Steamworks API 기반의 리슨 서버 협동 멀티플레이어 게임

<br/>


- 기술 스택: Unreal Engine(5.5.4), C++, Steamworks API

  
<br/>


- 상세 업무
    - Listen Server 기반 P2P 멀티플레이어 세션 및 WAN 네트워크 환경 구축
    - 관전 시스템 개발 및 호스트-클라이언트 간 상태 동기화
    - 네트워크 상태 동기화 패킷 최적화
 

<br/>
    

**ㅤ**

### 기술 설명

<blockquote>
📣

구현 기술에 대한 주된 메커니즘들을 설명합니다

[1. 글로벌 WAN 환경에서 Listen Server 기반 P2P 멀티플레이어 세션 생성](https://github.com/mw08081/necromancer/blob/main/README.md#1-%EA%B8%80%EB%A1%9C%EB%B2%8C-wan-%ED%99%98%EA%B2%BD%EC%97%90%EC%84%9C-listen-server-%EA%B8%B0%EB%B0%98-p2p-%EB%A9%80%ED%8B%B0%ED%94%8C%EB%A0%88%EC%9D%B4%EC%96%B4-%EC%84%B8%EC%85%98-%EC%83%9D%EC%84%B1)  

[2. 호스트-클라이언트 간 상태 동기화를 이용한 관전 시스템 개발](https://github.com/mw08081/necromancer/blob/main/README.md#2-%ED%98%B8%EC%8A%A4%ED%8A%B8-%ED%81%B4%EB%9D%BC%EC%9D%B4%EC%96%B8%ED%8A%B8-%EA%B0%84-%EC%83%81%ED%83%9C-%EB%8F%99%EA%B8%B0%ED%99%94%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EA%B4%80%EC%A0%84-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EA%B0%9C%EB%B0%9C)  

</blockquote>


**ㅤ**

**ㅤ**

#### 1. 글로벌 WAN 환경에서 Listen Server 기반 P2P 멀티플레이어 세션 생성

<blockquote>
🛠️

구현 목표

- 글로벌 WAN 환경에서 Listen Server 기반 P2P 멀티플레이어 세션 생성 및 스팀 UI를 활용한 친구 초대 기능 구현

**ㅤ**

구현 내용 

- Steam의 Session Interface를 활용하여 룸(Lobby) 생성, 참가, 파괴 등의 전체 세션 라이프사이클 구현

```cpp
// 세션 생성하는 함수
void ANecLobbyPlayerController::OnClickCreateSession()
{
	// 세션 생성 델리게이트 생성
	OnlineSessionInterface->AddOnCreateSessionCompleteDelegate_Handle(CreateSessionCompleteDelegate);
	TSharedPtr<FOnlineSessionSettings> SessionSettings = MakeShareable(new FOnlineSessionSettings());

	// 세션 생성 설정
	SessionSettings->bIsLANMatch = false;              // 글로벌 세션을 위한 설정
	SessionSettings->NumPublicConnections = 4;
	SessionSettings->bAllowJoinInProgress = true;
	SessionSettings->bAllowJoinViaPresence = true;
	SessionSettings->bShouldAdvertise = true;          // 세션 등록
	SessionSettings->bUsesPresence = true;
	SessionSettings->bUseLobbiesIfAvailable = true;    // 리슨서버를 위한 로비사용

  // 해당 플레이어 컨트롤러를 이용한 세션 생성
	const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
	OnlineSessionInterface->CreateSession(*LocalPlayer->GetPreferredUniqueNetId(), NAME_GameSession, *SessionSettings);
}
```

```cpp
// 생성된 세션으로 이동하는 함수
void ANecLobbyPlayerController::OnCreateSessionComplete(FName SessionName, bool bWasSuccessful)
{
  // 생성된 세션의 정보를 이용하여 ServerTravel(방장 이동)
	FString Address;
	if (OnlineSessionInterface->GetResolvedConnectString(SessionName, Address))

	FNamedOnlineSession* Session = OnlineSessionInterface->GetNamedSession(SessionName);
	if (Session)
	{
		GetWorld()->ServerTravel(TEXT("/Game/Necromancer/Maps/TestThirdPersonMap?listen"));
	}
}
```
<img width="616" height="321" alt="etner_ls" src="https://github.com/user-attachments/assets/7179849b-23dd-49cc-bbe3-01857bf72bfd" />


**ㅤ**

- 세션 생성 이후 Steam ExternalUIInterace를 활용하여 간편하게 친구 초대 로직을 구현

```cpp
void ANecLobbyPlayerController::OnClickInviteFriend()
{
	IOnlineSubsystem* Subsystem = IOnlineSubsystem::Get();
	if (Subsystem)
	{
		IOnlineExternalUIPtr ExternalUI = Subsystem->GetExternalUIInterface();
		if (ExternalUI.IsValid())
		{
			ExternalUI->ShowInviteUI(0, NAME_GameSession);
		}
	}
}
```

```cpp
void ANecLobbyPlayerController::OnInviteAccepted(bool bWasSuccessful, int32 LocalUserNum, TSharedPtr<const FUniqueNetId> UserId, const FOnlineSessionSearchResult& InviteResult)
{
		// 세션 참가
		OnlineSessionInterface->AddOnJoinSessionCompleteDelegate_Handle(JoinSessionCompleteDelegate);
		const ULocalPlayer* LocalPlayer = GetWorld()->GetFirstLocalPlayerFromController();
		if (LocalPlayer)
		{
			OnlineSessionInterface->JoinSession(*LocalPlayer->GetPreferredUniqueNetId(),  FName(TEXT("Necromancer")), InviteResult);
		}
	}
}
```
<img width="432" height="399" alt="image" src="https://github.com/user-attachments/assets/3163b24f-d5bb-4e6b-bcef-815032a57fe8" />

<img width="817" height="446" alt="move" src="https://github.com/user-attachments/assets/ceb17a9f-0eae-4e1a-b61b-5026519de857" />

</blockquote>

**ㅤ**

#### 2. 호스트-클라이언트 간 상태 동기화를 이용한 관전 시스템 개발

<blockquote>
🛠️

구현 목표

- 게임 중 사망하거나 나중에 난입한 유저가 활성 플레이어들의 시점을 부드럽게 전환하며 실시간으로 게임을 관전하는 기능

**ㅤ**

구현 내용 

- 관전에 필요한 플레이어 카메라 회전값을 주기적으로 복제함

```cpp
void ANecPlayerCharacter::Tick(float DeltaTime)
{
		Super::Tick(DeltaTime);
		ReplicateRemoteViewRot();
}

void ANecPlayerCharacter::ReplicateRemoteViewRot()
{
		if (HasAuthority())
		{
			RemoteViewRot = GetControlRotation();
		}
}
```

- 플레이어 사망 시, Server RPC를 통해 관전 타겟을 요청함

```cpp
void ANecGameMode::Server_ReqeustSpectatingTarget_Implementation(ANecPlayerController* RequestPC, AActor* CurSpectatingTarget, bool bIsUp)
{
		// 생존 플레이어 중에서 임의의 플레이어를 지정
    int32 CurIdx = -1;
    for (int32 i = 0; i < PlayerControllers.Num(); ++i)
    {
        if (PlayerControllers[i] && PlayerControllers[i]->GetPawn() == CurSpectatingTarget)
        {
            CurIdx = i;
            break;
        }
    }
    
    // 플레이어 탐색 순회 과정(생략)

		// Client RPC를 통해 사망한 플레이어에게 관전 타겟을 넘겨줌
    RequestPC->Client_HandleCameraTarget(PlayerControllers[0]->GetPawn());

}

```

```cpp
// 서버로 부터 받은 관전 타겟을 이용하여 주기적으로 해당 타겟의 팔로잉함
FRotator TargetRot = TargetCharacter->RemoteViewRot;

// 이때 관전자가 호스트라면 곧바로 해당 액터의 회전값을 사용
if (HasAuthority())
{
	TargetRot = TargetCharacter->GetControlRotation();
}
// 관전자가 클라이언트라면 Replicated된 회전 값을 이용해서 사용
else
{
	TargetRot = TargetCharacter->RemoteViewRot;
}
```

**ㅤ**

결과

<img width="835" height="482" alt="rhkswjs2" src="https://github.com/user-attachments/assets/241330f7-e47c-4b73-a534-e9a4802a2ad3" />
사망 후 생존한 플레이어를 찾아 관전 모드 진입

<img width="660" height="237" alt="rhkswjs3" src="https://github.com/user-attachments/assets/29184001-d8a2-4847-8939-f3e8e7eb162c" />

관전자의 카메라 시점까지 복제하여 동기화

**ㅤ**

</blockquote>

**ㅤ**

**ㅤ**

**ㅤ**

### 트러블 슈팅

<blockquote>
📣

주요 트러블 슈팅에 관해서 설명합니다.

[1. 클라이언트 카메라 회전값 복제(Replication) 누락 문제 해결](https://github.com/mw08081/necromancer/blob/main/README.md#1-%ED%81%B4%EB%9D%BC%EC%9D%B4%EC%96%B8%ED%8A%B8-%EC%B9%B4%EB%A9%94%EB%9D%BC-%ED%9A%8C%EC%A0%84%EA%B0%92-%EB%B3%B5%EC%A0%9Creplication-%EB%88%84%EB%9D%BD-%EB%AC%B8%EC%A0%9C-%ED%95%B4%EA%B2%B0)   

[2. 다수 인원 참여 시 과도한 네트워크 RPC 및 패킷 폭증에 따른 랙(Lag) 개선](https://github.com/mw08081/necromancer/blob/main/README.md#2-%EB%8B%A4%EC%88%98-%EC%9D%B8%EC%9B%90-%EC%B0%B8%EC%97%AC-%EC%8B%9C-%EA%B3%BC%EB%8F%84%ED%95%9C-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-rpc-%EB%B0%8F-%ED%8C%A8%ED%82%B7-%ED%8F%AD%EC%A6%9D%EC%97%90-%EB%94%B0%EB%A5%B8-%EB%9E%99lag-%EA%B0%9C%EC%84%A0)  

</blockquote>


**ㅤ**

**ㅤ**

#### 1. 클라이언트 카메라 회전값 복제(Replication) 누락 문제 해결

<blockquote>
🛠️

문제 상황

- 관전 시스템 개발 도중 다른 플레이어의 위치(Location) 및 이동은 정상적으로 복제되었으나, 카메라 회전값(Control Rotation/Pitch, Yaw)이 복제되지 않는 현상 발생
- 이로 인해 다른 플레이어가 실제로 보는 시점이 아닌 일정한 방향만 바라보게 되는 현상이 발생

**ㅤ**

원인 분석

- 엔진 내부 코드 분석: 언리얼 엔진의 `ACharacter` 구조 분석 결과, 기본 이동(`FRepMovement`)은 엔진 차원에서 자동으로 복제 데이터 전송 패킷에 포함되지만, `ControlRotation`은 클라이언트의 성능 최적화를 위해 기본적으로 네트워크 복제 대상(`Replicated`)이 아님을 확인

```cpp
// Unreal Engine Core Source (APawn.cpp / ACharacter.cpp 개념 구조)
// Pawns/Characters translate movement, but ControlRotation is not replicated by default to save bandwidth.
void APawn::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);

    // ReplicatedMovement는 기본적으로 포함되나, ControlRotation(카메라 시점)은 복제 목록에 없음
    DOREPLIFETIME(APawn, ReplicatedMovement); 
}
```

**ㅤ**

해결방법

- 커스텀 복제 변수 선언: 카메라 회전값을 전송하기 위한 별도의 Struct/Rotator 변수를 추가하고 네트워크 복제 대상으로 등록

```cpp
// NecPlayerCharacter.h
UPROPERTY(Replicated)
FRotator RemoteViewRot;
```

```cpp
// NecPlayerCharacter.cpp
void ANecPlayerCharacter::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    
    // 카메라 회전값 변수를 복제 목록에 추가
    DOREPLIFETIME(ANecPlayerCharacter, RemoteViewRot);
}

void ANecPlayerCharacter::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    // 매 프레임 서버 권한을 확인하여 ControlRotation 복제 수행
    ReplicateRemoteViewRot();
}

void ANecPlayerCharacter::ReplicateRemoteViewRot()
{
    if (HasAuthority())
    {
        RemoteViewRot = GetControlRotation();
    }
}
```

**ㅤ**

결과

- 사망 후 관전자(Spectator)에게 관전 타겟 플레이어의 시점(Pitch/Yaw)을 실시간 동기화함
- 언리얼 엔진이 대역폭 절감을 위해 기본적으로 복제하는 데이터와 그렇지 않은 데이터(Control Rotation 등)를 구분하는 기준을 파악함
</blockquote>

**ㅤ**

#### 2. 다수 인원 참여 시 과도한 네트워크 RPC 및 패킷 폭증에 따른 랙(Lag) 개선

<blockquote>
🛠️

문제 상황

- 멀티플레이어 세션에 참여 인원이 늘어날수록 호스트(Listen Server) 및 클라이언트 환경 전체에서 네트워크 랙(Ping 튐, 데이터 수신 지연) 및 프레임 드랍 발생

**ㅤ**

원인 분석

- 모든 클라이언트가 매 틱(Tick)마다 관전에 필요한 마우스 회전값(Control Rotation)을 서버로 전송하면서 네트워크 대역폭 과부하가 심화되는 것으로 추정
- 호스트의 경우 수신된 패킷을 처리한 뒤, 다시 다른 모든 클라이언트에게 `Replication`으로 전파하는 과정에서 네트워크 I/O 및 CPU 병목이 발생함

**ㅤ**

해결방법

- 관전자 유무에 따른 조건부 복제 (Conditional Replication)
    
    카메라 회전값 복제의 주 목적이 '관전(Spectating)'임을 착안하여, 세션 내 사망자(관전자)가 0명일 때는 카메라 회전값 복제 연산을 아예 스킵하도록 제어 로직 구현
    
- ServerRPC 전송 주기 조절
    
    네트워크 전송 주기(Timer)를 설정하거나 이전 회전값 대비 일정 이상(Threshold) 변동이 발생했을 때만 전송하도록 개선하여 패킷 수 감소
    

```cpp
// Tick() 에서 BeginPlay() 로 이동함으로써, 타이머를 통한 반복 실행 예약
void ANecPlayerCharacter::BeginPlay()
{
    Super::BeginPlay();

    // BeginPlay 시점에 반복 타이머 설정 (0.1초 간격으로 무한 반복)
    GetWorldTimerManager().SetTimer(
        TimerHandle_ReplicateViewRot,
        this,
        &ANecPlayerCharacter::CheckAndReplicateRemoteViewRot,
        ReplicateInterval,
        true
    );
}
```

```cpp
void ANecPlayerCharacter::CheckAndReplicateRemoteViewRot()
{
    // 세션 내 사망자(관전자)가 존재할 때만 복제 로직 수행
    if (HasSpectatorsInSession())
    {
        ReplicateRemoteViewRot();
    }
}

void ANecPlayerCharacter::ReplicateRemoteViewRot()
{
    if (HasAuthority())
    {
        RemoteViewRot = GetControlRotation();
    }
    else if (IsLocallyControlled())
    {
        // 클라이언트: 관전자가 있고 + 일정 회전 각도(Threshold) 이상 변동 시에만 ServerRPC 호출
        FRotator CurrentRot = GetControlRotation();
        if (FMath::Abs((CurrentRot - LastSentRotation).Size()) > RotationThreshold)
        {
            Server_UpdateRemoteViewRot(CurrentRot);
            LastSentRotation = CurrentRot;
        }
    }
}
```

**ㅤ**

결과

- 매 틱(Tick) 수행되던 RPC 및 복제 연산을 타이머 기반 주기적 전송(Timer)과 각도 변화량(Threshold) 검사로 전환하여, 초당 패킷 전송량을 대폭 감소시키고 호스트의 네트워크 I/O 병목 및 랙 현상을 일부 해결함
- 단순히 모든 상태를 실시간 복제하는 것이 아니라, "이 데이터가 현재 진짜 필요한 상황인가(관전자 유무)", "얼마나 자주, 어떤 조건에서 보내야 하는가(주기/변화량)"를 고민하며 최적화 설계 감각을 습득함
</blockquote>

<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

[발표자료](https://docs.google.com/presentation/d/1tmq24hBc5ymyBy50EJNEuVfvLeIoLY2TjuM0ZqjODSg/edit?slide=id.g3dcd956c7b3_6_65#slide=id.g3dcd956c7b3_6_65)
