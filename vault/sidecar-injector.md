#### Original: https://www.vaultproject.io/docs/platform/k8s/injector

# Agent Sidecar Injector
Vault Agent Injector는 [Vault Agent Templates](https://www.vaultproject.io/docs/agent/template) 을 사용하는 공유 메모리 볼륨에 Vault secret를 렌더링하는 Vault Agent 컨테이너를 포함하도록 pod 사양을 변경합니다. 공유 볼륨에 secret을 렌더링 함으로써, 팟내의 컨테이너는 Vault가 하는 행위를 이해하지 않고도 Vault secrets를 가져갈 수 있습니다.

Injector는 [Kubernetes Mutation Webhook Controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) 입니다. 컨트롤러는 팟 이벤트를 인터셉트하고 annotation이 요청에 존대한다면, mutation를 pod에 적용합니다. 이 기능은 [vault-k8s](https://github.com/hashicorp/vault-k8s) 가 제공하며, [Vault Helm](https://github.com/hashicorp/vault-helm) 를 사용해서 설치하고 설정할 수 있습니다.

## Overview
Vault Agent Injectorsms pod의 `CREATE`와 `UPDATE` 이벤트를 인터셉트함으로써 동작합니다. 컨트롤러는 이 이벤트들을 파싱하고, `vault.hashicorp.com/agend-inject: true` metadata annotaion을 찾습니다. 만약 발견한다면, 컨트롤러는 다른 annotation present를 기반으로 pod spec을 변경합니다.

### Mutations
최소한, 팟 내의 모든 컨테이너는 공유 메모리 볼륨을 탑재하도록 구성됩니다. 이 볼륨은 `vault/secrets`에 탑재되며, 팟 내의 다른 컨테이너들과 secret을 공유하기위한 Vault Agent에 의해서 사용됩니다.
다음으로, Vault Agent 컨테이너의 두가지 타입들이 주입될 수 있습니다: 이는 init과 sidecar입니다. init 컨테이너는 다른 컨테이너들이 기동되기 이전에 요청받은 secret을 공유 메모리에 채워둡니다. sidecar는 팟이 실행되는 위치에서 인증과 secret 렌더링을 지속합니다. annotation을 사용해서 초기와화 sidecar 컨테이너는 비활성화 될 수 잇습니다.
마지막으로, 볼륨의 두 가지 추가 속성은 Vault Agent 컨테이너에 선택적으로 마운트될 수 있습니다. 첫번째는 client와 CA와 같은 TLS를 요구를 포함하는 secret 볼륨입니다. 이 볼륨은 TLS를 활용해 Vault 서버의 인증을 검증하고 통신할 때 유용합니다. 두번째는 Vault Agent 설정 파일을 포함하는 설정 map입니다. 이 볼륨은 어떤 annotation이 제공하는 것 이상으로 Vault Agent를 조작하는데 유용합니다.

### Authenticating with Vault
Vault Agent Inject를 사용할 때 Vault의 주된 인증 방법은 팟에 첨부된 service account입니다. 다른 인증 방식은 사용하는 annotation으로 설정할 수 있습니다.
Kubernetes 인증을 위해서, service account는 Vault role과 원하는 secret 접근 권한을 부여하는 정책에 바운드 되어야합니다.
Service account는 Kubernetes 인증 수단을 함께 사용하는 Vault Agent Injector여야 합니다. Vault role을 service account가 정의되지 않은 pod에 default service account에 바인딩 하는것은 추천하지 않습니다.

### Requesting Secrets
Secret을 렌더링하는 Vault agent 컨테이너의 설정에는 2가지 방법이 있습니다.
- `vault.hashicorp.com/agent-inject-secret` annotation이나
- Vault Agent configuration file을 포함한 설정 map 입니다

#### Secrets via Annotations
Annotation을 사용하는 secret injection을 설정하기위해, 사용자는 아래의 정보들을 제공해야합니다:
- 하나 혹은 그 이상의 annotation 이나
- secret에 접근할 수 있는 Vault role

Annotation은 아래의 format을 가져야 합니다:
```yaml
vault.hashicorp.com/agent-inject-secret-<unique-name>: /path/to/secret
```

식별 가능한 이름은 렌더링된 secret의 파일명이 될 수 있고, 다수의 secret이 사용자에 의해 정의될 경우 unique해야합니다. 예를 들어, secret annotation은 아래와 같아야 합니다:
```yaml
vault.hashicorp.com/agent-inject-secret-foo: database/roles/app
vault.hashicorp.com/agent-inject-secret-bar: consul/creds/app
vault.hashicorp.com/role: 'app'
```
첫번째 annotation은 `vault/secrets/foo`로 렌더링 될 것이며, 두번째 annotaion은 `vault/secrets/bar`로 렌더링 될 것 입니다.

annotaion을 사용하는 rendered secret의 파일 포맷을 설정하는 것도 가능합니다. 예를 들어 아래의 secret은 `/vault/secrets/foo.txt`로 렌더링 됩니다.
```yaml
vault.hashicorp.com/agent-inject-secret-foo.txt: database/roles/app
vault.hashicorp.com/role: 'app'

```

secret unique name은 `.`, `_`, `-`, 알파벳으로 구성해야만 합니다.

##### Secret Templates
```text
Vault Agent는 secrets을 렌더링 하기 위해 Consul template을 사용합니다. template 작성과 관련된 다양한 정보는 Consul template documentation에서 확인할 수 있습니다.
```

Secret이 파일에 렌더링 되는 것도 설정가능합니다. 사용하는 template을 설정하기위해, 사용자는 `template` annotation과 secret과 동일한 unique name을 제공해야합니다. 이 annotation은 아래의 형식을 갖습니다.
```yaml
vault.hashicorp.com/agent-inject-template-<unique-name>: |
  <
    TEMPLATE
    HERE
  >

```

예를 들어, 아래와 같이 설정 가능합니다
```yaml
vault.hashicorp.com/agent-inject-secret-foo: 'database/roles/app'
vault.hashicorp.com/agent-inject-template-foo: |
  {{- with secret "database/creds/db-app" -}}
  postgres://{{ .Data.username }}:{{ .Data.password }}@postgres:5432/mydb?sslmode=disable
  {{- end }}
vault.hashicorp.com/role: 'app'
```

컨테이너가 참조하게되는 secret은 아래와 같습니다
```text
cat /vault/secrets/foo
postgres://v-kubernet-pg-app-q0Z7WPfVN:A1a-BUEuQR52oAqPrP1J@postgres:5432/mydb?sslmode=disable
```

template이 제공되지 않는다면 아래의 default template을 사용합니다
```
{{ with secret "/path/to/secret" }}
    {{ range $k, $v := .Data }}
        {{ $k }}: {{ $v }}
    {{ end }}
{{ end }}

```

예를 들어, 아래의 annotation은 설정된 경로를 찾기위해 default template을 사용합니다.
```yaml
vault.hashicorp.com/agent-inject-secret-foo: 'database/roles/pg-app'
vault.hashicorp.com/role: 'app'
```

컨테이너내의 렌더링된 secret은 아래와 같습니다
```text
$ cat /vault/secrets/foo
password: A1a-BUEuQR52oAqPrP1J
username: v-kubernet-pg-app-q0Z7WPfVNqqTJuoDqCTY-1576529094
```

### Renewals and Updating Secrets
Vault Agent의 secret renew와 fetch는 [Agent documentation](https://www.vaultproject.io/docs/agent/template#renewals-and-updating-secrets) 을 참조하세요

### Vault Agent Configuration Map
용례들을 더 깊게 사용하기 위해서, secret과 template annotation을 마운트 하는 대신에 Vault Agent 설정 파일의 정의가 요구될 수 있습니다. Vault Agent Injector는 `vault.hashicorp.com/agent-configmap` annotation을 사용하여 이름을 구체화 함으로써 ConfigMaps를 마운트 할 수 있습니다. Configuration 파일들은 `vault/configs`에 마운트 됩니다.
Configuration map은 이 중 하나나 두개 모두 포함해야합니다.
- **config-init.hcl** 은 init 컨테이너가 사용합니다. 이는 `exit_after_auth`가 `true`로 설정되어야 합니다
- **config.hcl** 은 sidecar container가 사용합니다. `exit_after_auth`가 `false`로 설정되어야 합니다.
Vault Agent configmap의 예시는 [여기](https://www.vaultproject.io/docs/platform/k8s/injector/examples#configmap-example) 를 참조해주세요.

## Learn
[Injecting Secrets into Kubernetes Pods via Vault Helm Sidecar](https://learn.hashicorp.com/tutorials/vault/kubernetes-sidecar) 에서 실습을 해봅시다!
