#### Original: [What Are Refresh Tokens and How to Use Them Securely](https://auth0.com/blog/refresh-tokens-what-are-they-and-when-to-use-them/)

이 포스트는 [OAuth2.0](https://auth0.com/docs/authenticate/protocols/oauth) 으로 정의되는 refresh token의 개념에 대해서 설명합니다. 우리는 다른 token들과 비교해볼 것이며, 보안, 사용성, 개인정보보호간의 균형을 어떻게 맞추는지 알아 볼 것입니다.

## Token은 무엇일까요?
Token은 사용자의 식별자를 확인하하거나 작업을 수행하도록 권한을 부여하는 프로세스를 용이하게 하기에 충분한 정보를 포함하고 있는 data 덩어리입니다. 전반적으로, token은 application이 인증과 인가 프로세스를 수행할 수 있게 하는 아티팩트입니다.

indentity framework와 protocol은 application과 resource에 안전하게 접근하기 위하여 일반적으로 token-based 전략을 사용합니다. 예를 들어 우리는 OAuth 2.0 인가와 OIDC 인증을 사용할 수 있습니다.

OAuth2.0은 수많은 인가 프레임워크들 중에서 가장 대중적입니다. 이것은 어플리케이션이 사용자를 대신하여 다른 서버에서 호스팅하는 리소스에 액세스 할 수 있도록 설계되어있습니다. OAuth 2.0는 **Access Token**과 **Refresh Token**을 사용합니다.

OpenID Connect (OIDC)는 유저 인증, 유저 동의, 토큰 발행을 하는 검증(identity) 프로토콜입니다. OIDC는 **ID Token**을 사용합니다.

우리는 어플리케이션 시스템의 인증과 인가 프로세스를 처리해주는 OAuth 2.0과 OpenID Connect를 활용한 3가지의 token type을 알아볼 것입니다. 이 과정에서, 우리는 개발자가 보안수준에 타협하지 않으면서 편의성을 제공하는 어플리케이션을 개발하는데 도움이 되는 refresh token의 중요한 역할을 확인해보겠습니다.

## Token Types
### ID token이 무엇인가요?
이름에서 제안하는 바처럼, [ID token](https://auth0.com/docs/tokens/id-tokens) 은 클라이언트 어플리케이션이 사용자의 식별자를 얻기를 원할때(소비하기위해) 사용할 수 있는 아티팩트입니다. 예를 들어, ID token은 이름, 이메일, 유저의 프로필 사진을 포함할 수 있습니다. 즉, client application은 ID token을 유저경험 개인화를 위한 사용자 프로필 구성을 위해 활용할 수 있습니다.

인증 프로세스를 구현한 OIDC 프로토콜을 따르는 인증서버는 클라이언트에게 유저가 로그인을 하는 시점에 ID token을 발행합니다. ID token의 소비자는 주로 SPA(Single Page Application)나 모바일 어플리케이션입니다. 그들은 의도된 사용자들입니다.(They are the intended audience)

### Access token이 무엇인가요?
사용자가 로그인을 할 때, 인가 서버는 access token을 발행합니다. 이것은 클라이언트 어플리케이션이 API server를 안전한 방법으로 호출하기 위해 사용합니다. 클라이언트 어플리케이션은 사용자를 대신하는 서버상의 보호받는 리소스에 접근이 필요할 때, access token은 서버에게 클라이언트가 유저가 특정 작업을 수행하거나 리소스에 접근하기 위한 인가를 받았다는 신호가 됩니다.

OAuth 2.0은 access token의 형식을 정의하지 않습니다. 예를 들어, Auth0에서 access token은 Management API를 위해서 발행되며, access token은 JSON Web Token 표준을 따르는 Auth0로 등록된 다양한 API를 위해서 발행됩니다. 이 기본 구조는 특정 JWT 구조를 따르며, 여기에는 토큰 자체에 assertion된 표준 JWT 클레임이 포함됩니다.

아래의 json body는 JWT 포맷을 따르는 디코딩된 access token입니다.
```json
{
  "iss": "issuer",
  "sub": "subject",
  "aud": [
    "audience",
    "api-indentifier"
  ],
  "exp": 1489179954,
  "iat": 1489143954,
  "scope": "openid profile read:account birthday"
} 
```

여기서 중요한 점은 access token이 Bearer token이라는 점입니다. 이 token을 가지고 있다면, 누구든지 사용할 수 있습니다. access token은 식별 인자로 활용하기 보다는 보호받는 리소스에 접근하기 위한 credential artifact로서 기능합니다. 어뷰저는 이론적으로 access token을 탈취하여 리소스에 접근 가능합니다.

따라서, 어뷰저가 사용할 수 있게 된 access token의 위험을 최소화 하기 위한 보안 전략들이 중요해집니다. 한 가지 이 문제를 완화시켜줄 수 있는 방법은 짧은 수명을 갖는 access token을 발행하는 것입니다: 문자그대로 수시간내에서 수일내에 만료되는 토큰입니다.

Client application이 새로운 access token을 받아갈 수 있는 다양한 방법들이 존재합니다. 예를 들어, access token이 만료된 이후에, client application은 유저에게 죽시 로그인을 요구하고 access token을 발행받을 수 있습니다. 혹은, 인가 서버는 만료된 access token을 대신할 수 있는 token을 받아가도록 refresh token을 client application에 발행할 수 있습니다.

## Refresh Token은 무엇인가요?
앞서 언급한대로, 보안을 목적으로 하기위해, access token은 짧은 시간내에 만료되어야 할 때가 있습니다. 일단 만료가 디면, client application은 access token을 "refresh"하기 위해 refresh token을 사용할 수 있습니다. 즉, [refresh token](https://auth0.com/docs/secure/tokens/refresh-tokens) 은 사용자에게 로그인을 요청하지 않고 새로운 access token을 받아갈 수 있는 credential artifact입니다.
SPA -> Refresh Token -> Authorization Server -> Access Token -> SPA
    \-> Access Token -> Resource Server -> Protected Resource -> SPA

client application은 refresh token이 만료되기 전까지는 계속해서 access token을 발행받을 수 있습니다. 결론적으로, refresh token은 상당히 긴 수명을 갖고 있으며 이론적으로 보호받는 리소스에 접근할 수 있는 access token을 token bearer에게 무한정으로 제공할 수 있는 수단이 됩니다. Refresh token의 전달자는 올바른 유저가 될수도 어뷰저가 될수도 있습니다. 즉, Auth0와 같은 보안담당자는 이런 강력한 토큰이 의도된 사용자에 의해서만 사용되고 보관할 수 있도록 신뢰성있는 메커니즘을 마련해야합니다.

## 언제 Refresh Token을 사용할까요
OAuth 2.0 스펙은 access token와 refresh token을 정의하고 있음을 명심해야합니다. 그래서, 만약 우리가 SAML과 같은 검증 프로토콜이나 프레임워크의 인가 전략에 대해서 논의한다면, 우리는 access token / refresh token의 개념을 가지고 있지 않습니다.

웹 개발을 하고 있는 분들에게 access token과 refresh token은 흔한 상식입니다. 웹에서는 OAuth 2.0 프레임워크와 OIDC 프로토콜을 사용해서 token-based 인증과 인가를 광범위하게 사용하고 있습니다.

OAuth 2.0와 OIDC가 결합되면, 인가와 인증 플로우가 활성화 됩니다. 각 플로우는 이점과 절차를 가지며, 우리가 access token과 refresh token을 사용해야하는 환경에서 최적의 시나리오와 아키텍쳐를 정의합니다.
- 클라이언트가 서버를 호출하는 전통적인 web application이라면 [Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow#authorization-code-flow) 를 활용합니다.
- 클라이언트가 SPA라면 [PKCE를 사용하는 Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow#authorization-code-flow-with-proof-key-for-code-exchange-pkce-) 를 활용합니다.
- Access token이 필요하지 않은 SPA라면 [From Post를 활용하는 묵시적 플로우](https://auth0.com/docs/get-started/authentication-and-authorization-flow#implicit-flow-with-form-post) 를 사용합니다
- 클라이언트가 리소스 오너라면 [Client credential flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow#client-credentials-flow) 를 사용합니다.
- 클라이언트가 절대적으로 신뢰가능한 user credential 이라면 [Resource Owner Password Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow#resource-owner-password-flow) 를 사용합니다.

이를 위한 앱이 있다면, 이를 위한 플로우도 있기 마련입니다.

Implicit Flow를 사용할 때 명심해야할 점은 인가 서버는 refresh token을 발행하지 않아야 한다는 점입니다. Implicit flow는 시스템 아키텍쳐의 프론트엔뜨 에서 보통 실행되는 SPA에서 종종 구현됩니다. 프론트엔드에서 refresh token을 안전하게 관리하는 것은 쉬운일이 아닙니다.

PKCE와 Authorization code flow의 사용은 Implicit Flow에 내재된 많은 위험이 완화됩니다. 예를 들어, implicit grant type을 사용할 때, [access token은 URI fragment로 전달되며](https://tools.ietf.org/html/rfc6749#section-10.3) 이는 비인가 그룹에게 노출될 수 있습니다. 당신은 이 취약점에 대해서 [묵시적 흐름에서 리소스 소유자를 가장하기 위한 액세스 토큰의 오용](https://datatracker.ietf.org/doc/html/rfc6749#section-10.16) 에서 더 알아볼 수 있습니다.

하지만, 당신의 application에 PKCE를 구현하는 것은 refresh token이 얼마나 안전한지와는 무관합니다.

#### 하지만, 당신은 refresh token을 필요로 하지 않을 수도 있습니다.
사용자의 인터럽트와 refresh token의 강력한 기능에 기대지 않은 채로 access token을 받아갈 수 있는 시나리오를 가정합시다. 혹은 다른 예로 쿠키나 [silent authentication](https://auth0.com/docs/authenticate/login/configure-silent-authentication) 와 같은 session base 인증을 예로 들어보겠습니다.

하지면 매일 수십억명의 사람들이 SPA를 사용합니다. 사용자에게 [보안과 편의성 사이에서 균형감있는 경험을 선사하는 것](https://auth0.com/blog/balance-user-experience-and-security-to-retain-customers/) 은 중요합니다. SPA가 덜 위험하고 더 안전한 방식으로 refresh token을 제공하기 위해서 우리가 할 수 있는 일은 있을까요?

[인증 플랫폼이 refresh token rotation](https://auth0.com/blog/securing-single-page-applications-with-refresh-token-rotation/) 의 제공은 SPA가 refresh token을 사용할 수 있습니다. [스펙](https://datatracker.ietf.org/doc/html/rfc6749#section-10.4) 은 SPA와 같이 client가 refresh token을 검증할 수 없을때, refresh token rotation 없이는 사용해서는 안된다고 강조하고 있습니다.

## Refresh Token의 보안 유지
짧은 생명 주기를 갖는 access token은 application의 보안수준을 높여주지만, 만료되었을 때 로그인이 필요하다는 다른 비용을 요구합니다. 빈번한 재인증은 application의 사용자 경험을 경감시킵니다. 당신이 사용자의 데이터를 보호하는 동안, 사용자는 당신의 application을 사용을 어려워 하고 좌절감을 맛볼 것입니다.

Refresh token은 사용성과 보안의 균형감에 도움을 줍니다. refresh token은 긴 생명주기를 갖기 때문에, 당신은 짧은 주기의 access token을 요청할 때 사용할 수 있습니다.

하지만, refresh token 또한 bearer token이기 때문에, 우리는 refresh token이 유출되거나 의도되지 않은 상황이 발생했을 때 사용을 제한하거나 축소시킬 수 있는 적절한 전략을 세워야 합니다. Refresh token을 가지고 있다면 누구든지 원하는 시점에 access token을 가져갈 수 있기 때문입니다. "그들"은 올바른 사용자 일수도, 어뷰저 일수도 있습니다.

Auth0에서는 safeguard를 도입하고 refresh token의 라이프사이클을 제어함으로써 refresh token의 사용과 관련된 문제점을 완화시켰습니다. Auth0는 refresh token rotation을 지원하며 이는 자동 재사용 감지 역시 제공합니다.

## Refresh Token Rotation
최근 까지, SPA가 사용자의 세션을 유지하는데 도움을 주는 강건한 전략은 silent authentication과 같이 활용하는 [PKCE를 사용하는 Authorization Code Flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow#authorization-code-flow-with-proof-key-for-code-exchange-pkce-) 였습니다. Refresh token ratation은 [silent authentication](https://auth0.com/docs/authenticate/login/configure-silent-authentication) 보다 뛰어난 방법이며, refresh token을 사용해서 access token을 발행받아 갈 수 있는 기술입니다.

Refresh token rotation은 매번 application이 refresh token으로 accesst token을 발행받아 갈 때, 새 refresh token도 발행할 것을 보장합니다. 그러므로, 당신은 더 이상 생명주기가 긴 refresh token을 유지않아도 되며, 이는 위태로운 상황에 빠졌을 때에도, resource로의 부적절한 접근을 제한을 제공할 수 있습니다. 부적잘한 접근의 위협은 refresh token을 교환하고 무효화함으로써 경감시킵니다.

## Refresh Token Automatic Reuse Detection
Refresh token은 bearer token입니다. 따라서, 인가 서버는 누가 적합한 유저인지 누가 부적합한 유저인지 access token의 생성 요청을 받았을 때 알 수 없습니다. 우리는 모든 유저가 어뷰저 일 수 있다고 생각하고 처리해야합니다.

아래와 같이 올바른 유저와 어뷰저가 race condition 상황에 놓였을 때 어떻게 처리할 수 있을까요?
- 올바른 유저가 1개의 refresh token과 1개의 access token을 가지고 있습니다.
- 어뷰저가 올바른 유저로 부터 1개의 Refresh token을 탈취했습니다.
- 올바른 유저가 refresh token으로 새 access token과 refresh token을 발행받아갔습니다.
- 인가 서버는 올바른 유저에게 2번째 refresh / access token을 발행합니다.
- 어뷰저가 이전 refresh token으로 access token의 습득을 시도합니다.

(Auth0)와 같이 Automatic reuse detection을 가지고 있는 인가 서버는 이 상황에서 아래와 같이 동작합니다.
- 인가 서버는 모든 refresh token을 내림차순으로 트래킹하고 있습니다. 이를 "token family"라고 합니다.
- 인가 서버는 누군가가 첫번째 refresh token을 재사용하려는 것을 인지 하고, token family를 모두 무효화합니다. (이는 새로 발급된 refresh token도 포함합니다)
- 인가 서버는 어뷰저로부터의 접근을 제한합니다
- 새로 발급된 두번째 Access token이 만료되면, 올바른 사용자는 refresh token으로 새로운 refresh/access token을 요청합니다
- 인가 서버는 올바른 유저로 부터의 접근을 제한합니다
- 그리고 인가 서버는 refresh/access token을 제공하기 위해서 재인증을 요구합니다.

이전의 refresh token으로 token refresh를 요구할 때, 모든 refresh token이 무효화 되는것은 상당히 중요합니다. 이것은 token family의 모든 token들로 access token을 받아가는 것을 제한합니다.

이는 어뷰저이거나 적법한 사용자이거나 누구든지 상관없이 동작합니다. Sender-constraint를 적용하지 않는다면 인가 서버는 접근하는 사용자가 적법한지 어뷰저인지 알 수 없습니다.

## Refresh Tokens Help Us Embrace Privacy Tools
개인 정봅 보호는 현 IT 세계에서 가장 핫한 주제입니다. 우리는 사용자의 경험과 보안의 균형뿐만 아니라, 개인정보보호와도 균형을 맞추어야 하빈다.

최근 Intelligent Tracking Prevention과 같은 브라우저 개인정보보호 기술의 개발은 session cookie로의 접근을 제한하고, 사용자에게 재인증을 요구합니다.

브라우저에는 특정 application만 접근이 허용된 persistent storage를 유지 할 수 없습니다. 보통, 긴 생명 주기를 갖는 refresh token으로 보호되는 리소스에 접근 할 수 있게 한다는 점에서 어뷰저가 이용할 수 있다는 취약성으로 SPA에는 적합하지 않다고 합니다.

왜냐하면 refresh token rotation은 session cookie의 접근과 관여되어있지 않고, ITP나 그와 유사한 메커니즘에 의해 영향받지 않습니다.

하지만, refresh token은 access token의 수명에 의해 제한됩니다. 이는 우리가 refresh token을 브라우저 개인정보보호 툴과 함께 사용할 수 있음을 의미하고 유저 경험을 경감하지 않은 채로 지속적은 접근 방법을 제공할 수 있습니다.

## You can store Refresh Token in local storage
Refresh token을 page refresh가 tab에 상관이 없는 persistent storage에 저장할 수 있습니다. 하지만 이는 XSS 공격이 포함된 JPA의 JS코드에 의해 어뷰저가 탈취하고 사용할 수 있습니다. XSS 공격은 SPA내의 코드 혹은 Bootstrap이나 Google Analytics 등 서드파티의 코드에 포함되어있을 수 있습니다.

하지만, refresh token rotation은 refresh token이 긴 수명을 갖고 있다고 하더라도, access token과 동일한 수명을 갖게 하기 때문에, 이 공격으로부터 발생하는 문제점을 줄일 수 있으며, 이는 refresh token을 local storage에 저장할 수 있게 합니다.
