# UE-Racing(언리얼엔진 레이싱게임)
---
카트라이더 모작으로 만든 레이싱 게임입니다.

* 개발 엔진:Unreal Engine 4 (4.24.2)
* 제작기간:2020/02/05 ~ 2021/09/01​
* 제작인원:1명​
* 게임 장르:레이싱 게임​

### 기술 설명​
* Vehicle(Pawn), MyPlayerController, Projectile등의 Actor ​
* 물체의 네트워크 동기화 모델 구현
* ReplayRecordComponent으로 실시간 정보를 기록.​
* ReplayController로 기록된 정보를 읽어서 재생하기.​
* Object Pooling 기법을 이용한 메모리 최적화​

### Vehicle Movement

* 차량의 움직임은 Addforce(앞뒤)와 AddTorque(좌우)로 움직임을 구현​

* 그립주행을 위해 미끄러지는 반대 방향으로 AddForce를 실행함​

* 차량이 바퀴에 지탱하는 것처럼 표현​

* 차량의 바퀴 중심점과 지면의 거리에 따라 위쪽 방향으로 Addforce가 실행된다.​
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
