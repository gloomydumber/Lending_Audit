# TnVsbDBSTQ==

## Scope

<table>
  <tr>
    <th>Name</th>
    <td>Lending Security Audit</td>
  </tr>
  <tr>
    <th>Target / Version</th>
    <td>Git Repository (<a href="https://bit.ly/4d0fBbt" target="_blank">Lending_solidity</a>): commit <mark>ef1a322f73e57207c0a8b8f42a6e74ba3a14d4f6</mark> (<mark>main</mark> branch)</td>
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

이에, `shortNumerator` 변수 값을 `10000001388195003` 으로 하드코드하여 구현하였으나, 좀 더 *Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요함

#### Impact

**Informational**

이자율의 괴리 발생.

#### Recommendation

*Floating Point* 이하 부분을 잘 반영하여 이자율을 적용할 수 있는 방법의 구현이 필요.

### #2 `002` No Mechanism to Remove Inactive `users`

|ID|Summary|Severity|
|:---:|-------|:---:|
|002|더 이상 해당 프로토콜을 사용하지 않는 유저가 `users` 변수에 삭제되지 않고 영구히 저장 됨.|Informational|

#### Description

유저가 유동성을 공급하게 되면, 해당 유저의 관리를 `users` 가변 배열 변수에서 `address` 타입으로 저장하여 관리하게 되는데, 더 이상 해당 프로토콜을 사용하지 않는 유저가 `users` 변수에 삭제되지 않고 영구히 저장 됨. 해당 변수는 `_depositUpdate()` 함수 등에서 지속적으로 불러와 반복문으로 순회하게 되는데, 해당 변수에 많은 유저의 주소가 저장되면 해당 컨트랙트 이용에 점점 많은 가스비가 소모될 수 있음.

#### Impact

**Informational**

이용자가 증가할 수록 컨트랙트 스토리지 사용 증대로 인해 가스비 증가 및 부작용 발생.

#### Recommendation

특정 유저의 유동성 공급 등이 종료되면 `users` 변수에서 해당 유저의 주소를 제거.