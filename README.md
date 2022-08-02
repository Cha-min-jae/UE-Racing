# UE-Racing(언리얼엔진 레이싱게임)
---
카트라이더 모작으로 만든 레이싱 게임입니다.

* 개발 엔진:Unreal Engine 4 (4.24.2)
* 제작기간:2020/02/05 ~ 2022/08/02
* 제작인원:1명
* 게임 장르:레이싱 게임

* 영상
* [![카트라이더 모작 포트폴리오](http://img.youtube.com/vi/O5TG4-8mJEg/0.jpg)](https://youtu.be/O5TG4-8mJEg) 

## 기술 설명​
* Vehicle(Pawn), MyPlayerController, Projectile등의 Actor
* 물체의 네트워크 동기화 모델 구현
* ReplayRecordComponent으로 실시간 정보를 기록.
* ReplayController로 기록된 정보를 읽어서 재생하기.
* Object Pooling 기법을 이용한 메모리 최적화

## Vehicle Movement
* 차량의 움직임은 Addforce(앞뒤)와 AddTorque(좌우)로 움직임을 구현
* 그립주행을 위해 미끄러지는 반대 방향으로 AddForce를 실행함
* 차량이 바퀴에 지탱하는 것처럼 표현
* 차량의 바퀴 중심점과 지면의 거리에 따라 위쪽 방향으로 Addforce가 실행된다.
![image2](https://user-images.githubusercontent.com/55441587/154320276-66e23977-f757-471d-a0f2-af64eb8ee5d0.jpeg)
```
void UWheelAndMeshComponent::WheelFunction(float Dt,bool isSlip)​

{​

///1.차량이 바퀴에 지탱하기 위해 위로 가해질 힘(Fz)을 계산하기 위해 Length(차량 바퀴축 위치와 지면의 거리)를 계산한다​

    SumVector = FVector::ZeroVector;​

    HitCount = 0;​

        for (int Raycount = 0; Raycount < OutHits.Num(); Raycount++)​

        {​

            FVector Start = GetComponentLocation(); // 시작점​

            FVector End = (Start - (UKismetMathLibrary::RotateAngleAxis(GetUpVector(), -60.f + Raycount * 4, GetRightVector())) * Radius * 1.0f);// 끝점​

            //추가설명::바퀴의 오른쪽방향 축을 기준으로 -60~60도에 3도를 간격으로 레이를 쏴준다.​

            isHits[Raycount] = GetWorld()->LineTraceSingleByChannel(OUT OutHits[Raycount], Start, End, ECC_Visibility, CollisionParams);​

            if (isHits[Raycount])​

            {​

                SumVector += OutHits[Raycount].Location;​

                HitCount++;​

            }​

        }​

        if (HitCount)​

            Length = (GetComponentLocation() - SumVector / HitCount).Size();​

        else​

            Length = Radius;​
            ///2.Fz(타이어에 받치는 힘)와 힘의 방향을 계산하고 힘을 가한다​

    Fz = TireStiffness * (Radius- Length);​

    Fz += (LastLength - Length) / Dt * 50;//상쇄(Damper 기능)​

    Fz = FMath::Clamp(Fz, 0.f, 10000.f);​

    FVector NormalVector = SumVector != FVector::ZeroVector ? ((GetComponentLocation() - SumVector / HitCount)).GetSafeNormal() : UpperComponent->GetUpVector();​

    UpperComponent->AddForceAtLocation(NormalVector * Fz * 100.f, GetComponentLocation());​

​

    LastLength = Length;​

​

///3.isSlip에 따라 스키드 자국을 만든다​

    if (isSlip && DecalMaterial != nullptr && HitCount)​

    {​

        UGameplayStatics::SpawnDecalAtLocation(GetWorld(), DecalMaterial, FVector(50.f, 10.f, 10.f), SumVector / HitCount, FRotator(-90.f, 0.f, 0.f), 30.f);​

    }​

}

​
```
![image3](https://user-images.githubusercontent.com/55441587/154320377-88d9fb27-f695-434b-af3f-f455230f3ae1.png)

## 백미러 구현
 백미러 카메라가 바라보는 방향을 보정하는 함수
- 액터기준 100m지름의 콜리전을 만든다.
- 그 안에 있는 액터중에 Vehicle객체만 모은다.
- Vehicle객체들 중 뒤에 있는 것만 고르고
- 그 중에 제일 가깝거나 액터 뒤쪽 기준으로 각도가 좁은 것을 최종적으로 고른다.
- BackMirror방향을 내 시점에서 선택된 차량을 바라보는 방향으로 설정한다
```
void AMyCustomVehicle::CheckBack()​

{​

///1.액터기준 100m지름의 콜리전을 만든다.​

    UWorld* World = GetWorld();​

    if (nullptr == World) return;​

    FVector Center = GetActorLocation();​

    float DetectRadius = 10000.0f;​

    TArray<FOverlapResult> OverlapResults;​

    FCollisionQueryParams CollisionQueryParam(NAME_None, false, this);​

    bool bResult = World->OverlapMultiByChannel(​

        OverlapResults,​

        Center,​

        FQuat::Identity,​

        ECollisionChannel::ECC_GameTraceChannel2,​

        FCollisionShape::MakeSphere(DetectRadius),​

        CollisionQueryParam​

    );​
    ///2.그 안에 있는 액터중에 Vehicle객체만 모은다.​

    TArray<AMyCustomVehicle*> VehicleArray;​
    if (bResult)​
    {​
        for (auto OverlapResult : OverlapResults)​

        {​
            AMyCustomVehicle* TargetItemVehicle = Cast<AMyCustomVehicle>(OverlapResult.GetActor());​
            if (TargetItemVehicle)​
            {​
                VehicleArray.Add(TargetItemVehicle);​
            }​
        }​
    }​
    ///3.Vehicle객체들 중 뒤에 있는 것만 고르고​
    ///4.그 중에 제일 가깝거나 액터 뒤쪽 기준으로 각도가 좁은 것을 최종적으로 고른다.​

    BackMirrorVehicleTarget = nullptr;​

    for (auto VV : VehicleArray)​
    {
        FVector dist = (VV->GetActorLocation() - GetActorLocation());
        float dot = FVector::DotProduct(GetActorForwardVector(), dist.GetSafeNormal());
        float AcosAngle = FMath::Acos(dot);    // dot한 값을 아크코사인 계산해 주면 0 ~ 180도 사이의 값 (0 ~ 1)의 양수 값만 나온다.
        float angle = FMath::RadiansToDegrees(AcosAngle); //그값은 degrees 값인데 이것에 1라디안을 곱해주면 60분법의 도가 나온다.
        if (150.f <=angle&&angle <= 210.f)
        {
            if (BackMirrorVehicleTarget == nullptr)
            {
                BackMirrorVehicleTarget = VV;
                break;
            }
            else
            {
                /// 현재 검사할 타겟이 지정된 목표보다 거리가 낮을경우 새로 변경​
                FVector distPrev = BackMirrorVehicleTarget->GetActorLocation() - GetActorLocation();
                float anglePrev = FMath::Abs<float>(FVector::DotProduct(GetActorForwardVector(), distPrev.GetSafeNormal()) * 57.14);
                if (distPrev.Size() > dist.Size()) BackMirrorVehicleTarget = VV;
             }
         }
       }​

​
    ///5.BackMirror방향을 내 시점에서 선택된 차량을 바라보는 방향으로 설정한다​

    if (BackMirrorVehicleTarget != nullptr)
    {
        FRotator LookAtTarget = UKismetMathLibrary::FindLookAtRotation(Center, BackMirrorVehicleTarget->GetActorLocation());
        BackMirrorSpringArm->SetWorldRotation(LookAtTarget);
    }
}
```

## 물체의 네트워크 동기화 모델 구현
* 자신의 폰의 위치를 자신의 클라이언트 시점의 위치로 다른 클라이언트들의 시점의 위치를 비슷하게 맞추는 것​
* 서버에서 받아온 정보를 토대로 외삽값을 산출​
* 현재 클라이언트 정보와 외삽값으로 보간​

* 참조 영상
* [NDC] 〈카트라이더〉 0.001초 차이의 승부 - 300km/h 물체의 네트워크 동기화 모델 구현기​
* https://www.youtube.com/watch?v=r4ZaolMQOzE&t=23s​

![2022-02-20](https://user-images.githubusercontent.com/55441587/154846290-91c9f63a-674d-411a-b7da-ae07918e7e79.png)
![2022-02-20 (4)](https://user-images.githubusercontent.com/55441587/154846283-ece3c0a5-ffdb-427f-8b3c-fb5eee8cdfa5.png)
![2022-02-20 (5)](https://user-images.githubusercontent.com/55441587/154846286-8a7f77b2-36e6-4b32-814c-f53ec921fb63.png)


## Projectile,Mine,Spaceship등의 투사체 오브젝트&오브젝트 풀링

* 메모리 최적화를 위해 오프젝트 풀링 기법 사용
* 투사체 클래스는 풀액터(PoolActor)의 클래스를 상속 받음​
* 인게임 내 서버 측에서 한 개체의 액터풀이 투사체들을 관리​

* 장점: 오브젝트의 생성과 삭제로 인한 과부하 방지​
* 오브젝트 삭제로 인한 가비지 컬렉터 실행(성능 저하 요인)을 방지​

## 리플레이 병합 시스템

## 아이템전 리플레이
