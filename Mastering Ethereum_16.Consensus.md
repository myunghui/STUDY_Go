# 16장. 합의알고리즘(Consensus)

### Consensus란?

- 분산화된 환경에서 중재자(검열, 신뢰)없이 합의에 도달하기 위한 방법론

- Smart Contract 보다는 낮은 추상화 레벨에 필요한 알고리즘. 예) 채굴, 검증...등

- Ethereum의 Consensus : 현재는 PoW (Ethash)이나, PoS로 전환 예정(Casper 프로젝트)

- 비잔틴 장군 문제 (The byzantine generals problem)

  전체 검증자 (validator) 중 3분의 2가 합의 도달(reaching consensus) 되어야 해결 가능 

------

### 16.1 PoW (Proof of Work)
- 블록의 유효성을 Hash연산으로 증명 =>  채굴(mining)
  목표값 이하의 Hash를 찾는 과정을 무수히 반복

- 채굴 능력 ==  빠른 계산 능력

- 보안 == Hash

- 가장 빨리 채굴된 블록만 인정을 받고 나머지는 버려지는 구조로 이중지불 문제가 해결

- 채굴의 목적 : 블록체인을 안전하게 하며, 보상을 수단으로 분산네트워크의 참여를 유도. 

- 보상(Reword) : Transaction Fee

- 처벌(Purnishment) : 채굴 비용 (전기 요금, 채굴기 장비 비용....) 

- BitCoin, Ethereum, Litecoin, Zcash, Monero 등이 채택

- 문제점

  - 51% 공격

    전체 채굴 연산능력의 과반수 이상을 보유하는 자가 네트워크 상에서 부정을 일으키는 공격.

    높은 비용에도 불구하고 모나코인(거래소 이중 입금), 버지(마이닝풀에서 타임스탬프 조작), 비트코인골드(거래소 이중 지불) 등의 공격 있었음.

  - PoW 파이널리티 불확실성

    블록체인이 분기되는 경우, Transaction이 Rollback 될수도 있음.

    비트코인은 해당 Transcation이 Confirm된 이후, 6번의 Confirm 이후 확정되도록 보완.

  - BitCoin의 채굴 집중화 : Hash 연산 난이도가 높아지도록 설계되어 있어, 특화된 ASIC 장치(반도체)를 가진 마이너들에 의해 집중화되어 탈중앙화의 가치에 역행하는 문제 발생. 

  - 과도한 전기 에너지 소모가 사회 문제.

    아이슬란드에서는, 가정용 전기보다 채굴업체들의 전기 사용량이 커지는 현상 발생.

    비트코인과 이더리움의 채굴로 소비되는 에너지량이 시리아(에너지 소비량 72위)보다 많음.

  - Ethereum은 BitCoin의 채굴의 집중화를 유도하는 원리를 회피하도록 (ASIC 장치 무력화) 경량 클라이언트에서 채굴 가능하도록 하도록 설계됨

- 탈중앙화 관련 참고자료

  [탈중앙화에 대한 정의와 측정방법](https://medium.com/symverse/탈중앙화에-대한-정의와-측정방법-5dbf94d2edd3)

  [탈중앙화란 무엇인가-비탈릭 부테린](https://medium.com/hashed-kr/the-meaning-of-decentralization-kr-f7942cf9fed6)

- Hash 관련 참고자료

  [Hash 함수의 특징과 유형](http://ihpark92.tistory.com/60?category=743397)

  [Sha256 Hash 실습](https://anders.com/blockchain/hash.html)

------
### 16.2  Proof of Stake (PoS)

- 블록의 유효성 증명을 지분으로 증명 => Forging

  네트워크내 지분(stake) 양에 비례하여 블록을 생성할 권한을 부여받는 방식

  해당 네트워크의 자산을 소유하는 만으로 참여 가능. 채굴 장비 불필요.

  Validator는 자신의 자산을 예치(Lock)하고, 검증을 위한 특별한 Transaction 을 보내는 것으로 참여 가능.

- 블록 생성자와 지분 생성자의 이해관계를 일치시킴.

  - PoW에서 51%의 Hash Power를 가지는 비용 = 약 2500억원
  - PoS에서 전 세계 자산의 51% = 약 25조원

- 보안 == 자산

- 보상(Reword) :  Validation Fee. Stack 유지.

- 처벌(Purnishment) : Stack 감소.

- DPOS(Delegated Proof of Stake, DPoS)

  - 지분을 위임받은 Validator가 Forging 수행하고 그 수익을 배분하는 모델
  - 직접 네트워크에 참여하는 모델보다 실제 접속하는 노드수는 작아져 성능에 유리
  - Bitshares, Steem, Zcash, EOS등이 채택

- 문제점

  - Nothing at Stake 

    자산증명을 하는데 있어 한계 비용(marginal cost)이 전혀 없다.

    체인의 fork가 발생시, 노드가 투표를 할 때  두 블럭체인에 모두 투표를 해도 이 노드가 전혀 손해보는 것이 없는 것.

    지분이 많은 참여자는 의도에 따라 분기를 선택할 확율 높아짐.

    - The long range attack consensus problem

      ![](D:\01_STUDY\Go_Ethereum\Mastering Ethereum\그림_Long Range Attack.jpg)

    - The short range attack consensus problem

      ![](D:\01_STUDY\Go_Ethereum\Mastering Ethereum\그림_Short Range Attack.jpg)

  - 불공평한 경제 모델

  - 참여자는 항상 네트워크에 연결된 온라인 상태여야 함 (DPOS 제외)

  - Validator가 Validation Transaction을 보내는 동안 Hot Wallet이 연동되어야 하므로, 해킹 위험 높음.

------

### 16.3 Ethash

- Ethereum의 메모리연산 기반 Pow 합의 알고리즘

- KECCAK(SHA-3) 사용

  cf) Bitcoin은 SHA-256 사용

- Ethash 동작 원리

  1. 현재 포인트까지 블록의 헤더를 스캔해 Seed를 계산(seed는 30000블록 마다 바뀜)

  1. 이 Seed를 통해 pseudo-random 하게 cache data 생성
  2. cache data로 1GB 이상의 dataset을 생성. full client와 miner는 이 dataset(DAG)을 저장
  3. mining을 수행할 때 이 dataset의 일부를 함께 Hashing

- DAG (Dagger Hashimoto)

  - Dagger–Hashimoto algorithm의 수정 버전 사용

     Vitalik Buterin’s Dagger algorithm과 Thaddeus Dryja’s Hashimoto algorithm의 조합

  - 30,000 블록(Epoch) 단위로 생성

    블록 헤더들의 Seed으로 Cache생성하여 Full Dataset(1GB 이상)을 생성.

    - Seed : 비어있는 32개 길이의 배열을 Epoch 수 만큼 해싱
    - Cache : Seed 사용한 pseudo-random 하게 cache data 생성
  ![](D:\01_STUDY\Go_Ethereum\Mastering Ethereum\그림_Ethash DAG Acyclic Graph.jpg)
    
  - Mining시 DAG 사용하여 Hashing 함 

  - DAG 용량은 선형적으로 증가하는 구조로, 메모리 읽기 연산과 데이터 저장 공간에 제약이 ASIC 저항성을 높이도록 설계됨.

  - DAG 를 통해 예측 불가능한 순차적 Memory 연산을 요구하도록 설계

  ![](D:\01_STUDY\Go_Ethereum\Mastering Ethereum\그림_Ethash Hashing Algorithm.png)

- 한계점

  - Ethereum의 합의알고리즘이 Pos로 바뀔것을 예고한 것과 Ethash의 알고리즘으로 ASIC의 진입을 억제하고 있으나, PoS 가 적용되는 시점에는 ASIC 의 채산성이 바뀔수 있다는 한계가 존재.

    PoW 기반의 Ethereum 암호화폐(RIPL, Rpiq)가 여전히 유통되고,  이더리움 클래식이 존재하기 떄문.

  - PoW가 암호화폐의 발행량이 줄어든 구조인데 비해, PoS는 조금씩 늘어나는 구조.

    대량의 자원을 초기에 투입할 유인이 적어 가격이 안정적이나, PoW보다 보상이 상대적으로 작다.

  - 초기의 암호화폐의 발행에 대한 공평성에 문제 있음.

------

### 16.4 Casper

- Ethereum의  Pos 합의 알고리즘으로, Casper FFG / Casper CBC 의 2가지로 설계되고 있음.

- Casper FFG (Friendly finality gadget)

  - Vitalik Buterin 이 주도적으로 진행중

  - PoW와 PoS의 Hybrid 형태. FFG는 PoW 기반 블록체인 위에 PoS 알고리즘을 덮어씌우는 방식. 구현합니다. 블록체인은 Ethash 알고리즘을 통해 한 블록씩 생성되지만, 블록이 50번 생성될 때마다 PoS의 Checkpoint 를 찍고 그 시점에서 네트워크의 Validator들이 완결성(Finanlity)을 검증.

- Casper CBC (Correct-By-Construction)

  - Vlad Zamfir 가 주도적으로 진행중
  - 합의 프로토콜을 “도출"하는 방법에 집중
  - 안정성을 담보하는 방법들에 집중
  - 6개의 CBC 프로토콜
    1. Casper TFBC (the Friendly Binary Consensus Protocol): 0과 1 중 하나를 선택 동의
    2. Casper TFOC (the Friendly Ordinal Consensus Protocol): 정수에 동의
    3. Casper TFLO (the Friendly List Ordering Protocol) : 리스트 순서에 동의
    4. Casper TFG (the Friednly GHOST Protocol): 블록체인상 동의
    5. Casper TFCSR (the Friendly Concurrent Schedule Replication Protocol) :병행 스케줄상 동의
    6. Casper TFSB (the Friendly Sharded Blockchain): 여러 개의 샤드가 있는 블록체인상 동의

- 참고자료

  [Casper FFG Overview](https://medium.com/onther-tech/casper-ffg-overview-e09fbe4f7d2c)


------

### 16.5 Principles of Consensus

- 합의 알고리즘 원칙과 가정을 이해 할 수 있는 질문들

```
- Who can change the past, and how? (This is also known as immutability.)
- Who can change the future, and how? (This is also known as finality.)
- What is the cost to make such changes?
- How decentralized is the power to make such changes?
- Who will know if something has changed, and how will they know?
```



