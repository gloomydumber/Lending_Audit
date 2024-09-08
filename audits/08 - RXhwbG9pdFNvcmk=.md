# RXhwbG9pdFNvcmk=

## Scope

<table>
  <tr>
    <th>Name</th>
    <td>Lending Security Audit</td>
  </tr>
  <tr>
    <th>Target / Version</th>
    <td>Git Repository (<a href="https://bit.ly/3Zit8rK" target="_blank">Lending_solidity</a>): commit <mark>75e65dc5d5670cdb8894f75da9d6b38defb5c893</mark> (<mark>main</mark> branch)</td>
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

### #1 `001` There is no explicit variable or calculation for the interest rate

|ID|Summary|Severity|
|:---:|-------|:---:|
|001|이자율에 관해 계산하기 위한 변수의 명시적인 선언 및 명확한 함수의 구성 파악이 어려움.|Informational|

#### Description

해당 Lending 컨트랙트는 24시간 당, 0.1% 의 이자율이 적용되고, 12초에 1 블록이 생성되는 EVM 환경에서 배포됨. 이에, 1 블록당 이자율은 아래와 같은 식으로 계산됨.

$$ 1.001^{(1/7200)}×10^{18} = 1.00000013881950034147221897264156541137271898967894423311848... × 10^{18} $$

이에, `calc_func()` 함수에서 `rate` 변수 값을 `1001` 으로 하드코드하여 구현하였으나, 명확한 이자율에 대한 변수 선언과 이에 대한 처리를 담당하는 함수와 함께 좀 더 *Floating Point* 이하 부분을 잘 반영하는 이자율을 적용할 수 있는 방법의 구현이 필요함. 또, 컨트랙트 전반적으로 변수명을 명확하기 알기 어렵고, 각 함수들이 어떠한 비지니스 로직으로 동작하는지 명시적으로 파악하기 난해함.

#### Impact

**Informational**

이자율의 괴리 발생.

#### Recommendation

*Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현을 포함한 Lending Protocol 구현의 전반적인 개선 요망.

### #2 `002` Success or failure is not checked in the *low-level call* within the `withdraw()` and `borrow()` function

|ID|Summary|Severity|
|:---:|-------|:---:|
|002|`withdraw()` 및 `borrow()` 함수에서, 유저에게 Ether를 출금 및 대출을 위해 전송할 때 최종적으로 *low-level call*을 이용하고 있는데, 성공 여부를 체크하지 않음.|Informational|

#### Description

`withdraw()` 및 `borrow()` 함수에서, 유저에게 Ether를 출금 및 대출을 위해 전송할 때 최종적으로 *low-level call*을 이용하고 있는데, 성공 여부를 체크하지 않음. 이는, 해당 *low-level call*의 *Silently Fail*로 인해 유저가 정상적으로 Ether를 반환받지 못할 수 있고, 해당 트랜잭션의 Gas Estimation을 어렵게 할 수 있음.

#### Impact

**Informational**

이용자의 예기치 못한 출금 실패 및 자금 손실.

#### Recommendation

`(bool success, )` 및 `require(success)` 등의 성공 여부 체크 로직 도입 요망.