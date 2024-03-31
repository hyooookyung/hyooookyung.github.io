---
layout: default
title: PrometheusRule 쿼리 에러
parent: Troubleshooting
nav_order: 1
---

## PrometheusRule 쿼리 에러 트러블슈팅 기록
운영 환경에서 발생한 PrometheusRule의 Error 상태를 정상화한 것에 대한 기록입니다.

### 이슈(현상)
* PrometheusRuleFailures 알람 발생
* Prometheus 기본 알람 중 하나인 kubelet.rules 규칙의 헬스 상태가 error 상태임을 확인
* kubelet.rules 알람에 정의된 expression 쿼리를 실행할 때 에러가 발생
    - 에러 메세지: `Error executing query: found duplicate series for the match group ..`
* 에러 메세지 내용을 그대로 해석해서 PromQL 쿼리 결과 값에 1개 이상의 데이터가 조회되어 문제가 발생한다는 것을 확인
* 해당 PrometheusRule에 정의된 쿼리 확인 시 kubelet_node_name{} 메트릭에서 동일한 노드의 데이터가 중복으로 조회되고 있음을 확인

### 조치
* kubelet_node_name{} 쿼리 수집 시 활용되는 ServiceMonitor 리소스를 확인 후 kubelet 서비스 최신화
* `kubectl get svc -A -l k8s-app=kubelet` 명령어로 조회 시 프로메테우스에서 설치한 서비스 중 예전 버전의 헬름 차트에서 설치한 서비스가 존재하는 것을 확인
* 현재는 제거된 헬름 차트에서 남아있는 미사용 서비스(orphan resource) 삭제 처리
* kubelet_node_name{} 쿼리를 통해 삭제 이후 시점에 노드 당 1개의 데이터가 정상적으로 수집되는 것을 확인

### 원인 파악
* kubelet의 ServiceMonitor에서 활용하는 k8s-app:kubelet 라벨이 1개 이상의 서비스에 할당되어 데이터가 중복해서 수집되고 있었음
* 프로메테우스 활용 시 bitnami-kube-prometheus 헬름 차트에서 kube-prometheus-stack 헬름 차트로 변경 후 예전 헬름 차트의 리소스가 모두 삭제되지 않아 남아있어 이슈가 발생
* PromQL 쿼리에서 group_left 쿼리 실행 시 중복된 데이터(2개)가 조회되어 이슈에서 파악한 에러 메세지가 발생
* ServiceMonitor에 설정된 matchLabel 설정 값
```
  selector:
    matchLabels:
      app.kubernetes.io/name: kubelet
      k8s-app: kubelet
```
* ServiceMonitor matchLabel 기준으로 조회된 kubelet 서비스 목록
```
$ kubectl get svc -n kube-system -l k8s-app=kubelet
NAME                                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                        AGE
kube-prometheus-stack-kubelet        ClusterIP   None         <none>        10250/TCP,10255/TCP,4194/TCP   93d
prometheus-kube-prometheus-kubelet   ClusterIP   None         <none>        10250/TCP,10255/TCP,4194/TCP   100d
```