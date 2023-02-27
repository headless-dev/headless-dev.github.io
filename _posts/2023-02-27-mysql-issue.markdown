---
layout: post
title:  "[Bitnami-mysql] Bitnami mysql 운영 도중 password 에러가 발생 시"
date:   2023-02-27 16:03:36 +0900
categories: Infrastructure
---

K8S 혹은 가상 환경의 서버에서 mysql 을 손쉽게 운영하고 싶을 때 bitnami를 많이 사용한다.

K8S 환경의 bitnami mysql 운영 도중 발생한 이슈에 대해 해결 하는 방법에 대해 서술하고자 한다.

(Helm chart 기준)


# Access denied
```text
error: 'Access denied for user 'root'@'localhost' (using password: YES)'
```

운영 도중 혹은 재시작 도중 mysql 을 생성 시 해당 에러가 뜨며 서비스가 정상적으로 동직하지 않는 경우가 발생했다.

root password 를 그 누구도 변경하지 않았을 뿐더러 k8s의 'Secret' 이나 'Configmap' 에도 해당 password 가 변경된 이력이 전혀 없었다

원인은 나중에 찾고 우선 해결부터 진행했다.
<br><br>

1. skip grant tables 설정으로 root password 를 무시하고 서비스가 실행되도록 한다

```yaml
primary:
  configuration: |-
    [mysqld]
    skip-grant-tables
    ...
```
Helm chart의 values에서 추가해주면 된다.
<br><br>

2. Mysql Container의 bash shell로 attach 한다
```shell
kubectl exec -it pods/bitnami/mysql bash -n mysql
```
예시로 위와 같이 mysql 이 실행되고 있는 pod 에 들어간다.
<br><br>

```shell
mysql
```
mysql client를 바로 실행한다 grant 모드여서 바로 접근이 가능하다.
<br><br>

```mysql
FLUSH PRIVILEGES;
```
grant 모드의 테이블 권한을 사용할 수 있도록 서버에 리로드 명령어를 날린다.
<br><br>

```mysql
ALTER USER 'root'@'%' IDENTIFIED BY 'mypassword';
or
ALTER USER 'root'@'localhost' IDENTIFIED BY 'mypassword';
```
root password를 기존에 설정해두었던 비밀번호로 다시 변경한다.
<br><br>

```yaml
primary:
  configuration: |-
    [mysqld]
    #skip-grant-tables
    ...
```
mysqld 옵션에서 'skip-grant-tables'을 제거하고 다시 시작 후 정상 동작을 확인한다.
<br><br><br>
혹여나 이러한 설정으로 문제 해결이 안될 시 다음과 같은 상황을 확인해야 한다.

1. 위와 같이 강제로 비밀번호를 변경한 경우 secret 은 변경사항을 sync 하지 않고 재시작 해본다.
2. root 계정의 접근 dns localhost 가 pod 내의 dns에 등록되어 접근 가능한지 확인 한다.
3. password를 설정하는 파일의 액세스 권한을 확인한다.

<br><br>
발생 원인은 정확히 찾지 못하였으나 root password 가 어떤 이유에 의해 자동으로 변경되는 것으로 보인다.

이번엔 문제 해결에 초점을 맞추어 원인 분석을 못 했지만

다음에 같은 문제가 발생한다면 원인 분석을 할 예정이다.