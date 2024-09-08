# ZG9rcGFyazIx

## Scope

<table>
  <tr>
    <th>Name</th>
    <td>Lending Security Audit</td>
  </tr>
  <tr>
    <th>Target / Version</th>
    <td>Git Repository (<a href="https://bit.ly/4grf5pU" target="_blank">DEX_Lending_solidity</a>): commit <mark>7c149b1257ebb4ec30925da15c2e9821c4db5037</mark> (<mark>main</mark> branch)</td>
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

### #1 `001` Denial of service could occur due to spamming the `updateDepositInterest()` modifier

|ID|Summary|Severity|
|:---:|-------|:---:|
|001|`updateDepositInterest()` *modifier* 내부에 반복문을 통해 모든 이용자들의 예치 및 부채에 대한 값을 업데이트하는데, 공격자가 이를 활용해 *Denial of Service* 공격을 시도할 수 있음.|Low|

#### Description

`updateDepositInterest()` *modifier* 내부에 반복문을 통해 모든 이용자들의 예치 및 부채에 대한 값을 업데이트하는데, 공격자가 이를 활용해 Gas Cost가 상대적으로 저렴한 EVM 환경에서 해당 컨트랙트가 배포될 시에는 *Denial of Service* 공격을 시도할 수 있음. 특히, `getAccruedSupplyAmount()` 함수 등을 반복적으로 호출 시에 해당 컨트랙트 및 프로토콜 혼잡으로 인해 다른 이용자들의 원할한 이용이 방해될 수 있음.

```solidity
function testSpammingGetAccruedSupplyAmount() external {
    usdc.transfer(user3, 30000000 ether);
    vm.startPrank(user3);
    usdc.approve(address(lending), type(uint256).max);
    lending.deposit(address(usdc), 30000000 ether);
    vm.stopPrank();

    supplyUSDCDepositUser1();
    supplySmallEtherDepositUser2();

    dreamOracle.setPrice(address(0x0), 4000 ether);

    bool success;

    vm.startPrank(user2);
    {
        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.borrow.selector, address(usdc), 1000 ether)
        );
        assertTrue(success);

        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.borrow.selector, address(usdc), 1000 ether)
        );
        assertTrue(success);

        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.withdraw.selector, address(0x0), 1 ether)
        );
        assertFalse(success);
    }
    vm.stopPrank();

    vm.roll(block.number + (86400 * 1000 / 12));
    {
        vm.startPrank(user3);
        assertTrue(lending.getAccruedSupplyAmount(address(usdc)) / 1e18 == 30000792); // Call Succeed
        assertTrue(lending.getAccruedSupplyAmount(address(usdc)) / 1e18 == 30000792); // Call Succeed
        assertTrue(lending.getAccruedSupplyAmount(address(usdc)) / 1e18 == 30000792); // Call Succeed
        assertTrue(lending.getAccruedSupplyAmount(address(usdc)) / 1e18 == 30000792); // Call Succeed
        assertTrue(lending.getAccruedSupplyAmount(address(usdc)) / 1e18 == 30000792); // Call Succeed
        assertTrue(lending.getAccruedSupplyAmount(address(usdc)) / 1e18 == 30000792); // Call Succeed
        vm.stopPrank();
    }
}
```

#### Impact

**Low**

해당 컨트랙트를 대상으로한 *Denial of Service* 공격에 의한 정상적인 트랜잭션 처리 지연.

#### Recommendation

`updateDepositInterest()` *modifier* 등에서 실행 전에 `require` 문 삽입 등을 통해 반복적이거나 올바르지 않은 요청 차단.

### #2 `002` The interest rate is hard-coded

|ID|Summary|Severity|
|:---:|-------|:---:|
|002|이자율이 하드코드되어 있음.|Informational|

#### Description

해당 Lending 컨트랙트는 24시간 당, 0.1% 의 이자율이 적용되고, 12초에 1 블록이 생성되는 EVM 환경에서 배포됨. 이에, 1 블록당 이자율은 아래와 같은 식으로 계산됨.

$$ 1.001^{(1/7200)}×10^{18} = 1.00000013881950034147221897264156541137271898967894423311848... × 10^{18} $$

이에, `INTEREST_RATE` 변수 값을 `1000000011568290959081926677` 으로 하드코드하고, `DSMath.sol` 라이브러리로 연산하여 구현하였으나, 좀 더 *Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요함

#### Impact

**Informational**

이자율의 괴리 발생.

#### Recommendation

*Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요.

### #3 `003` No Mechanism to Remove Inactive `lenders`

|ID|Summary|Severity|
|:---:|-------|:---:|
|003|더 이상 해당 프로토콜을 사용하지 않는 유저가 `lenders` 변수에 삭제되지 않고 영구히 저장 됨.|Informational|

#### Description

유저가 유동성을 공급하게 되면, 해당 유저의 관리를 `lenders` 가변 배열 변수에서 `address` 타입으로 저장하여 관리하게 되는데, 더 이상 해당 프로토콜을 사용하지 않는 유저가 `lenders` 변수에 삭제되지 않고 영구히 저장 됨. 해당 변수는 `updateDepositInterest()` *modifier* 등에서 지속적으로 불러와 반복문으로 순회하게 되는데, 해당 변수에 많은 유저의 주소가 저장되면 해당 컨트랙트 이용에 점점 많은 가스비가 소모될 수 있음.

#### Impact

**Informational**

이용자가 증가할 수록 컨트랙트 스토리지 사용 증대로 인해 가스비 증가 및 부작용 발생.

#### Recommendation

특정 유저의 유동성 공급 등이 종료되면 `lenders` 변수에서 해당 유저의 주소를 제거.

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