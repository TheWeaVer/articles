#### Original: https://www.vaultproject.io/docs/platform/k8s/helm/examples/kubernetes-auth

# Bootstrapping Kubernetes Auth Method

[Kubernetes Auth Method](https://www.vaultproject.io/docs/auth/kubernetes) 의 set up에 대해서 설명합니다.
아래의 커맨드들은 Kubernetes 내에서 기동중인 Vault pod에서 실행함을 가정합니다.
```
# JWT is a service account token that has access to the Kubernetes TokenReview API
# You can retrieve this from inside a pod at: /var/run/secrets/kubernetes.io/serviceaccount/token
JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

# Address of Kubernetes itself as viewed from inside a running pod
KUBERNETES_HOST=https://${KUBERNETES_PORT_443_TCP_ADDR}:443

# Kubernetes internal CA
KUBERNETES_CA_CERT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt)

```

pod에서 실행할 경과
```
kubectl exec -it vault-0 /bin/sh
```

Kubernetes Auth Method를 설정하는 커맨드의 실행
```
vault write auth/kubernetes/config \
   token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
   kubernetes_host=https://${KUBERNETES_PORT_443_TCP_ADDR}:443 \
   kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

# Kubernetes Auth Method
`kubernetes` auth method는 Kubernetes Service Account Token을 사용하는 Vault를 인증할 수 있습니다. 이 인증 수단은 Kubernetes Pod내로 Vault token을 쉽제 주입할 수 있게 합니다.
또한, 당신은 [JWT auth login](https://www.vaultproject.io/docs/auth/jwt/oidc_providers#kubernetes) 에 Kubernetes Service Account를 사용할 수 있습니다. 이번 장에서는 [짧은 생명을 갖는 Kubernetes token](https://www.vaultproject.io/docs/auth/kubernetes#how-to-work-with-short-lived-kubernetes-tokens) 에 대한 요약과 JWT auth를 대체하고 Kubernetes와의 비교를 볼 것입니다.
```
Note: If you are upgrading to Kubernetes v1.21+, ensure the config option disable_iss_validation is set to true. Assuming the default mount path, you can check with vault read -field disable_iss_validation auth/kubernetes/config. See Kubernetes 1.21 below for more details.
```

## Authentication
### Via the CLI
기본 path는 `/kubernetes`입니다. 다른 path에도 활성화 했다면, `-path=/my-path`를 CLI에 넣어야 합니다.
```
vault write auth/kubernetes/login role=demo jwt=...
```

### Via the API
기본 endpoint는 `auth/kubernetes/login` 입니다. 다른 path에 auth method가 활성화 되었다면, `kubernetes` 대신에 그 값을 사용하면 됩니다.
```
curl \
    --request POST \
    --data '{"jwt": "<your service account jwt>", "role": "demo"}' \
    http://127.0.0.1:8200/v1/auth/kubernetes/login

```

`auth.clinet_token`에 token을 포함한 응답입니다.
```
{
  "auth": {
    "client_token": "38fe9691-e623-7238-f618-c94d4e7bc674",
    "accessor": "78e87a38-84ed-2692-538f-ca8b9f400ab3",
    "policies": ["default"],
    "metadata": {
      "role": "demo",
      "service_account_name": "vault-auth",
      "service_account_namespace": "default",
      "service_account_secret_name": "vault-auth-token-pd21c",
      "service_account_uid": "aa9aa8ff-98d0-11e7-9bb7-0800276d99bf"
    },
    "lease_duration": 2764800,
    "renewable": true
  }
}
```

## Configuration
Auth method는 사용자나 머신이 인증을 할 수 있기 전에 사전에 구성되어야 합니다. 일반적으로 amdin이나 configuration tool을 사용하여 configuration을 할 수 있습니다.
1. Kubernetes auth method를 활성화합니다.
```
vault auth enable kubernetes

```

2. `/config` endpoint를 Vault을 설정하고 Kubernetes와 소통하기 위해 사용합니다. `kubectl cluster-info`를 Kubernetes host 주소와 TCP port를 검증하기위해 사용합니다. 사용가능한 설정 옵션의 리스트를 위해, [API Documentation](https://www.vaultproject.io/api/auth/kubernetes) 을 확인해주세요
```
vault write auth/kubernetes/config \
    token_reviewer_jwt="<your reviewer service account JWT>" \
    kubernetes_host=https://192.168.99.100:<your TCP port or blank for 443> \
    kubernetes_ca_cert=@ca.crt
```

```
Note: The pattern Vault uses to authenticate Pods depends on sharing the JWT token over the network. Given the security model of Vault, this is allowable because Vault is part of the trusted compute base. In general, Kubernetes applications should not share this JWT with other applications, as it allows API calls to be made on behalf of the Pod and can result in unintended access being granted to 3rd parties.```
```

3. Role 이름을 생성합니다
```
vault write auth/kubernetes/role/demo \
    bound_service_account_names=vault-auth \
    bound_service_account_namespaces=default \
    policies=default \
    ttl=1h

```

이 role은 default namespace의 `vault-auth` 서비스 어카운트를 인가하며, default policy로 정의합니다.
configuration option을 마무리하기 위해서는, [API documentation](https://www.vaultproject.io/api/auth/kubernetes) 을 확인바랍니다.
