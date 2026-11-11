# SSO 연동 (관리 화면에서 설정하기)

이 문서는 사내 인증 시스템(IdP — Keycloak, Azure AD/Entra ID, ADFS, Okta, 사내 SAML IdP,
Active Directory 등)을 Ontul 에 연동하는 절차를 **관리 화면(Admin UI) 기준으로** 단계별로
설명합니다. 동작 원리와 속성 이름은 [Single Sign-On (OIDC/SAML/LDAP)](sso.md) 에 있습니다.

Ontul 을 플랫폼의 IAM 엔진(권한 검사 서버)으로 쓰는 구성에서도 절차는 같습니다. 이때 아래의
"관리 화면"은 **IAM 엔진(Ontul) 자신의 화면**입니다. 플랫폼 콘솔의 `Settings → IAM` 화면이 아닙니다.

---

## 1. 시작하기 전에 정할 것

### 1-1. 어떤 방식으로 연동할지

| 방식 | 사용자가 보는 것 | 언제 고르나 |
| --- | --- | --- |
| **OpenID Connect (OIDC)** | 로그인 화면의 `Sign in with OpenID Connect` 버튼 → IdP 로그인 → 돌아옴 | IdP 가 OIDC 를 지원하면 가장 간단합니다 (Keycloak, Entra ID, Okta 등) |
| **SAML 2.0** | 로그인 화면의 `Sign in with SAML` 버튼 → IdP 로그인 → 돌아옴 | 사내 표준이 SAML 이거나 IdP 가 SAML 만 제공할 때 (ADFS 등) |
| **LDAP / Active Directory** | 별도 버튼 없음. **기존 로그인 폼에 사내 계정·비밀번호를 그대로 입력** | 웹 SSO 가 없고 AD/LDAP 디렉터리만 있을 때 |

여러 방식을 동시에 켤 수 있습니다. **로컬 계정(`admin` 등)의 비밀번호 로그인은 SSO 를 켜도 계속
동작**하므로, 설정을 잘못해도 관리자가 잠기지 않습니다.

### 1-2. 브라우저가 접속하는 주소 (가장 중요)

OIDC 의 Redirect URI, SAML 의 Assertion Consumer URL 은 **사용자 브라우저가 실제로 여는 주소**여야
합니다. 마스터가 여러 대이고 앞에 프록시(nginx·로드밸런서)가 있으면 **프록시 주소**를 씁니다.
마스터 한 대의 주소를 넣으면 그 마스터가 멈췄을 때 SSO 가 통째로 실패합니다.

이 문서에서는 그 주소를 `https://iam.example.com` 으로 표기합니다. 실제 값으로 바꾸십시오.

| 용도 | 주소 |
| --- | --- |
| 관리 화면 | `https://iam.example.com/admin/` |
| OIDC Redirect URI | `https://iam.example.com/admin/auth/sso/oidc/callback` |
| SAML Assertion Consumer URL (ACS) | `https://iam.example.com/admin/auth/sso/saml/acs` |

로그인 흐름은 어느 마스터에서 시작하고 어느 마스터로 돌아와도 동작합니다(로그인 상태를 모든
마스터가 공유하는 키로 봉인해 전달합니다). 그래서 프록시에 sticky session 설정은 필요 없습니다.

### 1-3. 권한은 IdP 가 아니라 Ontul 의 그룹·정책이 정합니다

SSO 로 로그인한 사용자는 IdP 가 알려준 **그룹 이름**을 가지고 들어옵니다. 그 그룹 이름이
Ontul 의 그룹으로 매핑되고, **그 그룹에 붙은 정책**이 모든 권한을 결정합니다.

- SSO 사용자를 위해 **로컬 사용자 계정을 만들지 않습니다.** 사용자 목록에 나타나지도 않습니다.
- 대신 **Ontul 쪽에 그룹과 정책을 미리 만들어 두어야** 합니다(2장).
- 매핑되는 그룹이 하나도 없으면 로그인을 **거부**합니다(아무 권한 없는 상태로 들여보내지 않음).
- 같은 이름의 로컬 사용자가 있으면 **로컬 계정의 정책이 우선**합니다.

IdP 관리자에게 미리 받아야 할 것:

- 사용자 이름으로 쓸 값 (OIDC claim 이름 / SAML attribute 이름 / AD 의 로그인 속성)
- 그룹 정보를 담는 값 (OIDC `groups` claim / SAML `groups` attribute 등) — **IdP 에서 이 값을 토큰에
  넣도록 설정해야** 합니다
- Ontul 에 권한을 줄 IdP 그룹 이름 목록 (예: `DATA-ADMIN`, `DATA-ANALYST`)

---

## 2. Ontul 에 그룹과 정책 만들기

관리자(`AdministratorAccess` 정책을 가진 계정)로 관리 화면에 로그인합니다. 왼쪽 메뉴
**`GOVERNANCE` → `IAM`**.

예: IdP 그룹 `DATA-ADMIN` → Ontul 관리자, `DATA-ANALYST` → 조회 전용.

| 하는 일 | 클릭 순서 |
| --- | --- |
| 그룹 만들기 | `Groups` 탭 → **`New Group`** → 이름 입력 (예: `ontul-admins`, `ontul-readers`) |
| 정책 만들기 (필요 시) | `Policies` 탭 → **`New Policy`** → `Policy Identifier` · `Governance Template`(고르면 manifest 가 채워짐) · `IAM Manifest (JSON)` → **`Commit Security Policy`** |
| 그룹에 정책 붙이기 | `Groups` 탭 → **그룹 행을 눌러 펼침** → `Policy Compliance` 의 **`Attach Custom`** → 드롭다운에서 정책 → **`Attach Security Manifest`** |

- 그룹 행에는 버튼이 없습니다. **행을 눌러 펼쳐야** 정책 부착 버튼이 나옵니다.
- SSO 사용자가 관리 화면에서 설정을 바꿀 수 있으려면 그 그룹에 **`AdministratorAccess`** 가 붙어 있어야
  합니다. 없으면 화면은 읽기 전용으로 보입니다(변경 버튼이 숨겨지거나 비활성).
- 어떤 정책을 붙일지는 [IAM Policy Templates](iam-policy-templates.md) 를 참고하십시오.

> **그룹 이름을 IdP 와 똑같이 지을 수도 있습니다.** 그러면 4장의 그룹 매핑을 비워 둬도 됩니다
> (IdP 그룹 이름이 그대로 Ontul 그룹 이름으로 쓰임). 다만 그 경우 IdP 에 같은 이름의 그룹을 새로
> 만드는 것만으로 권한이 생기므로, 운영 환경에서는 매핑을 명시하는 쪽을 권장합니다.

---

## 3. SSO 화면 열기

왼쪽 메뉴 **`GOVERNANCE` → `Single Sign-On`**. 탭이 네 개 있습니다.

| 탭 | 내용 |
| --- | --- |
| `OpenID Connect` | OIDC 설정 |
| `SAML 2.0` | SAML 설정, 메타데이터 교환 |
| `LDAP / AD` | 디렉터리 설정 |
| `Identity mapping` | 그룹 매핑, 세션 시간 |

공통 동작:

- 입력을 바꾸면 아래 **`Save (n)`** 버튼에 바뀐 항목 수가 표시됩니다. 여러 탭을 오가며 고친 뒤 한 번에
  저장해도 됩니다.
- 저장하면 **모든 마스터에 즉시 적용**됩니다. 파일 수정이나 재시작이 필요 없습니다
  ("Saved. Every master picks this up from the cluster store — no restart.").
- 활성화된 방식의 탭 이름 옆에 **초록 점**이 표시됩니다.
- 비밀값(Client secret, Bind password)은 저장 후 다시 보여주지 않습니다. 라벨 옆에 `set` / `not set`
  으로 저장 여부만 표시되고, **비워 두고 저장하면 기존 값이 유지**됩니다.
- 칸을 비워 두면 기본값(또는 `ontul.properties` 에 적힌 값)이 쓰입니다.
- 관리자가 아닌 사용자에게는 모든 칸이 읽기 전용이고 `Read-only: changing SSO settings needs
  AdministratorAccess.` 가 표시됩니다.

---

## 4. 방식별 설정

### 4-A. OpenID Connect

**IdP 쪽 (IdP 관리자에게 요청)**

1. 클라이언트(애플리케이션) 등록 — 종류: confidential(웹 애플리케이션), 흐름: Authorization Code.
2. Redirect URI 에 `https://iam.example.com/admin/auth/sso/oidc/callback` 을 **정확히 같은 문자열로**
   등록 (끝의 `/` 유무까지 일치해야 함).
3. ID 토큰에 그룹 claim 이 들어가도록 설정 (Keycloak 은 `Group Membership` mapper, Entra ID 는
   `groupMembershipClaims` 등).
4. 받을 것: **Issuer URL**, **Client ID**, **Client secret**.

**Ontul 쪽: `OpenID Connect` 탭**

| 화면 항목 | 입력 값 | 비고 |
| --- | --- | --- |
| `Enable OpenID Connect` | 켬 | |
| `Issuer URL` | 예: `https://sso.example.com/realms/company` | 이 주소의 discovery 문서(`/.well-known/openid-configuration`)에서 나머지 엔드포인트와 서명 키를 자동으로 읽습니다 |
| `Client ID` | IdP 에서 받은 값 | |
| `Client secret` | IdP 에서 받은 값 | 저장 후 `set` 으로 표시 |
| `Redirect URI` | `https://iam.example.com/admin/auth/sso/oidc/callback` | IdP 에 등록한 값과 정확히 일치 |
| `Scopes` | 비워 둠 (기본 `openid profile email`) | **`groups` 를 넣지 마십시오.** 표준 scope 가 아니어서 정의하지 않은 IdP 는 요청 전체를 `invalid_scope` 로 거부합니다. IdP 문서에 있는 경우만 추가 |
| `Username claim` | 기본 `preferred_username` | 사번 등 다른 claim 을 쓰면 그 이름. 값이 없으면 `sub` 를 씁니다 |
| `Groups claim` | 기본 `groups` | IdP 가 그룹을 넣는 claim 이름 |
| `Audience` | 비워 둠 | 비우면 Client ID. 다른 애플리케이션용으로 발급된 토큰은 거부됩니다 |

**`Save`** 를 누른 뒤 **`Test the connection`**. 테스트는 **저장된 값**을 검사하므로 저장 전에 누르면
"save your changes first" 가 나옵니다.

성공 예: `Discovery succeeded, 2 signing key(s) published. Authorization endpoint: https://…/auth`

### 4-B. SAML 2.0

SAML 연동은 칸을 채우는 작업이 아니라 **메타데이터 문서를 서로 교환**하는 작업입니다.

**`SAML 2.0` 탭에서 순서대로**

1. `Enable SAML 2.0` 켬.
2. 아래 칸 중 SP 쪽 두 개를 먼저 입력하고 **`Save`**:
    - `SP entity ID` — 예: `ontul` (IdP 에 등록되는 이 서비스의 식별자. 비우면 `ontul`)
    - `Assertion consumer URL` — `https://iam.example.com/admin/auth/sso/saml/acs`
3. (IdP 가 요청 서명 또는 assertion 암호화를 요구하는 경우만) **`Generate an SP keypair`** 를 누르고,
   `Sign authentication requests` 를 켜고 **`Save`**.
4. **`This cluster's SP metadata`** 를 눌러 `ontul-sp-metadata.xml` 을 내려받아 **IdP 관리자에게 전달**합니다.
   entity ID, ACS 주소, (3을 했다면) 인증서가 들어 있습니다.
5. IdP 관리자에게 받을 것: **IdP 메타데이터 XML**. 또한 IdP 에서 assertion 에 사용자 이름 attribute 와
   그룹 attribute 를 넣도록 설정해 달라고 요청합니다.
6. 받은 IdP 메타데이터 XML 전체를 `Paste your identity provider's SAML metadata XML here` 칸에 붙여 넣고
   **`Read the provider's metadata`**. `IdP entity ID`, `IdP sign-on URL`, `IdP signing certificate`
   세 칸이 자동으로 채워지고 **바로 저장됩니다**(수기 입력할 필요 없음).
7. 나머지 항목 확인 후 **`Save`**:

| 화면 항목 | 입력 값 | 비고 |
| --- | --- | --- |
| `NameID format` | 비워 둠 | 비우면 IdP 가 설정된 형식으로 발급. 지정하면 그 형식을 발급하지 않는 IdP 가 요청을 거부합니다 |
| `Username attribute` | 기본 `uid` | assertion 에서 사용자 이름을 읽을 attribute. 없으면 NameID 를 씁니다 |
| `Groups attribute` | 기본 `groups` | IdP 가 그룹을 넣는 attribute 이름 (ADFS 는 보통 `http://schemas.microsoft.com/ws/2008/06/identity/claims/role` 같은 URI 형식) |

8. **`Check the configuration`**. 성공 예: `Configuration is usable. AuthnRequest built and signed (… bytes),
   identity-provider certificate parsed.`

> SP keypair 를 **나중에 다시 생성하면** 인증서가 바뀝니다. 4번(SP 메타데이터 전달)을 다시 해서
> IdP 가 새 인증서를 받게 하십시오.

모든 assertion 은 서명·audience·유효 기간·issuer 를 검사하고, 이미 쓰인 assertion 은 재사용을 거부합니다.

### 4-C. LDAP / Active Directory

**IdP 쪽에서 받을 것**: 디렉터리 주소, 사용자 검색용 서비스 계정(DN 과 비밀번호), 사용자·그룹이
있는 OU(base DN).

**`LDAP / AD` 탭**

| 화면 항목 | 입력 값 (Active Directory 예) | 비고 |
| --- | --- | --- |
| `Enable LDAP / Active Directory` | 켬 | |
| `Directory URL` | `ldaps://ad.example.com:636` | `ldaps://` 를 쓰거나 아래 `StartTLS` 를 켜십시오. 아니면 비밀번호가 평문으로 전송됩니다 |
| `Bind DN` | `cn=svc-ontul,ou=service,dc=example,dc=com` | 사용자 항목을 찾는 서비스 계정 |
| `Bind password` | 서비스 계정 비밀번호 | |
| `User base DN` | `ou=people,dc=example,dc=com` | |
| `User filter` | `(sAMAccountName={0})` | `{0}` 은 로그인 이름. OpenLDAP 은 기본 `(uid={0})` |
| `Group base DN` | `ou=groups,dc=example,dc=com` | |
| `Group filter` | 기본 `(member={0})` | `{0}` 은 사용자 DN. 사용자의 `memberOf` 도 함께 읽습니다 |
| `Group name attribute` | 기본 `cn` | |
| `StartTLS` | `ldap://` 를 쓸 때 켬 | |

**`Save`** → **`Test the bind`**. 서비스 계정으로 디렉터리 접속·바인드가 되는지 확인합니다.

로그인은 별도 버튼 없이 **기존 로그인 폼**에 사내 계정과 사내 비밀번호를 입력합니다. 로컬 계정
비밀번호를 먼저 확인하고, 맞지 않으면 디렉터리로 확인합니다.

---

## 5. 그룹 매핑: `Identity mapping` 탭

| 화면 항목 | 입력 값 | 비고 |
| --- | --- | --- |
| `Group mappings` | `DATA-ADMIN:ontul-admins,DATA-ANALYST:ontul-readers` | `IdP그룹:Ontul그룹` 을 쉼표로 구분. **대소문자까지 정확히** 일치해야 합니다 |
| `Allow identities whose groups map to nothing` | 끔 (권장) | 켜면 매핑되는 그룹이 없는 사용자도 로그인되지만 아무것도 할 수 없습니다 |
| `Federated session seconds` | 기본 `3600` | SSO 세션 유효 시간(초). 짧을수록 IdP 에서 사용자를 막았을 때 빨리 반영됩니다 |

- 비워 두면 IdP 그룹 이름을 그대로 Ontul 그룹 이름으로 씁니다.
- **한 번 설정하면 목록에 없는 그룹은 모두 버려집니다.** IdP 에 그룹을 새로 만드는 것만으로 권한이
  생기지 않습니다.
- Keycloak 처럼 그룹 이름 앞에 `/` 가 붙어 오면(`/DATA-ADMIN`) 자동으로 떼고 비교합니다.
- 매핑 대상 Ontul 그룹(오른쪽)은 2장에서 **실제로 만들어 둔 그룹**이어야 합니다. 없는 그룹으로 매핑하면
  로그인은 되지만 정책이 없어 모든 작업이 거부됩니다.

**`Save`** 를 누릅니다.

---

## 6. 로그인 확인

1. 다른 브라우저(또는 시크릿 창)에서 `https://iam.example.com/admin/` 을 엽니다.
2. 로그인 화면에 `or` 아래 **`Sign in with OpenID Connect`** / **`Sign in with SAML`** 버튼이 보이는지
   확인합니다. LDAP 만 켰다면 폼 아래에 "Directory accounts sign in with the form above" 안내가 보입니다.
3. 버튼을 눌러 IdP 에서 로그인 → 관리 화면으로 돌아오면 성공입니다.
4. 관리자 그룹으로 매핑된 사용자는 `Single Sign-On` 화면의 `Save` 가 활성이고, 조회 전용 그룹 사용자는
   `Read-only` 안내가 보이는지 확인합니다.

**관리 화면 첫 연동 때는 관리자 창(로컬 `admin` 로그인)을 닫지 말고** 다른 창에서 시험하십시오.
로컬 로그인은 SSO 와 상관없이 계속 됩니다.

### API 로 확인 (선택)

```bash
# 로그인 화면이 보여줄 방식 (인증 불필요)
curl -s https://iam.example.com/admin/auth/sso/providers
# {"providers":["oidc","saml"],"localLoginEnabled":true}

# LDAP 계정으로 토큰 받기 (스크립트·SDK 용)
curl -s -X POST https://iam.example.com/v1/api/auth/sso \
  -H 'Content-Type: application/json' \
  -d '{"username":"hong.gildong","password":"<사내 비밀번호>"}'
# {"accessToken":"...","username":"hong.gildong","federated":true,"idp":"LDAP","groups":["ontul-readers"]}
```

---

## 7. 문제 해결

로그인이 실패하면 로그인 화면 위에 이유가 표시됩니다. 마스터 로그(`Console SSO refused`,
`Federated login refused`, `Directory login refused`)에 같은 내용이 남습니다.

| 증상 | 원인과 조치 |
| --- | --- |
| IdP 화면에서 `redirect_uri` 오류 | `Redirect URI` 와 IdP 에 등록한 값이 다릅니다. 프로토콜(`http`/`https`), 포트, 끝 `/` 까지 맞추십시오 |
| `invalid_scope` | `Scopes` 에 `groups` 등 IdP 가 모르는 scope 가 있습니다. 비우십시오 |
| OIDC/SAML 로그인 후 "Signed in as …, but no group from the identity provider maps to a group on this cluster" | IdP 로그인은 성공했고 **그룹 매핑이 비었습니다.** ① IdP 가 그룹 claim/attribute 를 넣는지, ② `Groups claim`/`Groups attribute` 이름이 맞는지, ③ `Group mappings` 의 왼쪽 이름이 IdP 그룹 이름과 대소문자까지 같은지 확인하십시오 |
| LDAP: 로그인 폼에서 "Invalid credentials" | 로그인 폼은 비밀번호 오류와 그룹 매핑 없음을 **구분하지 않습니다.** 아래 명령으로 구분하십시오: 401 이면 계정·비밀번호·검색 설정 문제, **403** 이면 비밀번호는 맞았고 그룹 매핑이 빈 것입니다(비밀번호를 초기화할 일이 아님) |
| 로그인은 되는데 모든 화면이 권한 오류 | 매핑된 Ontul 그룹이 없거나 그 그룹에 정책이 없습니다(2장) |
| 로그인은 되는데 설정 화면이 읽기 전용 | 그 사용자의 그룹에 `AdministratorAccess` 가 없습니다 |
| SAML: "The assertion consumer received no SAMLResponse" | IdP 가 ACS 로 POST 하지 않았습니다. IdP 의 ACS binding 이 HTTP-POST 인지, ACS 주소가 맞는지 확인하십시오 |
| SAML: 서명 검증 실패 | IdP 서명 인증서가 바뀌었습니다. IdP 메타데이터를 다시 받아 `Read the provider's metadata` |
| SAML: IdP 가 요청을 거부 | `NameID format` 을 비우십시오. IdP 가 서명된 요청을 요구하면 4-B 의 3번 |
| LDAP: `Test the bind` 실패 | 주소·포트·인증서(ldaps 의 경우 마스터 JVM 이 디렉터리 인증서를 신뢰해야 함)·Bind DN/비밀번호 확인 |
| LDAP: 아래 명령이 401 | 사용자 검색 실패(`User base DN`/`User filter`) 또는 비밀번호 오류. AD 는 `(sAMAccountName={0})` 인지 확인 |
| 일부 시도만 실패 | `Redirect URI`/ACS 가 마스터 한 대 주소입니다. 프록시 주소로 바꾸십시오(1-2) |

LDAP 로그인 실패 원인 구분:

```bash
curl -s -w '\n%{http_code}\n' -X POST https://iam.example.com/admin/auth/sso/ldap/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"hong.gildong","password":"<사내 비밀번호>"}'
# 200 → 정상, 401 → 계정/비밀번호/검색 설정, 403 → 그룹 매핑 없음
```

---

## 8. 운영 시 알아둘 점

- **퇴사·권한 회수**: IdP 에서 사용자를 막으면 새 로그인은 즉시 막힙니다. 이미 발급된 세션은
  `Federated session seconds` 가 지날 때까지 유지됩니다(Ontul 은 IdP 의 변경을 통보받지 않음).
  SSO 세션에는 refresh token 이 없어 연장되지 않습니다.
- **엔진의 권한 검사와 SSO**: 쿼리 엔진·분산 처리·스트림 처리의 권한 플러그인은 사용자를 **이름으로**
  조회합니다. SSO 사용자의 그룹은 그 사용자가 SSO 로 로그인한 뒤 세션 시간 동안만 유효합니다.
  세션이 없는 SSO 사용자의 이름은 그룹이 없는 것으로 판정되어 거부됩니다. 엔진을 계속 쓰는 서비스
  계정이나 배치 사용자는 **로컬 사용자**로 만들고 그룹에 넣어 두십시오.
- **같은 이름의 로컬 사용자**가 있으면 SSO 로 로그인해도 로컬 계정의 그룹·정책이 적용됩니다.
- 설정은 클러스터 저장소에 복제되므로 마스터를 추가·재시작해도 다시 입력할 필요가 없습니다.
- SSO 설정 변경은 관리자만 할 수 있습니다. 일반 사용자도 이 화면을 볼 수 있고 SP 메타데이터를 내려받을
  수 있지만, 저장·테스트·키 생성·메타데이터 가져오기는 할 수 없습니다.

## 함께 보기

- [Single Sign-On (OIDC/SAML/LDAP)](sso.md) — 동작 원리와 `ontul.properties` 속성
- [IAM](iam.md) — 그룹·정책
- [IAM Policy Templates](iam-policy-templates.md) — 그룹에 붙일 정책
