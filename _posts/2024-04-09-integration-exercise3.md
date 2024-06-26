---
title: 통합 실습 3 - NAT 포트포워딩을 통한 MariaDB, DNS, IIS, FTP 및 원격 데스크톱 연결
excerpt: "윈도우 2003 서버에서 NAT 포트포워딩을 진행해 사설망 안의 MariaDB, DNS, IIS, FTP 서버들에 접속하고 원격 데스크톱으로 연결한다."
author: minyeokue
date: 2024-04-09 19:08:51 +0900
last_modified_at: 2024-04-13 16:42:13 +0900
categories: [Exercise]
tags: [Linux, Windows, Firewall, MariaDB, Secure, Network, DNS]

toc: true
toc_sticky: true
---

<br>

윈도우 2003 서버에서 NAT 포트포워딩을 진행해 사설망 안의 MariaDB, DNS, IIS, FTP 서버들에 접속하고 원격 데스크톱으로 연결한다.

<br>

---

## 시나리오

<br>

- Win2003

    - [NAT 서버 설정](#nat-서버-설정)

        - 사설망 게이트웨이 : 10.10.10.253/24

        - 공인망 : 192.168.1.99  GW : 192.168.1.2  DNS : 8.8.8.8

            - [NAT 서버 포트포워딩](#nat-서버-포트포워딩) => [MariaDB](#mariadb-서버-테스트), [DNS](#dns-서버-테스트), [IIS](#윈도우-iis-서버-테스트), [FTP](#ftp-서버-테스트) 서버 접근 및 [원격 데스크톱 연결](#원격-데스크톱-연결-테스트)
            
- CentOS8

    - IP : 10.10.10.10/24  GW : 10.10.10.253

    - [DNS 서버 설정](#dns-서버-설정)

        - 도메인명 : minyeokue.gitblog

        - 호스트 생성 : www.minyeokue.gitblog
    
    - [MariaDB 서버 설정](#mariadb-서버-설정)

        - 로컬 DB 접속 시 root 계정 비밀번호 : 1234

        - 원격 DB 접속 시 root 계정 비밀번호 : 4321

        - 데이터베이스 생성 : gitblog

- DC1

    - IP : 10.10.10.20/24  GW : 10.10.10.253

    - [윈도우 IIS 서버 설정](#윈도우-iis-서버-설정)

    - [FTP 서버 설정](#ftp-서버-설정)

        - FTP 서버 홈 디렉토리 : C:\ftpdata

        - minyeokue 사용자 생성 => FTP 서버 읽기, 쓰기 권한 부여

        - gitblog 사용자 생성 => FTP 서버 읽기 권한 부여

        - 방화벽 인바운드 규칙 생성
    
    - [원격 데스크톱 연결 설정](#원격-데스크톱-연결-설정)

        - 포트번호 3389에서 35000으로 변경

        - 방화벽 인바운드 규칙 생성

---

<br>

### NAT 서버 설정

<br>

먼저 공인망과 사설망 IP가 구분되어 있기 때문에, 랜카드를 하나 더 붙여준다.

![NAT 서버 랜카드 추가 1](/assets/img/2024-04-09/1.png)
_NAT 서버 랜카드 추가 1_

![NAT 서버 랜카드 추가 완료](/assets/img/2024-04-09/2.png)
_NAT 서버 랜카드 추가 완료_

위 사진처럼 *Add...* 버튼을 누른 뒤 *Network Adapter*를 선택하고 **Finish** 버튼을 누른다.

<br>

가상머신을 실행시키고 네트워크 설정을 변경한다.

<br>

![NAT 서버 IP 설정 1](/assets/img/2024-04-09/3.png)
_NAT 서버 IP 설정 1_

*내 네트워크 환경*을 우측 마우스 클릭 -> **속성** 메뉴를 선택해 *LAN 또는 고속 인터넷* 창을 연다.

<br>

먼저 ~~가상의~~ 공인망 IP를 설정한다. 

![NAT 서버 IP 설정 2](/assets/img/2024-04-09/4.png)
_NAT 서버 IP 설정 2_

로컬 영역 설정을 *더블 클릭* 혹은 *오른쪽 마우스 클릭* -> **속성** -> *인터넷 프로토콜(TCP/IP)* 더블 클릭

<br>

![NAT 서버 공인망 IP 설정](/assets/img/2024-04-09/5.png)
_NAT 서버 공인망 IP 설정_

위 사진처럼 설정해둔 뒤 **확인** -> **확인** 버튼을 연달아서 누른다.

<br>

![NAT 서버 IP 설정 3](/assets/img/2024-04-09/6.png)
_NAT 서버 IP 설정 3_

새로 붙인 랜카드에 대한 설정을 진행한다. 사설망의 게이트웨이로 설정할 것이다.

<br>

![NAT 서버 사설망 게이트웨이 설정](/assets/img/2024-04-09/7.png)
_NAT 서버 사설망 게이트웨이 설정_

위 사진처럼 설정하고 **확인** 버튼을 누르면 DNS 목록이 비어있어 기본 DNS 서버 주소로 설정된다는 알림창이 팝업된다. **확인** -> **확인** 버튼을 연달아서 누르면 된다.

<br>

IP 설정은 끝났지만, 알아보기 쉽도록 각각의 로컬 영역 연결의 이름을 "공인망", "사설망 게이트웨이"로 변경한다.

![로컬 영역 연결 이름 변경](/assets/img/2024-04-09/8.png)
_로컬 영역 연결 이름 변경_

<br>

NAT 서버의 역할을 할 수 있도록 기능을 추가하기 위해 *사용자 서버 관리* 창으로 향한다. 해당 메뉴의 위치는 다음 사진을 참조한다.

![사용자 서버 관리 위치](/assets/img/2024-04-09/9.png)
_사용자 서버 관리 위치_

<br>

![사용자 서버 관리 역할 추가](/assets/img/2024-04-09/10.png)
_사용자 서버 관리 역할 추가_

위 사진처럼 강조된 부분을 클릭해 *서버 구성 마법사*를 진행한다.

<br>

![서버 구성 마법사 1](/assets/img/2024-04-09/11.png)
_서버 구성 마법사 1_

**다음** 버튼을 누른다.

<br>

![서버 구성 마법사 2](/assets/img/2024-04-09/12.png)
_서버 구성 마법사 2_

NAT 및 일종의 라우팅 역할을 하게 하기 위해 *원격 액세스 및 VPN 서버*를 선택한 뒤, **다음** -> **다음** 버튼을 누른다.

<br>

![라우팅 및 원격 액세스 서버 설치 마법사 1](/assets/img/2024-04-09/13.png)
_라우팅 및 원격 액세스 서버 설치 마법사 1_

*라우팅 및 원격 액세스 서버 설치 마법사* 창이 팝업되며, **다음** 버튼을 눌러서 계속 진행한다.

<br>

![라우팅 및 원격 액세스 서버 설치 마법사 2](/assets/img/2024-04-09/14.png)
_라우팅 및 원격 액세스 서버 설치 마법사 2_

*네트워크 주소 변환(NAT)* 라디오 버튼을 선택한 뒤, **다음** 버튼을 누른다.

<br>

![라우팅 및 원격 액세스 서버 설치 마법사 3](/assets/img/2024-04-09/15.png)
_라우팅 및 원격 액세스 서버 설치 마법사 3_

*~~가상의~~ 공인망*을 선택한 뒤, **다음** 버튼을 누른다.

![라우팅 및 원격 액세스 서버 설치 마법사 완료](/assets/img/2024-04-09/16.png)
_라우팅 및 원격 액세스 서버 설치 마법사 완료_

<br>

![서버 구성 마법사 완료](/assets/img/2024-04-09/17.png)
_서버 구성 마법사 완료_

*라우팅 및 원격 액세스 서버 설치 마법사* 창에서 **마침** 버튼을 누른 뒤, 잠시 기다리면 *서버 구성 마법사*에서도 **마침** 버튼을 누를 수 있게 된다.

<br>

이제 NAT 서버 구성을 마쳤으니 사설망(10.10.10.0/24) 대역에서 인터넷 연결이 가능해진 상태이다.

<br>

---

### DNS 서버 설정

<br>

먼저 CentOS8의 IP 및 호스트 이름의 설정을 변경하겠다.

`nmtui` 명령어를 입력한다.

![Network Manager Text User Interface](/assets/img/2024-04-09/18.gif)
_Network Manager Text User Interface_

리눅스 서버의 IP 및 호스트 이름 설정을 변경했다.

네트워크 랜카드를 재활성화하는 명령어로는 `nmcli con up [랜카드 이름]` 명령어로 변경한 IP를 적용시킬 수 있다.

> 호스트 이름을 변경하는 다른 방법은 `hostnamectl set-hostname [변경할 이름]` 명령어를 입력한 뒤, `exit` 명령어로 다시 로그인하면 적용된다.
{: .prompt-tip }

<br>

NAT 서버의 게이트웨이를 통해 외부 인터넷과 통신이 가능한지 확인한다.

![NAT 게이트웨이를 통한 외부 인터넷 연결 테스트](/assets/img/2024-04-09/19.gif)
_NAT 게이트웨이를 통한 외부 인터넷 연결 테스트_

정상적으로 진행되니, 이제 DNS 서버를 구축하기 위한 패키지를 설치해보겠다.

<br>

`yum -y install caching-nameserver vim` 명령어를 입력한다.

DNS 서버 관련 패키지들과 vim 패키지가 설치될 것이다.

DNS 서버 설정 파일을 수정해주도록 한다. `vim /etc/named.conf` 명령어를 입력한다.

![/etc/named.conf 수정](/assets/img/2024-04-09/20.png)
_/etc/named.conf 수정_

<br>

`/etc/named.rfc1912.zones`{: .filepath } 파일을 수정해 영역 파일에 관련된 설정을 진행한다.

![/etc/named.rfc1912.zones 수정](/assets/img/2024-04-09/21.png)
_/etc/named.rfc1912.zones 수정_

<br>

영역 파일을 복사해서 수정할 것이다. 기본이 되는 영역 파일은 `/var/named/named.localhost`{: .filepath }으로 이 파일의 권한과 함께 복사해서 `minyeokue.gitblog.zone`{: .filepath } 파일을 만들 것이다.

![minyeokue.gitblog.zone 파일 생성](/assets/img/2024-04-09/22.png)
_minyeokue.gitblog.zone 파일 생성_

<br>

![minyeokue.gitblog.zone 파일 수정](/assets/img/2024-04-09/23.png)
_minyeokue.gitblog.zone 파일 수정_

DNS 서버는 사설망인 10.10.10.10/24에 있지만, 공인망 IP를 적은 이유는 **SNAT(Source Network Address Translation)**을 하기 위함이다.

> **SNAT(Source NAT)**는 사설망에 존재하는 서버의 주소를 NAT의 공인망 주소로 변경하는 것을 뜻하며, 외부에서는 사설망의 존재를 모르고 공인망 주소를 해당 서비스를 하는 서버로 인식하기 때문에 SNAT를 사용한다. 또한 **DNAT(Destination Network Address Translation)**와 함께 적용되어야 외부에서의 응답을 사설망의 서버로 보낼 수 있다. 그렇기 때문에 **DNAT**와 포트포워딩이 거의 같은 의미로 통한다.
{: .prompt-info }

<br>

영역 파일 설정이 완료되면, `systemctl enable --now named` 명령어를 입력해 시스템이 재부팅되어도 서비스를 자동으로 시작할 수 있도록 설정한다.

서비스를 재시작하고 싶다면 `systemctl restart [서비스명]` 명령어를 입력한다.

정상적으로 링크파일이 생성되었다면 DNS 설정은 끝나게 된다.

<br>

---

### MariaDB 서버 설정

<br>

MariaDB 서버 구축을 위한 패키지를 설치한다. `yum -y install mariadb-server` 명령어를 입력한다.

![MariaDB 설정 데이터베이스 확인](/assets/img/2024-04-09/24.png)
_MariaDB 설정 데이터베이스 확인_

현재 MariaDB의 관리 데이터베이스를 확인했다.

<br>

> MariaDB 기본 보안 설정은 `mysql_secure_installation` 명령어를 통해 진행할 수 있으며, 이후 계정명과 비밀번호을 입력해야 접속할 수 있다.
{: .prompt-info }

"gitblog" 데이터베이스를 생성하고, 내부 테스트를 위해 MariaDB 서버에 접속할 때 root 계정으로 암호 '1234'를 입력하고, 외부에서 root 계정으로 접속할 때 암호 '4321'을 입력하도록 설정한다.

권한은 "gitblog"의 모든 테이블로 한정한다.

![MariaDB 데이터 베이스 생성 및 접속 설정](/assets/img/2024-04-09/25.png)
_MariaDB 데이터 베이스 생성 및 접속 설정_

`FLUSH PRIVILEGES;` SQL문으로 변경된 권한 설정을 적용한다.

~~root 계정으로의 접속은 보안 관점에서 봤을 때 바람직하지 않지만, 실습 테스트를 위해 진행한다.~~

<br>

CentOS8에서 진행해야 할 설정은 끝났다.

<br>

---

### 윈도우 IIS 서버 설정

<br>

![윈도우 서버 IP 설정](/assets/img/2024-04-09/26.png)
_윈도우 서버 IP 설정_

먼저 윈도우 서버의 IP를 설정한다.

<br>

![역할 및 기능 추가](/assets/img/2024-04-09/27.png)
_역할 및 기능 추가_

*관리* 탭의 *역할 및 기능 추가* 메뉴를 선택한다.

<br>

![역할 및 기능 추가 마법사 1](/assets/img/2024-04-09/28.png)
_역할 및 기능 추가 마법사 1_

*서버 역할* 단계까지 **다음** 버튼을 누른다.

<br>

![역할 및 기능 추가 마법사 2](/assets/img/2024-04-09/29.png)
_역할 및 기능 추가 마법사 2_

강조된 **웹 서버(IIS)**를 클릭한다.

<br>

![역할 및 기능 추가 마법사 3](/assets/img/2024-04-09/30.png)
_역할 및 기능 추가 마법사 3_

팝업된 웹 서버에 대한 **기능 추가** 버튼을 누른다.

완료되면 다음 상태가 된다.

![역할 및 기능 추가 마법사 4](/assets/img/2024-04-09/31.png)
_역할 및 기능 추가 마법사 4_

**다음** 버튼을 누른다.

<br>

![역할 및 기능 추가 마법사 5](/assets/img/2024-04-09/32.png)
_역할 및 기능 추가 마법사 5_

*웹 서버 역할(IIS) -> 역할 서비스* 단계까지 **다음** 버튼을 누르고, 보안 토글의 모든 항목을 체크한다.

이후 스크롤을 내린다.

![역할 및 기능 추가 마법사 6](/assets/img/2024-04-09/33.png)
_역할 및 기능 추가 마법사 6_

*HTTP 리디렉션*을 체크하고, *FTP 서버*를 체크하면 FTP 서비스가 자동으로 체크된다.

**다음** 버튼을 누른다.

<br>

![역할 및 기능 추가 마법사 7](/assets/img/2024-04-09/34.png)
_역할 및 기능 추가 마법사 7_

확인 단계에서 **설치** 버튼을 누른다.

조금 기다리면 체크했던 기능과 역할을 수행할 수 있도록 설치가 완료된다.

![역할 및 기능 추가 마법사 완료](/assets/img/2024-04-09/35.png)
_역할 및 기능 추가 마법사 완료_

> Window + R 키로 *실행* 창을 호출한 뒤 `inetmgr` 명령어를 입력하면 *IIS 관리자* 창을 실행할 수 있다.
{: .prompt-info }

이제 DC1은 IIS 서버와 FTP 서버의 역할을 수행할 수 있다.

<br>

FTP 서버에 대한 설정을 시작한다. 파일 탐색기를 실행시킨다.

![웹 서버 기본 경로](/assets/img/2024-04-09/36.png)
_웹 서버 기본 경로_

![웹 서버 기본 문서 저장 경로](/assets/img/2024-04-09/37.png)
_웹 서버 기본 문서 저장 경로_

`C:\inetpub\wwwroot\`{: .filepath }안의 모든 파일을 삭제한다.

이후 Window + R -> notepad로 메모장을 실행시킨다.

![웹 서버 기본 문서 작성](/assets/img/2024-04-09/38.png)
_웹 서버 기본 문서 작성_

반드시 *파일 형식*을 **모든 파일**로 선택한 뒤, **저장** 버튼을 누른다.

<br>

오늘 실습에서 IIS 관리자에서 진행해야할 모든 과정이 끝났다.

IIS 서버 설정에는 많은 설정들이 있지만, 다음에 다루도록 하겠다.

IIS 관리자에서 FTP 서버 관련 설정을 조작할 수 있기 때문에 이어서 진행한다.

<br>

### FTP 서버 설정

<br>

먼저 **FTP 서버에 접속하기 위한 사용자를 생성**하도록 한다.

Window + R 키로 *실행* 창을 띄우고, `cmd` 명령어로 명령 프롬프트를 실행시킨다.

명령 프롬프트에서 `net user [사용자명] [비밀번호] /add` 명령어를 입력해 원하는 사용자를 추가한다.

<br>

*컴퓨터 관리* 창에서 로컬 사용자를 추가하는 방법이 존재한다.

![컴퓨터 관리 위치](/assets/img/2024-04-09/39.png)
_컴퓨터 관리 위치_

위 사진처럼 *서버 관리자*에서 진행하는 방법과 Window + R 키로 *실행* 창을 띄워서 `compmgmt.msc` 명령어로 *컴퓨터 관리* 창을 실행시킬 수 있다.

또는 *실행* 창에서 `lusrmgr.msc` 명령어로 *로컬 사용자 및 그룹* 창을 실행시킬 수 있다.

<br>

![컴퓨터 관리 - 사용자 추가 1](/assets/img/2024-04-09/40.png)
_컴퓨터 관리 - 사용자 추가 1_

*로컬 사용자 및 그룹*에서 *사용자*를 추가한다. **새 사용자** 메뉴를 선택한다.

![컴퓨터 관리 - 사용자 추가 2](/assets/img/2024-04-09/41.png)
_컴퓨터 관리 - 사용자 추가 2_

원하는 사용자명과 비밀번호를 입력한 뒤, **다음 로그온 시 사용자가 반드시 암호를 변경해야 함** 옵션을 **체크 해제**하고 **만들기** 버튼을 누른다.

<br>

사용자는 총 2명으로 minyeokue 사용자와 gitblog 사용자를 생성하였다고 가정하고 진행한다.

사용자 생성이 완료되었으니, 이제 FTP 서버에 대한 설정을 진행한다.

<br>

![IIS(인터넷 정보 서비스) 관리자 위치](/assets/img/2024-04-09/42.png)
_IIS(인터넷 정보 서비스) 관리자 위치_

*도구* 탭에서 **IIS(인터넷 정보 서비스) 관리자** 메뉴를 선택한다.

> Window + R 키를 눌러 나오는 실행창에서 `inetmgr` 명령어를 입력하면 IIS 관리자 창을 실행시킬 수 있다.
{: .prompt-info }

<br>

![FTP 사이트 추가 위치](/assets/img/2024-04-09/43.png)
_FTP 사이트 추가 위치_

*DC1(DC1\Administrator)* 아래 *사이트*를 우측 마우스 클릭 혹은 *작업* 탭에서 **FTP 사이트 추가** 메뉴를 선택한다.

<br>

![FTP 사이트 추가 - 사이트 정보 1](/assets/img/2024-04-09/44.png)
_FTP 사이트 추가 - 사이트 정보 1_

*FTP 사이트 이름*을 "FTP 서버"로 설정하고 *콘텐츠 디렉터리 -> 실제 경로*를 **...** 버튼을 눌러 지정해준다.

<br>

![FTP 사이트 추가 - 사이트 정보 2](/assets/img/2024-04-09/45.png)
_FTP 사이트 추가 - 사이트 정보 2_

`C:\`{: .filepath } 아래에 "ftpdata" 라는 폴더를 생성해서 선택 후 **확인** 버튼을 누른다.

실제 경로가 `C:\ftpdata`{: .filepath }로 설정된 것을 확인한 뒤, **다음** 버튼을 누른다.

<br>

![FTP 사이트 추가 - 바인딩 및 SSL 설정](/assets/img/2024-04-09/46.png)
_FTP 사이트 추가 - 바인딩 및 SSL 설정_

*SSL* 메뉴에서 **SSL 사용 안 함** 라디오 버튼을 체크한 뒤, **다음** 버튼을 누른다.

<br>

![FTP 사이트 추가 - 인증 및 권한 부여 정보](/assets/img/2024-04-09/47.png)
_FTP 사이트 추가 - 인증 및 권한 부여 정보_

*인증*은 **기본**을 체크, *권한 부여*에서 *액세스 허용* 드롭다운 메뉴에서 **지정한 사용자**를 선택한 뒤, 원하는 사용자 계정의 이름을 입력한다. -> minyeokue 사용자

그리고 사용 권한에 **읽기**와 **쓰기** 권한을 부여하도록 체크 한 뒤, **마침** 버튼을 누른다.

<br>

이제 gitblog 사용자에게 읽기 권한을 부여하기 위해 다음의 단계를 따른다.

![FTP 서버 - FTP 권한 부여 규칙 위치](/assets/img/2024-04-09/48.png)
_FTP 서버 - FTP 권한 부여 규칙 위치_

위 사진처럼 *FTP 권한 부여 규칙* 메뉴를 더블 클릭한다.

<br>

![FTP 서버 - FTP 권한 부여 규칙](/assets/img/2024-04-09/49.png)
_FTP 서버 - FTP 권한 부여 규칙 1_

*FTP 권한 부여 규칙* 메뉴에 들어오니, 읽기 권한과 쓰기 권한을 모두 가진 minyeokue 사용자가 보인다.

"gitblog" 사용자를 추가하기 위해 오른쪽 *작업* 탭에서 **허용 규칙 추가...** 메뉴를 선택한다.

<br>

![FTP 서버 - FTP 권한 부여 규칙 추가](/assets/img/2024-04-09/50.png)
_FTP 서버 - FTP 권한 부여 규칙 추가_

*지정한 사용자* 라디오 버튼을 선택한 뒤, 원하는 사용자 이름을 입력한다.

부여하고 싶은 권한을 선택한 뒤, **확인** 버튼을 누른다.

<br>

이제 *고급 보안이 포함된 Windows Defender 방화벽* 메뉴를 불러와야 한다.

FTP 서버는 21번 포트를 연결에 사용하고 데이터 전송에 20번 포트를 사용하기 때문에 해당 포트들을 방화벽에서 막지 않아야 하며, 원격 데스크톱 연결 또한 마찬가지이다.

![FTP 서버 - 방화벽 설정 위치](/assets/img/2024-04-09/51.png)
_FTP 서버 - 방화벽 설정 위치_

위 사진처럼 *서버 관리자* 창에서 *도구* 탭의 **고급 보안이 포함된 Windows Defender 방화벽** 메뉴를 선택한다.

또는 Window + R 키의 *실행* 창에서 `wf.msc`를 입력하거나, `firewall.cpl` -> **고급 설정** 탭 선택으로 **고급 보안이 포함된 Windows Defender 방화벽** 창을 실행시킬 수 있다.

<br>

#### 고급 보안이 포함된 Windows Defender 방화벽

<br>

공인망에서 DNAT로 포트 포워딩 되는 접근이 방화벽 입장에서 생각했을 때 *들어오는 신호*이기 때문에, 인바운드 규칙으로 FTP 서버와 원격 데스크톱 연결에 대해 차단하지 않도록 해야 한다.

먼저 FTP 포트를 설정한 뒤, 원격 데스크톱 연결의 포트를 설정하도록 한다.

![고급 보안이 포함된 Windows Defender 방화벽 인바운드 규칙](/assets/img/2024-04-09/52.png)
_고급 보안이 포함된 Windows Defender 방화벽 인바운드 규칙_

오른쪽 작업 탭에 **새 규칙**을 누른다.

<br>

![인바운드 규칙 추가 1](/assets/img/2024-04-09/53.png)
_인바운드 규칙 추가 1_

**포트**를 선택하고 **다음** 버튼을 누른다.

<br>

![인바운드 규칙 추가 2](/assets/img/2024-04-09/54.png)
_인바운드 규칙 추가 2_

포트 번호를 입력하는데, `20,21` 또는 `20-21`를 입력한다.

<br>

![인바운드 규칙 추가 3](/assets/img/2024-04-09/55.png)
_인바운드 규칙 추가 3_

**연결 허용**이 선택되어 있다. **다음** 버튼을 누른다.

<br>

![인바운드 규칙 추가 4](/assets/img/2024-04-09/56.png)
_인바운드 규칙 추가 4_

프로필 단계에서 **다음**을 선택한다.

<br>

![인바운드 규칙 추가 5](/assets/img/2024-04-09/57.png)
_인바운드 규칙 추가 5_

이름과 설명을 직접 적은 뒤, **마침** 버튼을 누른다.

FTP 서버의 설정이 모두 완료되었다.

<br>

---

### 원격 데스크톱 설정

<br>

이전 FTP 서버 설정에서 이어서 진행한다.

다른 시스템에서 DC1에 원격 데스크톱으로 접속할 수 있도록 설정해야 한다.

이 때 보안을 위해 본래 3389번 포트를 사용하는 원격 데스크톱 포트번호를 35000번으로 변경한다.

DC1 방화벽 설정에 35000번 포트로 들어오는 신호를 차단하지 않도록 인바운드 규칙을 설정하도록 한다.

![고급 보안이 포함된 Windows Defender 방화벽 인바운드 규칙](/assets/img/2024-04-09/58.png)
_고급 보안이 포함된 Windows Defender 방화벽 인바운드 규칙_

<br>

![인바운드 규칙 추가 6](/assets/img/2024-04-09/59.png)
_인바운드 규칙 추가 6_

**포트**를 선택하고 **다음** 버튼을 누른다.

<br>

![인바운드 규칙 추가 7](/assets/img/2024-04-09/60.png)
_인바운드 규칙 추가 7_

*특정 로컬 포트*에 "35000"을 입력하고, **다음** 버튼을 누른다.

<br>

![인바운드 규칙 추가 완료](/assets/img/2024-04-09/61.png)
_인바운드 규칙 추가 완료_

직접 이름과 설명을 입력한 뒤, **마침** 버튼을 누른다.

이제 외부에서 35000번 포트를 통해 들어오는 신호 등은 차단되지 않게 된다.

<br>

실질적으로 원격 데스크톱 연결 포트 번호를 변경하기 위해서는 레지스트리 값을 변경해야 한다.

Window + R 키로 *실행* 창에서 `regedit` 명령어를 입력해 *레지스트리 편집기*를 실행시킨다.

![실행 창 - 레지스트리 편집기](/assets/img/2024-04-09/62.png)
_실행 창 - 레지스트리 편집기_

<br>

![레지스트리 편집기 - 찾기](/assets/img/2024-04-09/63.png)
_레지스트리 편집기 - 찾기_

Ctrl + F 키를 입력해 *찾기* 창을 실행시키고, 찾을 내용에 "PortNumber"을 입력한 뒤 **다음 찾기** 버튼을 누른다.

`F3` 키를 누르면 **다음 찾기**를 진행한다.

![레지스트리 편집기 - 찾기 완료 및 변경](/assets/img/2024-04-09/64.gif)
_레지스트리 편집기 - 찾기 완료 및 변경_

3389번을 35000으로 변경하였다.

<br>

![원격 데스크톱 설정 위치](/assets/img/2024-04-09/65.png)
_원격 데스크톱 설정 위치_

윈도우 검색창에서 "원격"을 입력하면 위의 사진같은 상태가 된다. **원격 데스크톱 설정**을 선택한다.

<br>

![원격 데스크톱 설정 완료](/assets/img/2024-04-09/66.png)
_원격 데스크톱 설정 완료_

"끔"으로 되어있던 토글 키를 눌러 **켬** 상태로 전환한다.

<br>

원격 데스크톱 서비스를 실행시켰지만, 35000번 포트에 신호가 오는지 대기 중인지 확인해 보자. 

![원격 데스크톱 포트 변경 확인](/assets/img/2024-04-09/67.png)
_원격 데스크톱 포트 변경 확인_

명령 프롬프트를 실행시켜서 `netstat` 명령어를 통해 확인한 모습이다.

기존 3389번 포트에서 35000번으로 변경된 모습을 확인할 수 있다.

원격 데스크톱 설정이 완료되었다.

<br>

---

### NAT 서버 포트포워딩

<br>

다시 win2003 서버로 돌아온다.

![라우팅 및 원격 액세스 위치](/assets/img/2024-04-09/68.png)
_라우팅 및 원격 액세스 위치_

위 사진처럼 **라우팅 및 원격 액세스** 메뉴를 선택한다.

<br>

![공인망 - 서비스 및 포트 위치](/assets/img/2024-04-09/69.gif)
_공인망 - 서비스 및 포트 위치_

위 사진처럼 따라해서 **공인망**을 찾은 뒤 더블 클릭한다.

<br>

![공인망 - 서비스 및 포트 FTP 설정 위치](/assets/img/2024-04-09/70.png)
_공인망 - 서비스 및 포트 FTP 설정 위치_

*서비스 및 포트* 탭을 누른 뒤, **FTP 서버**를 더블 클릭한다.

![공인망 - 서비스 및 포트 FTP 설정 완료](/assets/img/2024-04-09/71.png)
_공인망 - 서비스 및 포트 FTP 설정 완료_

**확인** 버튼을 누른 뒤, *FTP 서버* 체크 박스를 체크한다.

<br>

![공인망 - 서비스 및 포트 웹 서버 설정 위치](/assets/img/2024-04-09/72.png)
_공인망 - 서비스 및 포트 웹 서버 설정 위치_

*웹 서버(HTTP)*를 더블 클릭한다.

![공인망 - 서비스 및 포트 웹 서버 설정 완료](/assets/img/2024-04-09/73.png)
_공인망 - 서비스 및 포트 웹 서버 설정 완료_

위 사진처럼 설정한 뒤, **확인** 버튼을 누른 뒤, *웹 서버(HTTP)* 체크 박스를 체크한다.

<br>

DNS 서버, MariaDB 서버, 원격 데스크톱 서비스에 대한 포트포워딩은 **추가**해서 진행해야 한다.

![공인망 - 서비스 및 포트 서비스 추가](/assets/img/2024-04-09/74.png)
_공인망 - 서비스 및 포트 서비스 추가_

위 사진에서 강조된 **추가** 버튼을 클릭한다.

<br>

MariaDB 서버 -> 원격 데스크톱 연결 -> DNS 서버 순서로 추가할 것이다.

![공인망 - 서비스 및 포트 MariaDB 추가](/assets/img/2024-04-09/75.png)
_공인망 - 서비스 및 포트 MariaDB 추가_

![공인망 - 서비스 및 포트 원격 데스크톱 연결 추가](/assets/img/2024-04-09/76.png)
_공인망 - 서비스 및 포트 원격 데스크톱 연결 추가_

<br>

DNS 서비스는 TCP 프로토콜과 UDP 프로토콜을 전부 사용하는 서비스이기 때문에 둘 다 추가해준다.

![공인망 - 서비스 및 포트 DNS (TCP) 추가](/assets/img/2024-04-09/77.png)
_공인망 - 서비스 및 포트 DNS (TCP) 추가_

![공인망 - 서비스 및 포트 DNS (UDP) 추가](/assets/img/2024-04-09/78.png)
_공인망 - 서비스 및 포트 DNS (UDP) 추가_

<br>

![공인망 - 모든 서비스 및 포트 추가 완료](/assets/img/2024-04-09/79.png)
_공인망 - 모든 서비스 및 포트 추가 완료_

시나리오의 모든 서비스를 NAT 포트포워딩한 상태이다.

<br>

---

### 테스트

<br>

모든 설정이 완료되었으니 테스트를 진행하도록 한다.

Linux01 -> CentOS7이며 192.168.1.10의 ~~가상의~~ 공인 IP를 가진 리눅스로 테스트를 하도록 하겠다.

<br>

#### MariaDB 서버 테스트

<br>

테스트를 진행하는 PC의 IP 설정을 먼저 진행한다.

![테스트 PC IP 설정](/assets/img/2024-04-09/80.png)
_테스트 PC IP 설정_

<br>

MariaDB 클라이언트가 되기 위해서 관련 패키지를 설치한다.

`yum -y install mariadb` 명령어를 터미널에 입력한다.

<br>

설치가 완료되면 NAT 서버 공인 IP 주소를 입력해 MariaDB 서버에 접속한다.

이 때, root 사용자로 접속하며 비밀번호는 '4321'로 한다. --> [MariaDB 서버 설정](#mariadb-서버-설정) 참조 

![테스트 PC - MariaDB 접속](/assets/img/2024-04-09/81.gif)
_테스트 PC - MariaDB 접속_

성공적으로 접속에 성공했다.

<br>

---

#### 윈도우 IIS 서버 테스트

<br>

Firefox 웹 브라우저에서 테스트를 진행한다.

![테스트 PC - 윈도우 웹 서버 접속](/assets/img/2024-04-09/82.gif)
_테스트 PC - 윈도우 웹 서버 접속_

정상적으로 접속되었다.

<br>

---

#### DNS 서버 테스트

<br>

DNS 서버를 설정해줘야 DNS 쿼리를 진행하기 때문에 `vim /etc/resolv.conf` 명령어를 입력해 임시 DNS 서버를 설정한다. => 재부팅시 없어지는 정보이다.

영구적인 DNS 설정을 위해서는 `nmtui` 명령어를 입력해 해당 부분을 수정하거나 `/etc/sysconfig/network-scripts/ifcfg-ens33`{: .filepath }를 수정한다.

![테스트 PC - DNS 서버 설정](/assets/img/2024-04-09/83.png)
_테스트 PC - DNS 서버 설정_

<br>

![테스트 PC - 윈도우 웹 서버 도메인 이름 접속](/assets/img/2024-04-09/84.gif)
_테스트 PC - 윈도우 웹 서버 도메인 이름 접속_

정상적으로 완료되었다.

<br>

---

#### FTP 서버 테스트

<br>

`yum -y install ftp` 명령어를 입력해 FTP 클라이언트로서 필요한 패키지를 설치한다.

![테스트 PC - FTP 서버 접속 테스트](/assets/img/2024-04-09/85.gif)
_테스트 PC - FTP 서버 접속 테스트_

<br>

알드라이브 프로그램을 통해서 minyeokue 사용자에게 읽기와 쓰기 권한이 있는지 확인하고, gitblog 사용자는 읽기 권한만 존재하는지 확인해보도록 하겠다.

![호스트 OS - FTP 서버 접속 테스트 1](/assets/img/2024-04-09/86.gif)
_호스트 OS - FTP 서버 접속 테스트 1_

위 사진에서 minyeokue 사용자가 FTP 서버에 올린 Test.txt 파일은 다음과 같다.

![Test.txt 내용](/assets/img/2024-04-09/87.png)
_Test.txt 내용_

minyeokue 사용자의 ~~읽기~~ 및 쓰기 권한을 확인했다.

<br>

![호스트 OS - FTP 서버 접속 테스트 2](/assets/img/2024-04-09/88.gif)
_호스트 OS - FTP 서버 접속 테스트 2_

minyeokue 사용자와는 다르게 쓰기 권한은 없는 것이 확인되었다.

이제 DC1 즉, FTP 서버에서 확인해보겠다.

![FTP 서버에서 직접 확인](/assets/img/2024-04-09/89.gif)
_FTP 서버에서 직접 확인_

정상적으로 완료되었다.

<br>

#### 원격 데스크톱 연결 테스트

<br>

호스트 OS(Window 11)에서 원격 데스크톱 연결을 시도하겠다.

![호스트 OS - 원격 데스크톱 연결 테스트](/assets/img/2024-04-09/90.gif)
_호스트 OS - 원격 데스크톱 연결 테스트_

정상적으로 완료되었다.