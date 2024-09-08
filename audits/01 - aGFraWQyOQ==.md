# aGFraWQyOQ==

## Scope

<table>
  <tr>
    <th>Name</th>
    <td>Lending Security Audit</td>
  </tr>
  <tr>
    <th>Target / Version</th>
    <td>Git Repository (<a href="https://bit.ly/3Z9IuP8" target="_blank">Lending_solidity</a>): commit <mark>25c37d56033fd27a3f5fd772223ff2552438d8ab</mark> (<mark>main</mark> branch)</td>
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

### #1 `001` Denial of service could occur due to spamming the `updateAll()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|001|`updateAll()` 함수 내부에 반복문을 통해 모든 이용자들의 예치 및 부채에 대한 값을 업데이트하는데, 공격자가 이를 활용해 *Denial of Service* 공격을 시도할 수 있음.|Low|

#### Description

`updateAll()` 함수 내부에 반복문을 통해 모든 이용자들의 예치 및 부채에 대한 값을 업데이트하는데, 공격자가 이를 활용해 Gas Cost가 상대적으로 저렴한 EVM 환경에서 해당 컨트랙트가 배포될 시에는 *Denial of Service* 공격을 시도할 수 있음. 특히, `liquidate()` 함수 등에서, 청산 물량을 아래와 같이 `0` 으로 설정하여 반복적으로 `liquidate()` 호출 시에 해당 컨트랙트 및 프로토콜 혼잡으로 인해 다른 이용자들의 원할한 이용이 방해될 수 있음.

```solidity
function testLiquidationWithAmountZeroSpamming() external {
    vm.deal(user2, 100000000 ether);
    vm.startPrank(user2);
    {
        lending.deposit{value: 1 ether}(address(0x00), 1 ether);
    }
    vm.stopPrank();

    dreamOracle.setPrice(address(0x0), 4000 ether);

    vm.startPrank(user2);
    {
        // use all collateral
        (bool success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.borrow.selector, address(usdc), 2000 ether)
        );
        assertTrue(success);

        assertTrue(usdc.balanceOf(user2) == 2000 ether);

        usdc.approve(address(lending), type(uint256).max);
    }
    vm.stopPrank();

    dreamOracle.setPrice(address(0x0), (4000 * 66 / 100) * 1e18); // drop price to 66%
    usdc.transfer(user3, 3000 ether);

    vm.startPrank(user3);
    {
        usdc.approve(address(lending), type(uint256).max);
        (bool success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.liquidate.selector, user2, address(usdc), 0 ether)
        );
        assertTrue(success); // succeed
                    (bool success1,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.liquidate.selector, user2, address(usdc), 0 ether)
        );
        assertTrue(success1); // succeed
                    (bool success2,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.liquidate.selector, user2, address(usdc), 0 ether)
        );
        assertTrue(success2); // succeed
                    (bool success3,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.liquidate.selector, user2, address(usdc), 0 ether)
        );
        assertTrue(success3); // succeed
                    (bool success4,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.liquidate.selector, user2, address(usdc), 0 ether)
        );
        assertTrue(success4); // succeed
    }
    vm.stopPrank();
}
```

#### Impact

**Low**

해당 컨트랙트를 대상으로한 *Denial of Service* 공격에 의한 정상적인 트랜잭션 처리 지연.

#### Recommendation

`updateAll()` 함수 실행 전에 `require` 문 삽입 등을 통해 반복적이거나 올바르지 않은 요청 차단.

### #2 `002` The interest rate is hard-coded

|ID|Summary|Severity|
|:---:|-------|:---:|
|002|이자율이 하드코드되어 있음.|Informational|

#### Description

해당 Lending 컨트랙트는 24시간 당, 0.1% 의 이자율이 적용되고, 12초에 1 블록이 생성되는 EVM 환경에서 배포됨. 이에, 1 블록당 이자율은 아래와 같은 식으로 계산됨.

$$ 1.001^{(1/7200)}×10^{18} = 1.00000013881950034147221897264156541137271898967894423311848... × 10^{18} $$

이에, `blockInterest` 변수 값을 `100000013881950034` 으로 하드코드하는 방식으로 설정할 수 있으나, 좀 더 *Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요함

#### Impact

**Informational**

아주 낮은 경우로 이자율의 괴리 발생.

#### Recommendation

*Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요.
