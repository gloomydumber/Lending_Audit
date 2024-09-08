# Y2hvaWhzMDQ1Nw==

## Scope

<table>
  <tr>
    <th>Name</th>
    <td>Lending Security Audit</td>
  </tr>
  <tr>
    <th>Target / Version</th>
    <td>Git Repository (<a href="https://bit.ly/4cWEapt" target="_blank">Lending_solidity</a>): commit <mark>175e80d51b44fa98538c24a1ad0121f43b7ab7c4</mark> (<mark>main</mark> branch)</td>
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

### #1 `001` The interest rate is hard-coded

|ID|Summary|Severity|
|:---:|-------|:---:|
|001|이자율이 하드코드되어 있음.|Informational|

#### Description

해당 Lending 컨트랙트는 24시간 당, 0.1% 의 이자율이 적용되고, 12초에 1 블록이 생성되는 EVM 환경에서 배포됨. 이에, 1 블록당 이자율은 아래와 같은 식으로 계산됨.

$$ 1.001^{(1/7200)}×10^{18} = 1.00000013881950034147221897264156541137271898967894423311848... × 10^{18} $$

이에, `INTEREST_RATE_PER_BLOCK` 변수 값을 `1000000138822311089` 으로 하드코드하여 구현하였으나, 좀 더 *Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요함

#### Impact

**Informational**

이자율의 괴리 발생.

#### Recommendation

*Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요.

### #2 `002` No Mechanism to Remove Inactive `lender` and `borrower`

|ID|Summary|Severity|
|:---:|-------|:---:|
|002|더 이상 해당 프로토콜을 사용하지 않는 유저가 `lender` 및 `borrower` 변수에 삭제되지 않고 영구히 저장 됨.|Informational|

#### Description

유저가 유동성을 공급하거나 대출을 받게 되면, 해당 유저의 관리를 `lender` 및 `borrower` 가변 배열 변수에서 `address` 타입으로 저장하여 관리하게 되는데, 더 이상 해당 프로토콜을 사용하지 않는 유저가 `lender` 및 `borrower` 변수에 삭제되지 않고 영구히 저장 됨. 해당 변수는 `accrueInterest()` *modifier* 등에서 지속적으로 불러와 반복문으로 순회하게 되는데, 해당 변수에 많은 유저의 주소가 저장되면 해당 컨트랙트 이용에 점점 많은 가스비가 소모될 수 있음.

#### Impact

**Informational**

이용자가 증가할 수록 컨트랙트 스토리지 사용 증대로 인해 가스비 증가 및 부작용 발생.

#### Recommendation

특정 유저의 유동성 공급 또는 대출이 종료되면 `depositList` 및 `debtList` 변수에서 해당 유저의 주소를 제거.

### #3 `003` The `transfer` method is used to send Ether within the `withdraw()` and `liquidate()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|003|`withdraw()` 및 `liquidate()` 함수에서, 유저에게 Ether를 출금할 때 최종적으로 `transfer`를 이용하고 있는데, 2300 가스 제한이 있어 실패할 수 있음.|Informational|

#### Description

`withdraw()` 및 `liquidate()` 함수에서, 유저에게 Ether를 출금할 때 최종적으로 `transfer`를 이용하고 있는데, `transfer`의 특성상 2300의 가스 제한이 있어 실패할 수 있음. 가령, 해당 컨트랙트의 이용자가 `receive()` 및 `fallback()` 등에서 특정 동작이 필요한 경우에, 예기치 못한 출금 실패 및 자금 손실이 발생할 수 있음.

#### Recommendation

`call()`을 통해 *low-level call* 도입 및 해당 *low-level call*의 성공 여부 체크 로직 도입 요망.

### #4 `004` Token address confirmation is not performed in the `borrow()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|004|`borrow()` 함수에서 대출하고자 하는 토큰이 *USDC* 토큰 주소임을 확인하지 않음.|Informational|

#### Description

이용자가 `borrow()` 함수를 통해 해당 함수의 `address` 타입의 인자 `token`의 값으로 이용자 부주의로 인해 *USDC* 토큰의 주소가 아닌 다른 토큰을 입력했을 시, 해당 이용자의 *USDC* 토큰의 부채 관련 값은 증가하나, 의도되지 않은 토큰이 이용자에게 전달되는 등, 예기치 못한 상황이 발생할 수 있음. 또 이후 `liquidate()` 함수를 통해 악의적인 공격이 더욱 진행될 수 있음.

#### Impact

**Informational**

악의적인 유저에 의해 부적절한 ERC20 Compatible 토큰이 해당 Lending Protocol로 입금되거나, 이용자의 과실로 인해 잘못된 *USDC* 토큰의 채무 관련 값이 저장이 되는 등 예기치 못한 상황 발생 가능. 이후 `liquidate()` 함수를 통해 부적절한 청산 과정 및 프로토콜 자산 탈취 위험.

#### Recommendation

`require()` 문 등을 통해, 이용자가 전달한 `address` 타입의 `token` 인자의 값이 *USDC* 토큰과 일치하는지 확인 요망.