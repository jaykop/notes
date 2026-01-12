# 코드 작성 중 안티패턴 체크리스트

> **사용법**: 함수 작성 중이나 완료 직후 이 문서를 빠르게 훑어보며 자가 점검

---

## 📌 빠른 참조 (체크리스트)

```
함수 작성 완료 시 5초 체크:
□ bool 파라미터로 if/else 분기하지 않았나?
□ 파라미터가 3개 넘어가지 않나?
□ 함수가 여러 일을 하지 않나?
□ 멤버 변수 세팅하면서 동시에 반환하지 않나?
□ Out 파라미터와 반환값을 동시에 쓰지 않나?
□ Tick에서 무거운 연산을 하지 않나?
□ nullptr 체크 없이 포인터를 쓰지 않나?
□ 매직 넘버를 하드코딩하지 않나?
```

---

## 1. 함수 파라미터 안티패턴

### ❌ 안티패턴 1: Boolean Flag로 함수 로직 분기

**나쁜 예:**
```cpp
void UClimbingComponent::UpdateIK(bool bIsHand)
{
    if (bIsHand)
    {
        // 손 IK 로직 (50줄)
        FVector HandTarget = CalculateHandTarget();
        ApplyHandIK(HandTarget);
        // ...
    }
    else
    {
        // 발 IK 로직 (50줄)
        FVector FootTarget = CalculateFootTarget();
        ApplyFootIK(FootTarget);
        // ...
    }
}

// 호출부
UpdateIK(true);   // true가 뭘 의미하는지 불명확
UpdateIK(false);  // false가 손인지 발인지 알 수 없음
```

**왜 나쁜가:**
- 함수명만 보고 동작을 알 수 없음
- 호출부에서 true/false가 무엇을 의미하는지 불명확
- 함수가 실제로는 2개의 함수 역할을 함

**좋은 예:**
```cpp
// 방법 1: 함수 분리
void UClimbingComponent::UpdateHandIK()
{
    FVector HandTarget = CalculateHandTarget();
    ApplyHandIK(HandTarget);
}

void UClimbingComponent::UpdateFootIK()
{
    FVector FootTarget = CalculateFootTarget();
    ApplyFootIK(FootTarget);
}

// 방법 2: Enum 사용 (공통 로직이 많을 때)
enum class ELimbType : uint8
{
    Hand,
    Foot
};

void UClimbingComponent::UpdateLimbIK(ELimbType LimbType)
{
    switch (LimbType)
    {
        case ELimbType::Hand:
            // 손 로직
            break;
        case ELimbType::Foot:
            // 발 로직
            break;
    }
}

// 호출부가 명확함
UpdateLimbIK(ELimbType::Hand);
```

**언제 이 실수를 하나:**
"손/발 로직이 비슷하니까 하나로 합치자" → bool 하나로 구분하면 되겠다 → 나중에 구분이 어려워짐

---

### ❌ 안티패턴 2: 파라미터가 너무 많음

**나쁜 예:**
```cpp
void UClimbingComponent::PerformClimb(
    FVector StartPos, 
    FVector EndPos, 
    float Duration, 
    bool bUseEasing,
    float EasingStrength,
    bool bLockHands,
    bool bLockFeet,
    UAnimMontage* CustomMontage
)
{
    // 구현...
}

// 호출할 때 악몽
PerformClimb(Start, End, 1.0f, true, 0.5f, true, false, nullptr);
```

**왜 나쁜가:**
- 파라미터 순서를 기억하기 어려움
- 호출부에서 각 값이 무엇인지 불명확
- 나중에 파라미터 추가 시 모든 호출부 수정 필요

**좋은 예:**
```cpp
// 구조체로 묶기
USTRUCT(BlueprintType)
struct FClimbingParams
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector StartPosition = FVector::ZeroVector;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector EndPosition = FVector::ZeroVector;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    float Duration = 1.0f;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bLockHands = true;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bLockFeet = false;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    TObjectPtr<UAnimMontage> CustomMontage = nullptr;
};

void UClimbingComponent::PerformClimb(const FClimbingParams& Params)
{
    // Params.Duration 처럼 명확하게 접근
}

// 호출부가 명확함
FClimbingParams Params;
Params.StartPosition = Start;
Params.EndPosition = End;
Params.Duration = 1.0f;
PerformClimb(Params);
```

**경험 법칙:**
- 파라미터 3개 이하: OK
- 파라미터 4-5개: 구조체 고려
- 파라미터 6개 이상: 무조건 구조체

---

### ❌ 안티패턴 3: Out 파라미터와 반환값 혼용

**나쁜 예:**
```cpp
bool UClimbingComponent::FindClimbSurface(FVector& OutHitLocation, FVector& OutNormal)
{
    FHitResult HitResult;
    if (GetWorld()->LineTraceSingle(...))
    {
        OutHitLocation = HitResult.Location;
        OutNormal = HitResult.Normal;
        return true;
    }
    return false;
}

// 호출부
FVector HitLoc, Normal;
if (FindClimbSurface(HitLoc, Normal))  // Out 파라미터 + bool 반환
{
    // HitLoc과 Normal 사용
}
```

**왜 나쁜가:**
- Out 파라미터를 써야 하나, 반환값을 써야 하나 헷갈림
- Out 파라미터는 초기화 필요 (추가 코드)

**좋은 예:**
```cpp
// 방법 1: 구조체 반환 (추천)
USTRUCT()
struct FClimbSurfaceResult
{
    GENERATED_BODY()
    
    UPROPERTY()
    bool bSuccess = false;
    
    UPROPERTY()
    FVector HitLocation = FVector::ZeroVector;
    
    UPROPERTY()
    FVector SurfaceNormal = FVector::ZeroVector;
};

FClimbSurfaceResult UClimbingComponent::FindClimbSurface()
{
    FClimbSurfaceResult Result;
    
    FHitResult HitResult;
    if (GetWorld()->LineTraceSingle(...))
    {
        Result.bSuccess = true;
        Result.HitLocation = HitResult.Location;
        Result.SurfaceNormal = HitResult.Normal;
    }
    
    return Result;
}

// 호출부가 깔끔함
FClimbSurfaceResult Result = FindClimbSurface();
if (Result.bSuccess)
{
    UseLocation(Result.HitLocation);
}

// 방법 2: TOptional 사용 (UE5.0+)
TOptional<FVector> UClimbingComponent::FindClimbPoint()
{
    FHitResult HitResult;
    if (GetWorld()->LineTraceSingle(...))
    {
        return HitResult.Location;
    }
    return {};  // 실패
}

// 호출부
if (TOptional<FVector> Point = FindClimbPoint())
{
    UseLocation(*Point);
}
```

---

## 2. 함수 설계 안티패턴

### ❌ 안티패턴 4: 멤버 변수 세팅 + 값 반환 동시에

**나쁜 예:**
```cpp
class UClimbingComponent : public UActorComponent
{
private:
    UPROPERTY()
    FVector CachedClimbTarget;  // 멤버 변수로 캐싱
    
public:
    FVector CalculateClimbTarget()  // 근데 반환도 함
    {
        FVector Result = PerformComplexCalculation();
        CachedClimbTarget = Result;  // 멤버에 저장
        return Result;               // 동시에 반환
    }
};

// 호출부 - 헷갈림
FVector Target = CalculateClimbTarget();  // 이걸 써야 하나?
// 아니면
CalculateClimbTarget();  // 멤버 변수만 업데이트하고
FVector Target = CachedClimbTarget;  // 이렇게 읽어야 하나?
```

**왜 나쁜가:**
- 함수의 목적이 불명확 (계산? 캐싱? 둘 다?)
- 호출부에서 반환값을 써야 할지, 멤버 변수를 읽어야 할지 혼란
- 같은 값이 2곳(반환값, 멤버)에 존재

**좋은 예:**
```cpp
// 방법 1: 역할 분리 (추천)
class UClimbingComponent : public UActorComponent
{
private:
    UPROPERTY()
    FVector CachedClimbTarget;
    
public:
    // 계산만 담당 (멤버 건드리지 않음)
    FVector CalculateClimbTarget() const
    {
        return PerformComplexCalculation();
    }
    
    // 캐시 업데이트만 담당 (반환 안 함)
    void UpdateCachedClimbTarget()
    {
        CachedClimbTarget = CalculateClimbTarget();
    }
    
    // 캐시된 값 읽기만 담당
    FVector GetCachedClimbTarget() const
    {
        return CachedClimbTarget;
    }
};

// 호출부가 명확함
FVector Target = ClimbComp->CalculateClimbTarget();  // 즉시 계산 필요 시
// 또는
ClimbComp->UpdateCachedClimbTarget();  // 캐시 업데이트
FVector Target = ClimbComp->GetCachedClimbTarget();  // 캐시 읽기

// 방법 2: 캐시 필요 없으면 단순화
FVector CalculateClimbTarget() const  // 그냥 계산만
{
    return PerformComplexCalculation();
}
```

**언제 이 실수를 하나:**
"어차피 계산한 김에 저장도 하고 반환도 하자" → 나중에 "이 함수 호출하면 멤버 변수가 바뀌는 부작용"을 놓침

---

### ❌ 안티패턴 5: 함수가 여러 일을 동시에 함

**나쁜 예:**
```cpp
void UClimbingComponent::UpdateClimbing(float DeltaTime)
{
    // 1. 입력 처리
    FVector2D Input = GetPlayerInput();
    
    // 2. 물리 계산
    FVector Velocity = CalculateClimbVelocity(Input);
    
    // 3. 충돌 검사
    if (CheckWallCollision())
    {
        // 4. 애니메이션 업데이트
        UpdateClimbAnimation(Velocity);
        
        // 5. 사운드 재생
        PlayClimbingSound();
        
        // 6. 파티클 스폰
        SpawnClimbParticles();
        
        // 7. 캐릭터 이동
        MoveCharacter(Velocity * DeltaTime);
        
        // 8. UI 업데이트
        UpdateStaminaUI();
    }
    
    // 9. 디버그 드로우
    DrawDebugLines();
}
```

**왜 나쁜가:**
- 함수명 "UpdateClimbing"은 구체적으로 뭘 하는지 알 수 없음
- 버그 발생 시 어느 부분이 문제인지 찾기 어려움
- 테스트 불가능 (모든 시스템이 동시에 필요)
- 재사용 불가능

**좋은 예:**
```cpp
void UClimbingComponent::TickComponent(float DeltaTime, ...)
{
    Super::TickComponent(DeltaTime, ...);
    
    if (!bIsClimbing)
        return;
    
    UpdateClimbingLogic(DeltaTime);
}

void UClimbingComponent::UpdateClimbingLogic(float DeltaTime)
{
    // 각 단계를 명확한 함수로 분리
    const FVector2D Input = ProcessClimbingInput();
    const FVector Velocity = CalculateClimbVelocity(Input, DeltaTime);
    
    if (IsValidClimbingSurface())
    {
        ApplyClimbingMovement(Velocity, DeltaTime);
        UpdateClimbingVisuals(Velocity);
        UpdateClimbingAudio();
    }
}

// 각 역할이 명확한 함수들
FVector2D UClimbingComponent::ProcessClimbingInput()
{
    // 입력 처리만
}

FVector UClimbingComponent::CalculateClimbVelocity(const FVector2D& Input, float DeltaTime)
{
    // 물리 계산만
}

void UClimbingComponent::ApplyClimbingMovement(const FVector& Velocity, float DeltaTime)
{
    // 이동 적용만
}

void UClimbingComponent::UpdateClimbingVisuals(const FVector& Velocity)
{
    UpdateClimbAnimation(Velocity);
    SpawnClimbParticles();
}

void UClimbingComponent::UpdateClimbingAudio()
{
    PlayClimbingSound();
}
```

**경험 법칙:**
- 함수는 **하나의 추상화 레벨**만 다뤄야 함
- "그리고(and)"가 들어가면 분리 신호

---

### ❌ 안티패턴 6: 함수명과 실제 동작 불일치

**나쁜 예:**
```cpp
bool UClimbingComponent::CheckCanClimb()
{
    // 체크만 할 줄 알았는데...
    
    // 1. LineTrace 수행
    FHitResult Hit = PerformWallTrace();
    
    // 2. 캐시 업데이트 (부작용!)
    CachedWallNormal = Hit.Normal;
    
    // 3. StateTree 업데이트 (부작용!)
    StateTreeComponent->SetClimbableWall(Hit.GetActor());
    
    // 4. 애니메이션 준비 (부작용!)
    PrepareClimbAnimation();
    
    return Hit.bBlockingHit;
}
```

**왜 나쁜가:**
- "Check"는 검사만 할 것 같은데 실제로는 상태를 변경함
- 순수 함수인 것처럼 보이지만 부작용이 있음
- 같은 함수를 두 번 호출하면 예상치 못한 결과

**좋은 예:**
```cpp
// 방법 1: 역할별 분리
bool UClimbingComponent::CanClimb() const  // const로 부작용 없음을 보장
{
    FHitResult Hit;
    return PerformWallTrace(Hit) && IsValidClimbSurface(Hit);
}

void UClimbingComponent::PrepareClimbing()  // 준비 작업은 별도 함수
{
    FHitResult Hit;
    if (PerformWallTrace(Hit))
    {
        CachedWallNormal = Hit.Normal;
        StateTreeComponent->SetClimbableWall(Hit.GetActor());
        PrepareClimbAnimation();
    }
}

// 호출부
if (CanClimb())  // 순수 검사
{
    PrepareClimbing();  // 명시적 준비
    StartClimbing();
}

// 방법 2: 함수명을 정직하게
bool UClimbingComponent::CheckAndPrepareClimbing()  // 이름에 명시
{
    FHitResult Hit;
    if (PerformWallTrace(Hit))
    {
        CachedWallNormal = Hit.Normal;
        StateTreeComponent->SetClimbableWall(Hit.GetActor());
        PrepareClimbAnimation();
        return true;
    }
    return false;
}
```

**함수명 규칙:**
- `Check`, `Can`, `Is`, `Has` → 상태만 검사, 부작용 없음 (const)
- `Update` → 상태 변경
- `Calculate` → 계산만, 부작용 없음 (const)
- `Apply` → 계산 결과를 실제로 적용

---

## 3. 상태 관리 안티패턴

### ❌ 안티패턴 7: Boolean 플래그 남발

**나쁜 예:**
```cpp
class UClimbingComponent : public UActorComponent
{
private:
    bool bIsClimbing;
    bool bIsJumping;
    bool bIsHanging;
    bool bIsTransitioning;
    bool bCanJumpFromClimb;
    bool bIsClimbingUp;
    bool bIsClimbingDown;
    bool bIsClimbingLeft;
    bool bIsClimbingRight;
};

// 로직에서 조합 폭발
void UpdateState()
{
    if (bIsClimbing && !bIsJumping && !bIsHanging && !bIsTransitioning)
    {
        if (bIsClimbingUp && !bIsClimbingDown)
        {
            // 무슨 상태인지 알 수 없음
        }
    }
}
```

**왜 나쁜가:**
- 상태 조합이 기하급수적으로 증가
- 논리적으로 불가능한 상태 조합 발생 가능 (동시에 위/아래 이동)
- 디버깅 시 현재 상태 파악 불가능

**좋은 예:**
```cpp
// Enum으로 명확한 상태 정의
UENUM(BlueprintType)
enum class EClimbingState : uint8
{
    None,
    Climbing,
    Hanging,
    JumpingFromClimb,
    TransitionToClimb,
    TransitionFromClimb
};

UENUM(BlueprintType)
enum class EClimbingDirection : uint8
{
    None,
    Up,
    Down,
    Left,
    Right
};

class UClimbingComponent : public UActorComponent
{
private:
    UPROPERTY()
    EClimbingState CurrentState = EClimbingState::None;
    
    UPROPERTY()
    EClimbingDirection CurrentDirection = EClimbingDirection::None;
    
public:
    bool IsInClimbingState() const
    {
        return CurrentState == EClimbingState::Climbing 
            || CurrentState == EClimbingState::Hanging;
    }
    
    bool CanTransition() const
    {
        return CurrentState != EClimbingState::TransitionToClimb
            && CurrentState != EClimbingState::TransitionFromClimb;
    }
};

// 로직이 명확함
void UpdateState()
{
    switch (CurrentState)
    {
        case EClimbingState::Climbing:
            HandleClimbingState();
            break;
        case EClimbingState::Hanging:
            HandleHangingState();
            break;
        // ...
    }
}
```

**경험 법칙:**
- bool 플래그 3개 이상 → Enum 고려
- bool 플래그가 서로 배타적 → 무조건 Enum

---

### ❌ 안티패턴 8: 상태 전이 로직 분산

**나쁜 예:**
```cpp
// 여기저기 흩어진 상태 변경
void UClimbingComponent::OnJumpPressed()
{
    if (bIsClimbing)
    {
        bIsClimbing = false;
        bIsJumping = true;  // 상태 변경 1
    }
}

void UClimbingComponent::Tick(float DeltaTime)
{
    if (bIsJumping && IsGrounded())
    {
        bIsJumping = false;  // 상태 변경 2
    }
    
    if (!bIsClimbing && DetectWall())
    {
        bIsClimbing = true;  // 상태 변경 3
    }
}

void UClimbingComponent::OnAnimationFinished()
{
    if (bIsTransitioning)
    {
        bIsTransitioning = false;
        bIsClimbing = true;  // 상태 변경 4
    }
}
```

**왜 나쁜가:**
- 상태 변경이 어디서 일어나는지 추적 불가능
- "어떻게 이 상태가 됐지?" 디버깅 악몽
- 같은 상태 전이가 여러 곳에 중복

**좋은 예:**
```cpp
class UClimbingComponent : public UActorComponent
{
private:
    EClimbingState CurrentState;
    
    // 모든 상태 전이를 한 곳에서 관리
    void TransitionToState(EClimbingState NewState)
    {
        // 전이 가능 여부 검증
        if (!CanTransitionTo(NewState))
        {
            UE_LOG(LogTemp, Warning, TEXT("Invalid transition: %s -> %s"),
                *UEnum::GetValueAsString(CurrentState),
                *UEnum::GetValueAsString(NewState));
            return;
        }
        
        // 이전 상태 종료 로직
        OnExitState(CurrentState);
        
        // 상태 변경
        const EClimbingState OldState = CurrentState;
        CurrentState = NewState;
        
        // 새 상태 진입 로직
        OnEnterState(NewState);
        
        // 디버그 로그
        UE_LOG(LogTemp, Log, TEXT("State transition: %s -> %s"),
            *UEnum::GetValueAsString(OldState),
            *UEnum::GetValueAsString(NewState));
    }
    
    bool CanTransitionTo(EClimbingState NewState) const
    {
        // 상태 전이 규칙을 한 곳에 정의
        switch (CurrentState)
        {
            case EClimbingState::None:
                return NewState == EClimbingState::TransitionToClimb;
                
            case EClimbingState::Climbing:
                return NewState == EClimbingState::Hanging
                    || NewState == EClimbingState::JumpingFromClimb
                    || NewState == EClimbingState::TransitionFromClimb;
                
            // ...
        }
        return false;
    }
    
    void OnEnterState(EClimbingState State)
    {
        switch (State)
        {
            case EClimbingState::Climbing:
                StartClimbingAnimation();
                EnableClimbingInput();
                break;
            // ...
        }
    }
    
    void OnExitState(EClimbingState State)
    {
        switch (State)
        {
            case EClimbingState::Climbing:
                StopClimbingAnimation();
                DisableClimbingInput();
                break;
            // ...
        }
    }
    
public:
    void OnJumpPressed()
    {
        TransitionToState(EClimbingState::JumpingFromClimb);
    }
};
```

---

## 4. UE5 특화 안티패턴

### ❌ 안티패턴 9: Tick에서 무거운 연산

**나쁜 예:**
```cpp
void UClimbingComponent::TickComponent(float DeltaTime, ...)
{
    Super::TickComponent(DeltaTime, ...);
    
    // 매 프레임 FindComponentByClass (느림!)
    USkeletalMeshComponent* Mesh = GetOwner()->FindComponentByClass<USkeletalMeshComponent>();
    
    // 매 프레임 GetAllActorsOfClass (매우 느림!)
    TArray<AActor*> Walls;
    UGameplayStatics::GetAllActorsOfClass(GetWorld(), AWall::StaticClass(), Walls);
    
    // 매 프레임 복잡한 계산
    for (AActor* Wall : Walls)
    {
        // 거리 계산
        float Distance = FVector::Dist(GetOwner()->GetActorLocation(), Wall->GetActorLocation());
        // 방향 계산
        FVector Direction = (Wall->GetActorLocation() - GetOwner()->GetActorLocation()).GetSafeNormal();
        // 각도 계산
        float Angle = FMath::RadiansToDegrees(FMath::Acos(FVector::DotProduct(Direction, GetOwner()->GetActorForwardVector())));
    }
}
```

**왜 나쁜가:**
- FindComponentByClass는 매우 느림 (전체 컴포넌트 순회)
- GetAllActorsOfClass는 극도로 느림 (월드의 모든 액터 검색)
- 60fps 기준 0.0166초 내에 완료해야 하는데 위 코드는 몇 ms 소요

**좋은 예:**
```cpp
class UClimbingComponent : public UActorComponent
{
private:
    // 캐싱
    UPROPERTY()
    TObjectPtr<USkeletalMeshComponent> CachedMeshComponent;
    
    UPROPERTY()
    TArray<TObjectPtr<AActor>> NearbyWalls;
    
    float LastWallUpdateTime = 0.0f;
    const float WallUpdateInterval = 0.5f;  // 0.5초마다만 업데이트
    
protected:
    virtual void BeginPlay() override
    {
        Super::BeginPlay();
        
        // BeginPlay에서 한 번만 캐싱
        CachedMeshComponent = GetOwner()->FindComponentByClass<USkeletalMeshComponent>();
        ensure(CachedMeshComponent);
        
        // 초기 벽 탐색
        UpdateNearbyWalls();
    }
    
    virtual void TickComponent(float DeltaTime, ...) override
    {
        Super::TickComponent(DeltaTime, ...);
        
        // 일정 간격으로만 벽 업데이트
        LastWallUpdateTime += DeltaTime;
        if (LastWallUpdateTime >= WallUpdateInterval)
        {
            UpdateNearbyWalls();
            LastWallUpdateTime = 0.0f;
        }
        
        // 캐시된 데이터 사용
        if (CachedMeshComponent)
        {
            ProcessClimbingWithCachedData();
        }
    }
    
    void UpdateNearbyWalls()
    {
        // Overlap으로 가까운 벽만 찾기 (GetAllActors보다 훨씬 빠름)
        TArray<FOverlapResult> Overlaps;
        FCollisionShape Sphere = FCollisionShape::MakeSphere(500.0f);  // 5m 반경
        
        GetWorld()->OverlapMultiByChannel(
            Overlaps,
            GetOwner()->GetActorLocation(),
            FQuat::Identity,
            ECC_WorldStatic,
            Sphere
        );
        
        NearbyWalls.Empty();
        for (const FOverlapResult& Overlap : Overlaps)
        {
            if (Overlap.GetActor()->IsA<AWall>())
            {
                NearbyWalls.Add(Overlap.GetActor());
            }
        }
    }
};
```

**최적화 규칙:**
- **절대 Tick에서 하면 안 되는 것:**
  - `FindComponentByClass` / `GetComponentByClass`
  - `GetAllActorsOfClass` / `GetAllActorsOfClassWithTag`
  - `FindObject` / `LoadObject`
  
- **Tick에서 해도 되는 것:**
  - 캐시된 포인터 사용
  - 간단한 수학 계산 (더하기, 곱하기, 벡터 연산)
  - 조건문, switch

- **간격을 두고 해야 하는 것:**
  - LineTrace (0.1초마다)
  - Overlap 검사 (0.2-0.5초마다)
  - AI 경로 탐색 (0.5-1초마다)

---

### ❌ 안티패턴 10: nullptr 체크 누락

**나쁜 예:**
```cpp
void UClimbingComponent::UpdateAnimation()
{
    // nullptr 체크 없음
    USkeletalMeshComponent* Mesh = GetOwner()->FindComponentByClass<USkeletalMeshComponent>();
    UAnimInstance* AnimInst = Mesh->GetAnimInstance();  // Crash!
    
    Cast<UMyAnimInstance>(AnimInst)->SetClimbingSpeed(Speed);  // Crash!
}
```

**왜 나쁜가:**
- FindComponentByClass는 nullptr 반환 가능
- GetAnimInstance도 nullptr 반환 가능
- Cast 실패 시 nullptr 반환
- 런타임 크래시 발생

**좋은 예:**
```cpp
void UClimbingComponent::UpdateAnimation()
{
    // 방법 1: Early return 패턴
    USkeletalMeshComponent* Mesh = GetOwner()->FindComponentByClass<USkeletalMeshComponent>();
    if (!Mesh)
        return;
    
    UAnimInstance* AnimInst = Mesh->GetAnimInstance();
    if (!AnimInst)
        return;
    
    if (UMyAnimInstance* MyAnimInst = Cast<UMyAnimInstance>(AnimInst))
    {
        MyAnimInst->SetClimbingSpeed(Speed);
    }
    
    // 방법 2: ensure로 문제 조기 발견
    if (!ensure(CachedMeshComponent))  // 개발 중 경고
        return;
    
    if (UAnimInstance* AnimInst = CachedMeshComponent->GetAnimInstance())
    {
        if (UMyAnimInstance* MyAnimInst = Cast<UMyAnimInstance>(AnimInst))
        {
            MyAnimInst->SetClimbingSpeed(Speed);
        }
        else
        {
            ensureMsgf(false, TEXT("AnimInstance is not UMyAnimInstance type!"));
        }
    }
}
```

**nullptr 체크 규칙:**
- `GetOwner()` → 항상 체크 (Owner가 없을 수 있음)
- `GetWorld()` → BeginPlay 이후는 안전, 생성자에서는 체크
- `FindComponentByClass` → 항상 체크
- `Cast<T>` → 항상 체크
- `GetAnimInstance` → 항상 체크

---

### ❌ 안티패턴 11: UPROPERTY 없이 TObjectPtr 사용

**나쁜 예:**
```cpp
class UClimbingComponent : public UActorComponent
{
private:
    // UPROPERTY 없음 → GC가 수집할 수 있음!
    TObjectPtr<USkeletalMeshComponent> CachedMesh;
    
    TObjectPtr<UAnimInstance> CachedAnimInstance;
};

void UClimbingComponent::BeginPlay()
{
    CachedMesh = GetOwner()->FindComponentByClass<USkeletalMeshComponent>();
    CachedAnimInstance = CachedMesh->GetAnimInstance();
}

void UClimbingComponent::TickComponent(float DeltaTime, ...)
{
    // 어느 순간 CachedAnimInstance가 nullptr이 됨 (GC에 수집됨)
    if (CachedAnimInstance)  // 이미 늦음
    {
        CachedAnimInstance->SomeFunction();  // Crash 가능
    }
}
```

**왜 나쁜가:**
- UPROPERTY 없으면 UE의 GC가 참조를 추적하지 않음
- 랜덤 타이밍에 객체가 사라져서 디버깅 어려움
- "아까는 됐는데 지금은 안 돼요" 현상

**좋은 예:**
```cpp
class UClimbingComponent : public UActorComponent
{
private:
    // UPROPERTY로 GC가 추적하게 함
    UPROPERTY()
    TObjectPtr<USkeletalMeshComponent> CachedMesh;
    
    UPROPERTY()
    TObjectPtr<UAnimInstance> CachedAnimInstance;
    
    // 또는 TransientFlags로 저장 안 함
    UPROPERTY(Transient)
    TObjectPtr<AActor> TemporaryReference;
};
```

**규칙:**
- UObject 포인터는 **무조건 UPROPERTY**
- Actor/Component 참조 → UPROPERTY
- AnimInstance 참조 → UPROPERTY
- Widget 참조 → UPROPERTY

---

## 5. 코드 가독성 안티패턴

### ❌ 안티패턴 12: 매직 넘버

**나쁜 예:**
```cpp
void UClimbingComponent::UpdateHandIK()
{
    FVector Start = GetOwner()->GetActorLocation();
    FVector End = Start + GetOwner()->GetActorForwardVector() * 150.0f;  // 150이 뭐지?
    
    if (FVector::Dist(HandTarget, CurrentHand) < 5.0f)  // 5가 뭐지?
    {
        Speed *= 0.8f;  // 0.8이 뭐지?
    }
    
    if (Angle > 45.0f)  // 45가 뭐지?
    {
        return;
    }
}
```

**좋은 예:**
```cpp
class UClimbingComponent : public UActorComponent
{
private:
    // 상수로 정의
    static constexpr float HAND_REACH_DISTANCE = 150.0f;
    static constexpr float HAND_SNAP_THRESHOLD = 5.0f;
    static constexpr float CLIMB_SPEED_DAMPENING = 0.8f;
    static constexpr float MAX_CLIMB_ANGLE = 45.0f;
    
    // 또는 UPROPERTY로 노출 (디자이너가 조정 가능)
    UPROPERTY(EditAnywhere, Category = "Climbing|Hand IK")
    float HandReachDistance = 150.0f;
    
    UPROPERTY(EditAnywhere, Category = "Climbing|Hand IK")
    float HandSnapThreshold = 5.0f;
};

void UClimbingComponent::UpdateHandIK()
{
    FVector Start = GetOwner()->GetActorLocation();
    FVector End = Start + GetOwner()->GetActorForwardVector() * HandReachDistance;
    
    if (FVector::Dist(HandTarget, CurrentHand) < HandSnapThreshold)
    {
        Speed *= CLIMB_SPEED_DAMPENING;
    }
    
    if (Angle > MAX_CLIMB_ANGLE)
    {
        return;
    }
}
```

**규칙:**
- 숫자가 2번 이상 나오면 상수화
- 의미 있는 숫자는 모두 이름 붙이기
- 예외: 0, 1, -1은 문맥상 명확하면 OK

---

### ❌ 안티패턴 13: 불명확한 변수명

**나쁜 예:**
```cpp
void UClimbingComponent::Process()
{
    float t = 0.0f;
    FVector v = FVector::ZeroVector;
    bool b = false;
    
    for (int i = 0; i < arr.Num(); i++)
    {
        auto temp = arr[i];
        float val = Calculate(temp);
        if (val > 10.0f)
        {
            b = true;
            v = temp->GetLoc();
        }
    }
}
```

**좋은 예:**
```cpp
void UClimbingComponent::FindNearestClimbPoint()
{
    float NearestDistance = FLT_MAX;
    FVector NearestClimbPoint = FVector::ZeroVector;
    bool bFoundValidPoint = false;
    
    for (const FClimbPoint& ClimbPoint : AvailableClimbPoints)
    {
        const float DistanceToPoint = FVector::Dist(
            GetOwner()->GetActorLocation(),
            ClimbPoint.Location
        );
        
        if (DistanceToPoint < NearestDistance)
        {
            bFoundValidPoint = true;
            NearestDistance = DistanceToPoint;
            NearestClimbPoint = ClimbPoint.Location;
        }
    }
}
```

**네이밍 규칙:**
- bool → `bIs...`, `bHas...`, `bCan...`, `bShould...`
- Array/Container → 복수형 (`ClimbPoints`, `AvailableWalls`)
- 임시 변수 금지 → `Temp`, `Var`, `Val`, `Data`
- 축약 최소화 → `Pos` → `Position`, `Vel` → `Velocity`

---

### ❌ 안티패턴 14: 중첩 if문 과다

**나쁜 예:**
```cpp
void UClimbingComponent::TryClimb()
{
    if (bCanClimb)
    {
        if (HasEnoughStamina())
        {
            if (DetectWall())
            {
                if (IsValidClimbAngle())
                {
                    if (!IsPlayingAnimation())
                    {
                        if (CanAffordStaminaCost())
                        {
                            StartClimbing();
                        }
                    }
                }
            }
        }
    }
}
```

**좋은 예:**
```cpp
void UClimbingComponent::TryClimb()
{
    // Early return으로 평탄화
    if (!bCanClimb)
        return;
    
    if (!HasEnoughStamina())
        return;
    
    if (!DetectWall())
        return;
    
    if (!IsValidClimbAngle())
        return;
    
    if (IsPlayingAnimation())
        return;
    
    if (!CanAffordStaminaCost())
        return;
    
    StartClimbing();
}

// 또는 조건을 하나로 합치기
void UClimbingComponent::TryClimb()
{
    if (CanStartClimbing())
    {
        StartClimbing();
    }
}

bool UClimbingComponent::CanStartClimbing() const
{
    return bCanClimb
        && HasEnoughStamina()
        && DetectWall()
        && IsValidClimbAngle()
        && !IsPlayingAnimation()
        && CanAffordStaminaCost();
}
```

**규칙:**
- 중첩 3단계 이상 → Early return 또는 별도 함수
- 조건이 많으면 `Can...()` 함수로 추출

---

## 6. 퍼포먼스 안티패턴

### ❌ 안티패턴 15: 루프 안에서 동적 할당

**나쁜 예:**
```cpp
void UClimbingComponent::ProcessClimbPoints(float DeltaTime)
{
    for (int32 i = 0; i < 100; i++)
    {
        // 매 반복마다 TArray 생성 (힙 할당!)
        TArray<FVector> Points;
        Points.Add(FVector::ZeroVector);
        Points.Add(FVector::UpVector);
        
        // 매 반복마다 FString 생성 (힙 할당!)
        FString DebugMessage = FString::Printf(TEXT("Processing %d"), i);
        
        ProcessPoints(Points);
    }
}
```

**좋은 예:**
```cpp
void UClimbingComponent::ProcessClimbPoints(float DeltaTime)
{
    // 루프 밖에서 한 번만 할당
    TArray<FVector> Points;
    Points.Reserve(2);  // 예상 크기 미리 할당
    
    for (int32 i = 0; i < 100; i++)
    {
        Points.Reset();  // 메모리 유지하고 내용만 지우기
        Points.Add(FVector::ZeroVector);
        Points.Add(FVector::UpVector);
        
        ProcessPoints(Points);
    }
}

// 더 나은 방법: 멤버 변수로 재사용
class UClimbingComponent : public UActorComponent
{
private:
    TArray<FVector> WorkingPoints;  // 작업용 배열 재사용
    
public:
    void ProcessClimbPoints(float DeltaTime)
    {
        for (int32 i = 0; i < 100; i++)
        {
            WorkingPoints.Reset();
            WorkingPoints.Add(FVector::ZeroVector);
            WorkingPoints.Add(FVector::UpVector);
            ProcessPoints(WorkingPoints);
        }
    }
};
```

---

## 7. 빠른 자가 진단

### 함수 작성 후 5초 체크

```cpp
void YourFunction()  // ← 이 함수를 작성했다면...
{
    // 코드...
}

// 아래 질문들을 빠르게 확인:
```

**1분 체크리스트:**

| 항목 | 확인 방법 | 문제 시그널 |
|------|----------|------------|
| **함수명** | 함수명만 보고 동작 설명 가능? | "Update", "Process", "Handle" 같은 모호한 이름 |
| **파라미터** | bool 파라미터가 있나? | `MyFunction(true, false, true)` 같은 호출 |
| **반환값** | 멤버 변수 세팅 + 반환 둘 다? | `FVector Result = Calc(); Member = Result;` |
| **책임** | 한 가지 일만 하나? | 함수 설명에 "그리고" 들어감 |
| **부작용** | Check/Can 함수가 상태 변경? | `CanClimb()` 호출 후 멤버 변수 변경됨 |
| **길이** | 50줄 이하인가? | 스크롤 필요 |
| **중첩** | if문 3단계 이하인가? | `if { if { if {` |
| **매직넘버** | 숫자에 이름이 있나? | `* 150.0f`, `< 5.0f` |
| **nullptr** | 포인터 사용 전 체크? | `Ptr->Function()` 직접 호출 |

---

## 8. Rider 실시간 감지 설정

### TODO 패턴 등록

```
Settings → Editor → TODO
```

다음 패턴 추가:
- `\bREVIEW\b.*` → Review 탭에 표시
- `\bREFACTOR\b.*` → Refactor 탭에 표시
- `\bFIXME\b.*` → High Priority

### 커스텀 Inspection 규칙

```
Settings → Editor → Inspections → C++
```

다음 규칙 심각도 높이기:
- "Function is too long" → Error
- "Too many parameters" → Warning
- "Magic number" → Warning

---

## 마무리

### 이 문서를 언제 보나?

1. **함수 작성 완료 직후** (5초)
   - 빠른 체크리스트만 훑어보기
   
2. **코드 리뷰 전** (2-3분)
   - 관련 안티패턴 섹션 정독
   
3. **버그 수정 중** (1-2분)
   - "이 버그가 어느 안티패턴 때문인지?" 확인

### 외울 필요 없는 것들

이 문서는 **외우기 위한 게 아니라 참고하기 위한 것**입니다.
2-3주 반복하면 자연스럽게 몸에 배게 됩니다.

**우선순위:**
1. **안티패턴 1-6** (함수 설계) → 가장 자주 발생
2. **안티패턴 9-11** (UE5 특화) → 크래시 직결
3. **안티패턴 12-14** (가독성) → 유지보수성

나머지는 여유 있을 때 참고! 🚀
