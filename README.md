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
