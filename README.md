# Gitea 설치 가이드

## 개요
- Gitea
	- Gitea is a painless self-hosted Git service.
## Prerequisites
- Kubernetes, Helm3

## 설치 가이드
1. Gitea Chart Museum 추가
	```bash
	helm repo add gitea-charts-tmax https://tmax-cloud.github.io/charts/gitea
	```

2. Keycloak 연동 시
	1. Keycloak에서 클라이언트 생성
	- Name: gitea
	- Client-Protocol: openid-connect
	- AccessType: confidential
	- Valid Redirect URIs: *

	2. 클라이언트 시크릿 복사
	- Client > gitea> Credentials > Secret

	3. values.yaml 설정
	```yaml
	gitea:
	  oauth:
	    - name: 'giteaOAuth' # oauth 이름
	      provider: 'openidConnect' # 클라이언트 프로토콜
	      key: 'gitea' # Keycloak 클라이언트 이름
	      secret: '******' # Keycloak 클라이언트 시크릿
	      autoDiscoverUrl: 'https://{keycloakhost}:{keycloakport}/auth/realms/{realm}/.well-known/openid-configuration' # Keycloak의 autoDiscoverUrl
	                            ex) 'https://hyperauth.tmaxcloud.org/auth/realms/tmax/.well-known/openid-configuration' # hyperauth의 경우 사용 예시
	```

3. 추가 values.yaml 설정
	```yaml
	gitea:
	  config:
	    server:
	      PROTOCOL: http
	      DOMAIN: gitea.testdomain.com # 도메인 설정
	      ROOT_URL: http://gitea.testdomain.com # root url 설정
	      SSH_DOMAIN: gitea.testdomain.com # ssh 도메인 설정
	```
	
	```yaml
	ingress:
	  hosts:
	    domain: testdomain.com # 도메인 설정
	    subdomain: gitea # 서브도메인 설정
	```

4. Chart 설치
	```bash
	kubectl create namespace gitea-system
	helm install gitea -f values.yaml gitea-charts-tmax/gitea -n gitea-system
	```

## 삭제 가이드
1. Chart 삭제
```bash
$ helm uninstall gitea -n gitea-system
$ k delete ns gitea-system
```

## Hyperauth selfsigned CA 설정
1. TLS 시크릿 생성
```bash
$ KEYCLOAK_CERT_FILE=/etc/kubernetes/pki/hypercloud-root-ca.crt 
$ KEYCLOAK_TLS_SECRET_NAME=<시크릿 이름>

$ kubectl create ns gitea-system

$ cat <<EOT | kubectl apply -n gitea-system -f -
apiVersion: v1
kind: Secret
metadata:
  name: $KEYCLOAK_TLS_SECRET_NAME
type: kubernetes.io/tls
data:
  tls.crt: $(cat $KEYCLOAK_CERT_FILE | base64 -w 0)
  tls.key: $(echo -n 'dummyKey' | base64 -w 0)
EOT
```

2. 추가 values.yaml 설정 
```yaml
extraVolumes:
  - name: selfsigned-ca     
    secret:
      secretName: <시크릿 >
```

```yaml
extraVolumeMounts:
  - name: selfsigned-ca   
    readOnly: true
    mountPath: /etc/ssl/certs 
```

## 폐쇄망 설치 가이드
1. 폐쇄망에서 설치하는 경우 사용하는 image를 다운받고 저장합니다.
   - 작업 디렉토리 생성 및 환경 설정

   ```bash
   mkdir -p ~/gitea-install
   export GITEA_HOME=~/gitea-install
   cd $GITEA_HOME
   ```

   - 외부 네트워크 통신이 가능한 환경에서 필요한 이미지를 다운받습니다.

   ```bash
   # gitea 이미지 Pull
   docker image pull gitea/gitea:1.16.8
   docker image pull docker.io/bitnami/memcached:1.6.9-debian-10-r114
   docker image pull docker.io/bitnami/memcached-exporter:0.8.0-debian-10-r105
   docker image pull docker.io/bitnami/postgresql:11.11.0-debian-10-r62
   docker image pull docker.io/bitnami/bitnami-shell:10
   docker image pull docker.io/bitnami/postgres-exporter:0.9.0-debian-10-r34
   
   # gitea 이미지 Save
   docker save gitea/gitea:1.16.8 > gitea.tar
   docker save docker.io/bitnami/memcached:1.6.9-debian-10-r114 > gitea-memcached.tar
   docker save docker.io/bitnami/memcached-exporter:0.8.0-debian-10-r105 > gitea-memcached-exporter.tar
   docker save docker.io/bitnami/postgresql:11.11.0-debian-10-r62 > gitea-postgresql.tar
   docker save docker.io/bitnami/bitnami-shell:10 > gitea-bitnami-shell.tar
   docker save docker.io/bitnami/postgres-exporter:0.9.0-debian-10-r34 > gitea-postgres-exporter.tar
   ```
   
2. 폐쇄망으로 파일(.tar)을 옮깁니다.

3. 폐쇄망에서 .tar 압축을 풀고 설치합니다.

   ```bash
   # 이미지 레지스트리 주소
   REGISTRY=[IP:PORT]

   # 이미지 Load
   docker load < gitea.tar
   docker load < gitea-memcached.tar
   docker load < gitea-memcached-exporter.tar
   docker load < gitea-postgresql.tar
   docker load < gitea-bitnami-shell.tar
   docker load < gitea-postgres-exporter.tar
   
   # 이미지 Tag
   docker tag gitea/gitea:1.16.8 ${REGISTRY}/gitea/gitea:1.16.8
   docker tag docker.io/bitnami/memcached:1.6.9-debian-10-r114 ${REGISTRY}/docker.io/bitnami/memcached:1.6.9-debian-10-r114
   docker tag docker.io/bitnami/memcached-exporter:0.8.0-debian-10-r105 ${REGISTRY}/docker.io/bitnami/memcached-exporter:0.8.0-debian-10-r105
   docker tag docker.io/bitnami/postgresql:11.11.0-debian-10-r62 ${REGISTRY}/docker.io/bitnami/postgresql:11.11.0-debian-10-r62
   docker tag docker.io/bitnami/bitnami-shell:10 ${REGISTRY}/docker.io/bitnami/bitnami-shell:10
   docker tag docker.io/bitnami/postgres-exporter:0.9.0-debian-10-r34 ${REGISTRY}/docker.io/bitnami/postgres-exporter:0.9.0-debian-10-r34

   # 이미지 Push
   docker push ${REGISTRY}/gitea/gitea:1.16.8
   docker push ${REGISTRY}/docker.io/bitnami/memcached:1.6.9-debian-10-r114
   docker push ${REGISTRY}/docker.io/bitnami/memcached-exporter:0.8.0-debian-10-r105
   docker push ${REGISTRY}/docker.io/bitnami/postgresql:11.11.0-debian-10-r62
   docker push ${REGISTRY}/docker.io/bitnami/bitnami-shell:10
   docker push ${REGISTRY}/docker.io/bitnami/postgres-exporter:0.9.0-debian-10-r34
   ```

4. values.yaml의 global.registry 수정
   gitea, memcached, postgresql 세 개의 values.yaml 을 수정한다.
   
   install-gitea/values.yaml
   install-gitea/charts/memcached/values.yaml
   install-gitea/charts/postgresql/values.yaml
   
   ```bash
   global:
     registry:
       is_offline: true # true로 수정
       private_registry: test-registry.com # private_registry 주소 입력
   ```
