# 컨텍스트 스위칭(Context Switching)은 왜 비용이 큰가
**PC(Program Counter)만 바꾸는 것이 아니라 메모리 접근, 캐시 무효화, 스케줄러 오버헤드가 모두 포함되어있기 때문.**

 특히 커널 스레드 ↔ 유저 스레드 전환이면 더 비싸진다.

## 레지스터 상태 저장/복원

- PC(프로그램 카운터)
- SP(스택 포인터)
- general-purpose registers(범용 레지스터들)
- 상태 레지스터(CPSR / EFLAGS)
    - CPU가 어떤 상태로 계산 중이었는지를 나타낸다.
    - carry flag, zero flag, sign flag, interrupt enabled, CPU 모드 (유저모드/커널모드) 같은 것들.
    - ex. 산술 연산 후 Z 플래그가 1이면 결과가 0, 비트 연산 후 carry 여부
- 커널 모드 정보(시스템 콜 중이라면 더 많음)
    - 커널 스택, 시스템 콜 번호, 커널 함수의 지역 변수, 내부 스케줄러 구조체, 파일 테이블/메모리 정보..

## **캐시 미스(cache miss)**

CPU가 필요한 데이터를 캐시에서 못 찾는 상황

L1/L2/L3 에서 못찾으면 ROM까지 가야하기 때문에 이 탐색 과정이 시간을 많이 잡아먹는다.

OS 논문에서 context switch 비용의 30~70%가 캐시 miss 때문이라고 한다.

***캐시 warm-up**

새로운 작업을 시작할 때 캐시에 필요한 데이터가 아직 없는 상태에서 CPU가 RAM에서 데이터를 하나씩 꺼내서 캐시에 채워넣어 성능이 정상화되는 것을 캐시가 따뜻해진다고(ㅋㅋ)해서 warm up이라고 함.

## TLB Flush 발생

TLB = 가상 주소 → 물리 주소 변환 캐시

프로세스 간 전환 시 주소 공간이 달라지므로 TLB(Translation Lookaside Buffer)를 flush해야 할 수 있음

TLB miss는 대부분 캐시 miss보다 더 큼

## **커널 모드 진입/복귀 비용**

1. 유저 모드 → 커널 모드 진입
2. 커널에서 스케줄러 실행
3. 다시 유저 모드 복귀

이 과정에서

- privilege level 변경
- 메모리 보호 매핑 변경
- CPU 파이프라인 flush

발생

## **스케줄러 오버헤드**
