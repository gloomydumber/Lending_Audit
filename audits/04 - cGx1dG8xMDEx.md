# cGx1dG8xMDEx

## Scope

<table>
  <tr>
    <th>Name</th>
    <td>Lending Security Audit</td>
  </tr>
  <tr>
    <th>Target / Version</th>
    <td>Git Repository (<a href="" target="_blank">-Lending-DEX-_solidity</a>): commit <mark>df61bf1d26cf2d0ec6059c2cacc83e6bbffb45e5 </mark> (<mark>main</mark> branch)</td>
  </tr>
  <tr>
    <th>Application Type</th>
    <td>Smart contracts</td>
  </tr>
  <tr>
    <th>Lang. / Platforms</th>
    <td>Smart contracts [Solidity]</td>
  </tr>
</table>

## Findings

### #1 `001` The core metrics of the contract are hard-coded and arbitrary overall

|ID|Summary|Severity|
|:---:|-------|:---:|
|001|컨트랙트의 핵심 *metric* 값들이 모두 하드코드되거나 본래 구현 의도한 바와 달리 의미를 알 수 없게 정의되어 있음.|Informational|

#### Description

해당 Lending 컨트랙트는 24시간 당, 0.1% 의 이자율이 적용되고, 12초에 1 블록이 생성되는 EVM 환경에서 배포됨.

또, *Loan To Value (LTV)*는 50%, *liquidation threshold*는 75% 로 설정되어야함.

그러나, 해당 사항과는 달리, 컨트랙트의 핵심 *metric* 값들이 아래와 같은 값들로 설정되어 있음.

```solidity
uint256 private constant LIQUIDATION_THRESHOLD = 66; // 75%
uint256 private constant LIQUIDATION_CLOSE_FACTOR = 25; // 25%
uint256 private constant INTEREST_RATE = 5; // 5% per year
uint256 private constant SECONDS_PER_YEAR = 31536000;
```

또, `getAccruedSupplyAmount()` 함수에서는 아래와 같이, 컨트랙트의 일반적인 동작이 아닌, 특정 상황 및 테스트코드를 통과 하기 위해 정의되어있음.

```solidity
function getAccruedSupplyAmount(address user) external view returns (uint256) {
    //수수료가 복리인 거 같은데 그냥 if문으로,,,,,
    uint256 acc = userAccounts[msg.sender].usdcCollateral;
    if (block.number == 7200001) {
        if (acc == 30000000 ether) {
            return 30000792 ether; //1000일 후 user는 이 친구밖에 없음
        }
    }

    if (block.number == 7200001 + 3600000) {
        if (acc == 30000000 ether) {
            if (userList.length == 2) {
                return 30001605 * 1e18;
            } else {
                return 30001547 * 1e18;
            } //1000일 후 user는 이 친구밖에 없음
        }
        if (acc == 100000000 ether) {
            return 100005158 * 1e18;
        }
        if (acc == 10000000 ether) {
            return 10000251 * 1e18;
        }
    }
}
```

이러한 특정 상황에 끼워 맞춘 비지니스 로직 도입으로, 일반적인 Lending Protocol 로서의 이용이 불가함.

#### Impact

**Informational**

일반적인 Lending Protocol 로서의 컨트랙트 이용 불가.

#### Recommendation

Lending Protocol 로서의 전반적인 코드 재작성 및 고도화 요망.

### #2 `002` No Mechanism to Remove Inactive `userList`

|ID|Summary|Severity|
|:---:|-------|:---:|
|002|더 이상 해당 프로토콜을 사용하지 않는 유저가 `userList` 변수에 삭제되지 않고 영구히 저장 됨.|Informational|

#### Description

유저가 유동성을 공급하게 되면, 해당 유저의 관리를 `userList` 가변 배열 변수에서 `address` 타입으로 저장하여 관리하게 되는데, 더 이상 해당 프로토콜을 사용하지 않는 유저가 `userList` 변수에 삭제되지 않고 영구히 저장 됨.

#### Impact

**Informational**

이용자가 증가할 수록 컨트랙트 스토리지 사용 증대로 인해 가스비 증가 및 부작용 발생.

#### Recommendation

특정 유저의 유동성 공급 등이 종료되면 `userList` 변수에서 해당 유저의 주소를 제거.

### #3 `003` The `transfer` method is used to send Ether within the `withdraw()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|003|`withdraw()` 함수에서, 유저에게 Ether를 출금할 때 최종적으로 `transfer`를 이용하고 있는데, 2300 가스 제한이 있어 실패할 수 있음.|Informational|

#### Description

`withdraw()` 함수에서, 유저에게 Ether를 출금할 때 최종적으로 `transfer`를 이용하고 있는데, `transfer`의 특성상 2300의 가스 제한이 있어 실패할 수 있음. 가령, 해당 컨트랙트의 이용자가 `receive()` 및 `fallback()` 등에서 특정 동작이 필요한 경우에, 예기치 못한 출금 실패 및 자금 손실이 발생할 수 있음.

#### Impact

**Informational**

이용자의 예기치 못한 출금 실패 및 자금 손실.

#### Recommendation

`call()`을 통해 *low-level call* 도입 및 해당 *low-level call*의 성공 여부 체크 로직 도입 요망.