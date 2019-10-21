# commercial-paper

* 참조 사이트 : https://hyperledger-fabric.readthedocs.io/en/latest/tutorial/commercial_paper.html
				https://ovila.tistory.com/66
				
1. 환경 셋팅
1.1 go Path 확인 및 설정
$ env
GOPATH=/Users/username/go


2. 서비스 시나리오
MagnetoCorp사의 직원인 Isabella가 커머셜 페이퍼(cp)를 발행하면 DigiBank의 직원인 Balaji가 cp를 구매하고, 
cp를 일정 기간동안 보유하고 있다가 다시 MagnetoCorp로 환급하며 약간의 이익을 얻는 시나리오.

3. 네트워크 구성
$ cd fabric-samples/basic-network
$ ./start.sh

$ docker ps -a
- peer, orderer, couchdb, ca 네개의 컨테이너 생성
- 이 컨테이너 들은 net_basic이라는 도커 네트워크를 형성

$ docker network inspect net_basic
- 커맨드를 통해 네트워크의 세부사항을 확인

4. [Working as MagnetoCorp]
logspout을 이용하면 도커 컨테이너로부터 생성되는 출력결과물을 확인할 수 있다. 
관리자가 스마트 컨트랙트를 설치하거나 개발자가 스마트 컨트랙트를 실행시키고자 할 때 유용하다.

4.1 MagnetoCorp의 관리자로서 PaperNet 네트워크 모니터링 방법.(logspout실행)
새로운 창에서 아래 명령어 실행.
$ cd commercial-paper/organization/magnetocorp/configuration/cli/
$ ./monitordocker.sh net_basic

MagnetoCorp는 도커컨테이너를 통해 네트워크와 통신방법
$ cd commercial-paper/organization/magnetocorp/configuration/cli/
$ docker-compose -f docker-compose.yml up -d cliMagnetoCorp
=> docker-dompose 명령을 통해 MagnetoCorp의 관리자를 위한 도커컨테이너를 실행
=> docker hub에서 도커이미지를 가져와서 네트워크에 추가된 것을 확인
=> Contain ID(ex:efb1a7532e7e)로 PaperNet과 통신

5. [체인코드 설치]
MagnetoCorp의 관리자는 peer chaincode install 명령어를 이용하여 
papercontract(스마트 컨트랙트)를 자신의 로컬 파일 시스템으로부터 복사하여 목표로 하는 피어의 도커컨테이너에 복사

$ docker exec cliMagnetoCorp peer chaincode install -n papercontract -v 0 -p /opt/gopath/src/github.com/contract -l node
=> docker exec 커맨트를 이용하여 cliMagnetCorp 컨테이너에서 peer chaincode install 명령어를 실행
=> MagnetoCorp 관리자는 papercontract를 단지 하나의 피어에만 설치
=> –p 다음의 경로는 /opt/gopath/src/github.com/contract이다. 이 경로는 MagnetoCorp의 로컬 파일 시스템인 .../organization/magnetocorp/contract과 연결
(참조 소스 : magnetocorp/configuration/cli/docker-compose.yml의 volumes : /../../../../organization/magnetocorp:/opt/gopath/src/github.com/ 매핑됨)

6. [인스턴스화]
papercontract를 실행하기 위해 새로운 도커 체인코드 컨테이너 생성

$ docker exec cliMagnetoCorp peer chaincode instantiate -n papercontract -v 0 -l node -c '{"Args":["org.papernet.commercialpaper:instantiate"]}' -C mychannel -P "AND ('Org1MSP.member')"
=> MagnetoCorp의 관리자가 mychannel에 papercontract를 인스턴스화.
=> -p 는 papercontract의 보증 정책을 정의
=> 오더러의 address인 orderer.example.com:7050을 전달

$ docker ps -a
=> papercontract의 버전은 0

7. [어플리케이션 구조]
7.1 시나리오 흐름
- isabella 이 지갑에서 x.509 인증서를 획득
- issue 어플리케이션은 채널에서 트랜잭션을 전송하기 위해 게이트웨이를 사용

$ cd commercial-paper/organization/magnetocorp/application/
$ npm install
$ node addToWallet.js
$ ls ../identity/user/isabella/wallet/
- 이사벨라의 인증서가 로컬 파일 시스템으로부터 wallet으로 옮겨졌다.
- 이사벨라가 사용하는 서로 다른 인증서들은 각자만의 폴더를 갖는다.
- c75bd6911a...-priv 는 개인키이며 서명에 사용된다. 이 키는 배포되지 않는다.
- c75bd6911a...-pub는 공개키이며 이사벨라의 x.509인증서에 포함되어 있다.
- User1@org.example.com는 공개키와 인증서의 속성이 포함되어 있다. 네트워크로 배포된다.

8.[Issue 어플리케이션 (Isabella)]
$ node issue.js
- MagnetoCorp의 커머셜페이퍼 00001을 발행하는 트랜잭션을 제출, value of 5M USD.

9.[Working as DigiBank working (balaji)]
- MagnetoCorp의 커머셜페이퍼 00001을 구매하기 위해 buy어플리케이션을 이용

(digibank admin)$ cd commercial-paper/organization/digibank/configuration/cli/
(digibank admin)$ docker-compose -f docker-compose.yml up -d cliDigiBank
$ docker ps -a (생성된 cliDigiBank 도커컨테이너 확인)

9.[Digibank 어플리케이션]
balaji는 DigiBank의 buy 어플리케이션을 이용하여 cp 00001의 소유자를 DigiBank로 바꾸는 트랜잭션을 제출

9.1 Wallet에 인증서 생성
(balaji)$ cd commercial-paper/organization/digibank/application/
(digibank admin)$ cd commercial-paper/organization/digibank/application/
(digibank admin)$ npm install
(balaji)$ node addToWallet.js (wallet 안에 인증서 생성 확인)

9.2 Buy application
balaji는 buy.js 사용하여 MagnetoCorp의 기업어음 00001의 소유권을 digibank로 양도 할 거래로 트랜젝션을 제출

(balaji)$ node addToWallet.js

9.3 Redeem application
digibank가 MagnetoCorp에게 기업어음 00001을 상환하는 것.(소유권이 MagnetoCorp에게 정상적으로 이동됨을 확인)

(balaji)$ node redeem.js

