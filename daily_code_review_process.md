# 코드 품질 개선 - 일일 프로세스 가이드

> **목표**: 클린 코드 원칙을 알고 있지만 적용하지 못하는 문제를 습관화된 프로세스로 해결

---

## 📋 일일 루틴 개요

```
코드 작성 전 (5-10분)
  ↓ [1단계: AI 설계 브리핑]
  
코드 작성
  ↓ [실제 구현]
  
코드 작성 후 (10-15분)
  ↓ [2단계: AI 코드 리뷰]
  
퇴근 전 (30분)
  ↓ [3단계: 셀프 코드 리뷰]
```

**총 추가 시간**: 하루 45-55분 (핵심 기능에만 적용 시)

---

## 1단계: 설계 브리핑 (코드 작성 전)

### 목적
- 구현 전 설계 검증
- 아키텍처 문제 조기 발견
- UE5 시스템과의 충돌 가능성 확인

### 템플릿

```markdown
## [기능명] 설계 브리핑

### 컨텍스트
- 프로젝트: [프로젝트명/장르]
- 관련 시스템: [연관된 Component/System들]
- 기존 아키텍처: [Component 기반 / StateTree / 등]

### 구현 목표
[무엇을 만들려고 하는지 1-2문장으로]

### 설계 의도
1. **구조**
   - [어떤 클래스/컴포넌트를 만들 것인지]
   - [상속/조합 구조]

2. **데이터 흐름**
   - [Input → Processing → Output 흐름도]

3. **주요 클래스/함수**
   - `FunctionName()`: [역할 설명]
   - `AnotherFunction()`: [역할 설명]

### 고민 지점
- [ ] [기술적 불확실성]
- [ ] [성능 우려사항]
- [ ] [다른 시스템과의 충돌 가능성]

### AI에게 질문
1. 이 설계에서 개선할 점이나 놓친 부분이 있을까?
2. UE5 아키텍처 관점에서 문제될 만한 부분은?
3. [구체적인 기술 질문]
```

### 실전 예시

```markdown
## Climbing Hand/Foot IK 시스템 설계 브리핑

### 컨텍스트
- 프로젝트: 3인칭 액션 어드벤처 게임
- 관련 시스템: ClimbingComponent, AnimInstance, Control Rig
- 기존 아키텍처: Component 기반, StateTree로 상태 관리

### 구현 목표
벽에 손과 발을 고정시키는 IK 시스템을 구현하여 자연스러운 클라이밍 애니메이션 구현

### 설계 의도
1. **구조**
   - ClimbingComponent에 IK Target 계산 로직 추가
   - AnimInstance는 IK Target 데이터만 받아서 적용

2. **데이터 흐름**
   - Input → ClimbingComponent::Tick() 
   → LineTrace로 벽 표면 탐지 
   → Component Space 좌표 계산
   → AnimInstance 변수로 전달
   → Control Rig IK 적용

3. **주요 클래스/함수**
   - `UpdateHandIK()`: 매 프레임 손 목표 위치 계산
   - `UpdateFootIK()`: 매 프레임 발 목표 위치 계산
   - `FindValidGrabPoint()`: LineTrace로 잡을 수 있는 지점 탐색
   - `ConvertToComponentSpace()`: World → Component 좌표 변환

### 고민 지점
- [ ] LineTrace를 매 프레임(60fps) 실행하면 성능 문제가 있을까?
- [ ] Component Space 변환을 Game Thread에서 할지 AnimGraph에서 할지?
- [ ] 손/발 4개를 각각 계산 vs 배열로 일괄 처리?

### AI에게 질문
1. 이 설계에서 개선할 점이나 놓친 부분이 있을까?
2. UE5의 Animation System과 충돌할 부분은?
3. LineTrace 성능 최적화 방법은?
```

### AI에게 구체적으로 물어볼 것들

- ✅ **"이 구조에서 순환 참조나 강한 결합이 생길 가능성은?"**
- ✅ **"UE5의 [Animation/Physics/Replication] 시스템과 충돌할 부분은?"**
- ✅ **"이 설계를 3개의 독립적인 컴포넌트로 분리할 수 있을까?"**
- ✅ **"성능 관점에서 매 프레임 실행되는 부분이 적절한가?"**
- ✅ **"Multiplayer 환경에서 Replication 필요한 부분은?"**

---

## 2단계: 코드 리뷰 (코드 작성 후)

### 목적
- 설계 의도와 실제 구현의 차이 확인
- 클린 코드 원칙 준수 여부 검증
- 리팩토링 포인트 조기 발견

### 템플릿

```markdown
## [기능명] 코드 리뷰 요청

### 원래 설계 의도 (요약)
[1단계에서 말했던 목표 1-2문장 복사]

### 실제 구현 코드

#### 헤더 파일 (.h)
```cpp
// 핵심 선언부만
```

#### 구현 파일 (.cpp)
```cpp
// 주요 함수 구현
```

### 설계와 다르게 구현한 부분
- **[항목]**: [어떻게 다른지]
  - 이유: [왜 바꿨는지]

### AI에게 질문
1. 원래 의도한 설계와 실제 코드가 일치하는가?
2. 클린 코드 관점에서 개선할 부분은? (함수 길이, 네이밍, 응집도)
3. UE5 코딩 컨벤션을 위반한 부분은?
4. 리팩토링이 필요한 냄새나는 코드가 있는가?
5. 이 코드를 3개월 후 다른 팀원이 보면 이해할 수 있을까?
```

### 실전 예시

````markdown
## Climbing Hand/Foot IK 시스템 코드 리뷰 요청

### 원래 설계 의도
벽에 손과 발을 고정시키는 IK 시스템을 구현하여 자연스러운 클라이밍 애니메이션 구현

### 실제 구현 코드

#### ClimbingComponent.h
```cpp
UCLASS()
class UClimbingComponent : public UActorComponent
{
    GENERATED_BODY()
    
public:
    virtual void TickComponent(float DeltaTime, ...) override;
    
private:
    UPROPERTY()
    TObjectPtr<USkeletalMeshComponent> CachedMeshComponent;
    
    UPROPERTY()
    TObjectPtr<UMyAnimInstance> CachedAnimInstance;
    
    void UpdateIKTargets();
    FVector CalculateHandPosition(bool bIsLeft);
    FVector CalculateFootPosition(bool bIsLeft);
};
```

#### ClimbingComponent.cpp
```cpp
void UClimbingComponent::TickComponent(float DeltaTime, ...)
{
    Super::TickComponent(DeltaTime, ...);
    
    if (!bIsClimbing) return;
    
    UpdateIKTargets();
}

void UClimbingComponent::UpdateIKTargets()
{
    if (!ensure(CachedAnimInstance)) return;
    
    // 손 위치 계산
    FVector LeftHandPos = CalculateHandPosition(true);
    FVector RightHandPos = CalculateHandPosition(false);
    FVector LeftFootPos = CalculateFootPosition(true);
    FVector RightFootPos = CalculateFootPosition(false);
    
    // AnimInstance에 전달
    CachedAnimInstance->LeftHandIKTarget = LeftHandPos;
    CachedAnimInstance->RightHandIKTarget = RightHandPos;
    CachedAnimInstance->LeftFootIKTarget = LeftFootPos;
    CachedAnimInstance->RightFootIKTarget = RightFootPos;
}

FVector UClimbingComponent::CalculateHandPosition(bool bIsLeft)
{
    FVector Start = GetOwner()->GetActorLocation();
    FVector Forward = GetOwner()->GetActorForwardVector();
    FVector Right = GetOwner()->GetActorRightVector();
    
    // 좌우 오프셋
    FVector Offset = bIsLeft ? -Right * 30.0f : Right * 30.0f;
    FVector End = Start + Forward * 100.0f + Offset;
    
    // LineTrace 실행
    FHitResult HitResult;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(GetOwner());
    
    if (GetWorld()->LineTraceSingleByChannel(
        HitResult, Start, End, ECC_Visibility, Params))
    {
        return HitResult.ImpactPoint;
    }
    
    return End; // 실패 시 목표 위치 그대로 반환
}

FVector UClimbingComponent::CalculateFootPosition(bool bIsLeft)
{
    // CalculateHandPosition과 거의 동일한 로직...
    FVector Start = GetOwner()->GetActorLocation();
    FVector Forward = GetOwner()->GetActorForwardVector();
    FVector Right = GetOwner()->GetActorRightVector();
    FVector Down = -GetOwner()->GetActorUpVector();
    
    FVector Offset = bIsLeft ? -Right * 25.0f : Right * 25.0f;
    FVector End = Start + Forward * 80.0f + Down * 50.0f + Offset;
    
    FHitResult HitResult;
    FCollisionQueryParams Params;
    Params.AddIgnoredActor(GetOwner());
    
    if (GetWorld()->LineTraceSingleByChannel(
        HitResult, Start, End, ECC_Visibility, Params))
    {
        return HitResult.ImpactPoint;
    }
    
    return End;
}
```

### 설계와 다르게 구현한 부분
- **Component Space 변환 생략**: World Space로 직접 전달
  - 이유: Control Rig에서 World Space로 변환하는 노드가 더 안정적이었음
- **FindValidGrabPoint() 함수 분리 안 함**: CalculateHandPosition/FootPosition에 통합
  - 이유: 로직이 간단해서 분리할 필요 없다고 판단

### AI에게 질문
1. 원래 의도한 설계와 실제 코드가 일치하는가?
2. 클린 코드 관점에서 개선할 부분은?
3. UE5 코딩 컨벤션을 위반한 부분은?
4. 리팩토링이 필요한 냄새나는 코드가 있는가?
5. 이 코드를 3개월 후 다른 팀원이 보면 이해할 수 있을까?
````

### AI에게 구체적으로 물어볼 것들

- ✅ **"이 함수는 몇 가지 책임을 가지고 있나? 분리해야 하나?"**
- ✅ **"변수/함수 이름이 의도를 명확히 드러내는가?"**
- ✅ **"UPROPERTY/UFUNCTION 매크로가 적절히 사용되었나?"**
- ✅ **"nullptr 체크가 필요한 곳을 놓치지 않았나?"**
- ✅ **"중복 코드가 있는가? 공통 함수로 추출 가능한가?"**
- ✅ **"매직 넘버(30.0f, 100.0f)를 상수로 빼야 하나?"**

### 예상되는 AI 피드백 (위 예시 기준)

```
문제점:
1. CalculateHandPosition()과 CalculateFootPosition() 중복 코드
   → CalculateLimbPosition(ELimbType, Offset) 하나로 통합 가능

2. 매직 넘버 사용 (30.0f, 100.0f, 80.0f, 50.0f)
   → UPROPERTY로 노출하거나 상수화

3. UpdateIKTargets()에서 같은 패턴 4번 반복
   → TArray<FVector>나 구조체로 일괄 처리

리팩토링 제안:
struct FLimbIKTarget
{
    FVector Position;
    ELimbType Type;
};

TArray<FLimbIKTarget> CalculateAllLimbTargets();
```

---

## 3단계: 매일 30분 셀프 코드 리뷰 (퇴근 전)

### 목적
- 오늘 작성한 모든 코드를 신선한 시각으로 재검토
- AI 리뷰에서 놓친 부분 발견
- 다음날 아침 작업 전 혼란 방지

### 프로세스 (30분)

#### 1. 오늘 변경사항 확인 (5분)

**Rider에서:**
```
Ctrl+Alt+H → Local History
또는
Alt+9 → Git → Local Changes
```

- 오늘 수정한 파일 목록 확인
- 각 파일의 변경 라인 수 파악

#### 2. 파일별 5가지 질문 체크 (20분)

각 파일마다 다음 질문에 답하기:

| 질문 | 체크 방법 |
|------|----------|
| **1. 함수 이름만 보고 동작을 알 수 있나?** | 함수명을 가리고 구현부만 보기 → 함수명 추측 → 일치하는지 확인 |
| **2. 주석 없이 이해할 수 있나?** | 주석을 전부 지웠다고 가정하고 읽어보기 |
| **3. 다음 주에 봐도 이해할 수 있나?** | "이 코드를 일주일 뒤 처음 보는 사람이라면?" 관점으로 읽기 |
| **4. 다른 팀원이 봐도 이해할 수 있나?** | 도메인 지식이 없는 신입이 보면? |
| **5. 3개월 후 버그 수정할 때 쉽게 찾을 수 있나?** | "이 기능에 버그가 생기면 어디부터 봐야 하지?" |

#### 3. 체크리스트 기반 검증 (5분)

```markdown
### 오늘의 코드 품질 체크리스트

#### 함수 품질
- [ ] 모든 함수가 50줄 이하인가?
- [ ] 각 함수가 하나의 일만 하는가?
- [ ] 중첩된 if문이 3단계 이하인가?

#### 네이밍
- [ ] bool 변수에 Is/Has/Can/Should 접두사가 있나?
- [ ] 함수명이 동사로 시작하나? (Calculate, Update, Find 등)
- [ ] 임시 변수를 Temp/Var 같은 이름으로 쓰지 않았나?

#### UE5 컨벤션
- [ ] TObjectPtr을 사용했나? (UE5.0+)
- [ ] UPROPERTY()에 적절한 specifier가 있나?
- [ ] ensure/check를 사용해 방어적 프로그래밍을 했나?

#### 성능
- [ ] 매 프레임 실행되는 코드에 FindComponentByClass 같은 무거운 함수가 없나?
- [ ] TArray 동적 할당이 루프 안에 있지 않나?

#### 가독성
- [ ] 매직 넘버를 상수나 UPROPERTY로 빼냈나?
- [ ] 중복 코드가 3번 이상 반복되지 않나?
```

### 실전 루틴 타임라인

```
17:30 - 17:35 (5분): Local History로 오늘 변경사항 확인
17:35 - 17:55 (20분): 각 파일별 5가지 질문 체크
                      → 문제 발견 시 즉시 TODO 코멘트 추가
17:55 - 18:00 (5분): 체크리스트 검증 + 내일 아침 TODO 정리
```

### TODO 코멘트 패턴

문제 발견 시 즉시 수정하지 말고 TODO로 표시:

```cpp
// TODO: [REVIEW] 이 함수 80줄, UpdateHandIK()와 UpdateFootIK()로 분리 필요
void UClimbingComponent::UpdateIKTargets()
{
    // ...
}

// TODO: [REVIEW] bIsValid 변수명 너무 모호, bHasValidGrabPoint로 변경
bool bIsValid = CheckSomething();

// TODO: [REVIEW] 이 로직 StateComponent로 이동하는 게 나을 듯
if (CurrentState == EClimbState::Climbing)
{
    // 복잡한 상태 로직...
}

// TODO: [REFACTOR] 매직 넘버 30.0f → HandGrabOffset으로 상수화
FVector Offset = Right * 30.0f;
```

**다음날 아침 첫 업무로 이 TODO들부터 처리하기!**

---

## Rider 설정 최적화

### 1. Solution-Wide Analysis 활성화

```
Settings → Code Inspection → Enable solution-wide analysis
```
- 실시간으로 프로젝트 전체의 코드 문제 표시
- 우측 상단 인디케이터로 상태 확인

### 2. Code Cleanup 프로파일 생성

```
Settings → Editor → Code Cleanup
→ 새 프로파일 생성: "UE5 Daily Cleanup"
→ 단축키 지정: Ctrl+E, C
```

**추천 옵션:**
- ✅ Reformat code
- ✅ Remove unnecessary usings
- ✅ Apply auto-properties preferences
- ✅ Remove unused references

코드 작성 완료 후 습관적으로 `Ctrl+E, C` 누르기!

### 3. Commit Checks 활성화

```
Settings → Version Control → Commit
```
- ☑ Analyze code
- ☑ Check TODO (cleanup)
- ☑ Perform code cleanup
- ☑ Optimize imports

SVN 커밋 전 자동으로 코드 검증!

### 4. Inspection Severity 조정

```
Settings → Editor → Inspection Settings
```

**자주 어기는 규칙은 Error로 변경:**
- "Method is too long" → Error
- "Parameter can be converted to local variable" → Error
- "Redundant qualifier" → Warning → Error

### 5. 구조 시각화 도구

```
Alt+7: Structure 창
→ 함수 목록으로 클래스 복잡도 파악

Ctrl+Alt+U: Diagram
→ 클래스 의존성 시각화
```

---

## 실전 적용 가이드

### 첫 2주: 핵심 기능만 적용

모든 코드가 아니라 다음 경우에만 3단계 프로세스 적용:

- ✅ **새로운 Component 추가**
- ✅ **50줄 이상 함수**
- ✅ **3개 이상 클래스 연관**
- ✅ **성능 중요한 로직 (Tick 관련)**

### 3-4주차: 점진적 확대

작은 기능에도 적용 시작:

- ✅ **단순 함수 추가**: 2단계(코드 리뷰)만
- ✅ **리팩토링 작업**: 1+2단계 적용

### 1달 후: 자동화 단계

- 코드 작성 중에도 "AI한테 설명하기 쉬운 구조"로 자연스럽게 작성됨
- 1단계 브리핑 시간이 5분 이하로 단축
- 셀프 리뷰에서 발견되는 문제가 현저히 감소

---

## 내가 자주 하는 실수 리스트 (커스터마이징용)

이 섹션은 본인이 직접 채워 넣으세요:

```markdown
### 내가 자주 하는 실수들

#### 함수 설계
- [ ] 함수가 50줄 넘어가면 무조건 분리
- [ ] 하나의 함수가 3가지 이상 일을 하고 있음
- [ ] 

#### 네이밍
- [ ] bool 변수명에 Is/Has/Can 접두사 안 붙임
- [ ] 임시 변수를 TempVar, Var1 같은 이름으로 둠
- [ ] 

#### 에러 처리
- [ ] nullptr 체크를 if (!IsValid) return; 한 줄로 퉁침
- [ ] ensure() 대신 그냥 if문 사용
- [ ] 

#### UE5 특화
- [ ] FindComponentByClass를 Tick에서 호출
- [ ] UPROPERTY 없이 TObjectPtr 사용
- [ ] 

#### 구조
- [ ] 3단계 이상 중첩된 if문 사용
- [ ] 같은 패턴이 3번 이상 반복됨 (배열로 처리 안 함)
- [ ] 
```

이 리스트를 Rider 옆에 띄워놓고 함수 작성할 때마다 5초씩 훑어보기!

---

## 템플릿 파일 보관 위치

이 문서와 함께 다음 템플릿들을 `Docs/Templates/` 폴더에 보관:

```
Docs/
└── Templates/
    ├── 01_design_brief_template.md  (1단계 템플릿)
    ├── 02_code_review_template.md   (2단계 템플릿)
    └── 03_daily_checklist.md        (3단계 체크리스트)
```

필요할 때 복사해서 사용!

---

## 요약

| 단계 | 시기 | 소요 시간 | 도구 |
|------|------|-----------|------|
| **1. 설계 브리핑** | 코드 작성 전 | 5-10분 | AI (Claude/GPT) |
| **2. 코드 리뷰** | 코드 작성 후 | 10-15분 | AI (Claude/GPT) |
| **3. 셀프 리뷰** | 퇴근 전 | 30분 | Rider + 체크리스트 |

**핵심**: 
- AI는 "설계 검증자 + 코드 리뷰어"
- 본인은 "최종 검토자 + 습관 형성자"

2-3주 지속하면 코드 작성 중에도 자연스럽게 깔끔한 코드를 짜게 됩니다! 🚀
