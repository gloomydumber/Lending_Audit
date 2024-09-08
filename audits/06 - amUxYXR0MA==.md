# amUxYXR0MA==

## Scope

<table>
  <tr>
    <th>Name</th>
    <td>Lending Security Audit</td>
  </tr>
  <tr>
    <th>Target / Version</th>
    <td>Git Repository (<a href="https://bit.ly/4dVPMKI" target="_blank">Lending_solidity</a>): commit <mark>7bb6357b7e5742133581be4acafe811bbbe3ee77</mark> (<mark>main</mark> branch)</td>
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

### #1 `001` Denial of service could occur due to spamming the `distributeInterest()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|001|`distributeInterest()` 함수 내부에 반복문을 통해 모든 이용자들의 예치 및 부채에 대한 값을 업데이트하는데, 공격자가 이를 활용해 *Denial of Service* 공격을 시도할 수 있음.|Low|

#### Description

`distributeInterest()` 함수 호출 시 해당 함수 내에서 반복문을 통해 모든 이용자들의 예치 및 부채에 대한 값을 업데이트하는데, 공격자가 이를 활용해 Gas Cost가 상대적으로 저렴한 EVM 환경에서 해당 컨트랙트가 배포될 시에는 *Denial of Service* 공격을 시도할 수 있음. 특히, 해당 함수의 *modifier*가 *public* 으로 선언되어 반복적으로 호출 시에 해당 컨트랙트 및 프로토콜 혼잡으로 인해 다른 이용자들의 원할한 이용이 방해될 수 있음.

#### Impact

**Low**

해당 컨트랙트를 대상으로한 *Denial of Service* 공격에 의한 정상적인 트랜잭션 처리 지연.

#### Recommendation

`distributeInterest()` 함수 등에서 실행 전에 `require` 문 삽입 등을 통해 반복적이거나 올바르지 않은 요청 차단.

### #2 `002` The interest rate is hard-coded

|ID|Summary|Severity|
|:---:|-------|:---:|
|002|이자율이 하드코드되어 있음.|Informational|

#### Description

해당 Lending 컨트랙트는 24시간 당, 0.1% 의 이자율이 적용되고, 12초에 1 블록이 생성되는 EVM 환경에서 배포됨. 이에, 1 블록당 이자율은 아래와 같은 식으로 계산됨.

$$ 1.001^{(1/7200)}×10^{18} = 1.00000013881950034147221897264156541137271898967894423311848... × 10^{18} $$

이에, `calExponentialInterestByBlock()` 함수에서 `baseRate` 변수 값을 `100000013888888888888` 으로 하드코드 및 `ABDKMathQuad.sol` 라이브러리를 이용하여 구현하였으나, 좀 더 *Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요함

#### Impact

**Informational**

이자율의 괴리 발생.

#### Recommendation

*Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요.

### #3 `003` No Mechanism to Remove Inactive `accountAddr`

|ID|Summary|Severity|
|:---:|-------|:---:|
|003|더 이상 해당 프로토콜을 사용하지 않는 유저가 `accountAddr` 변수에 삭제되지 않고 영구히 저장 됨.|Informational|

#### Description

유저가 유동성을 공급하게 되면, 해당 유저의 관리를 `accountAddr` 가변 배열 변수에서 `address` 타입으로 저장하여 관리하게 되는데, 더 이상 해당 프로토콜을 사용하지 않는 유저가 `accountAddr` 변수에 삭제되지 않고 영구히 저장 됨. 해당 변수는 `distributeInterest()` 함수 등에서 지속적으로 불러와 반복문으로 순회하게 되는데, 해당 변수에 많은 유저의 주소가 저장되면 해당 컨트랙트 이용에 점점 많은 가스비가 소모될 수 있음.

#### Impact

**Informational**

이용자가 증가할 수록 컨트랙트 스토리지 사용 증대로 인해 가스비 증가 및 부작용 발생.

#### Recommendation

특정 유저의 유동성 공급 등이 종료되면 `accountAddr` 변수에서 해당 유저의 주소를 제거.

### #4 `004` The `transfer` method is used to send Ether within the `withdraw()` and `liquidate()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|004|`withdraw()` 및 `liquidate()` 함수에서, 유저에게 Ether를 출금할 때 최종적으로 `transfer`를 이용하고 있는데, 2300 가스 제한이 있어 실패할 수 있음.|Informational|

#### Description

`withdraw()` 및 `liquidate()` 함수에서, 유저에게 Ether를 출금할 때 최종적으로 `transfer`를 이용하고 있는데, `transfer`의 특성상 2300의 가스 제한이 있어 실패할 수 있음. 가령, 해당 컨트랙트의 이용자가 `receive()` 및 `fallback()` 등에서 특정 동작이 필요한 경우에, 예기치 못한 출금 실패 및 자금 손실이 발생할 수 있음.

#### Recommendation

`call()`을 통해 *low-level call* 도입 및 해당 *low-level call*의 성공 여부 체크 로직 도입 요망.

### #5 `005` Confirmation of the user's deposited *USDC* balance is not performed in the `withdraw()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|005|유저의 *USDC* 토큰의 예치 수량의 확인없이 `withdraw()` 함수로 임의의 수량 출금 가능.|Critical|

#### Description

이용자가 `withdraw()` 함수를 통해 *USDC* 토큰을 출금하려고 하는 경우, 출금하고자하는 수량에 대해서, 이용자가 기예치하거나 이자가 발생한 수량에 대한 검증이 없이 곧바로 출금이 이루어짐. 특히, *USDC*를 전혀 예치한 바가 없는 공격자가 임의의 수량 및 프로토콜에 예치된 *USDC* 전액을 출금 요청하더라도 아무런 검증 없이 출금이 이루어짐.

```solidity
function testExploit() external {
    vm.prank(victim);
    (bool success,) = address(lending).call(
        abi.encodeWithSelector(DreamAcademyLending.deposit.selector, address(usdc), 2000 ether)
    );
    assertTrue(success); // succeed, Vicitm deposited 2000 USDC to the protocol.

    vm.prank(exploiter);
    (success,) = address(lending).call(
        abi.encodeWithSelector(DreamAcademyLending.withdraw.selector, address(usdc), 2000 ether)
    );
    assertTrue(success); // succeed, Even though 2000 USDC was not deposited by exploiter, the withdrawal was successfully completed.
}
```

#### Impact

**Critical**

Lending Protocol의 예치 토큰(*USDC*)의 전액 탈취 가능.

#### Recommendation

`require()` 문 등을 통해, 출금하고자하는 유저의 출금 신청 액수 및 그에 따른 잔고 확인 비지니스 로직 도입 요망.