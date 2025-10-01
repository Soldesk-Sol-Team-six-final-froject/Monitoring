본 스크립트는 Bastion Host 등 **AWS API 호출(aws CLI)** 과 **Kubernetes 제어(kubectl)** 를 모두 실행할 수 있는 환경을 기준으로 작성되었습니다.

필요 권한:
- 실행 환경은 EKS 클러스터에 Access Entry를 생성·연결할 수 있고,
  IAM 역할/정책을 생성·수정할 수 있는 AWS 자격증명(IAM Role)을 사용해야 합니다.

필요 도구:
- aws CLI
- kubectl
(사전 설치 필요)

실행 전 설정:
- 스크립트 상단의 A-0 변수 섹션(REGION, CLUSTER, S3 버킷 등)을
  사용자 환경에 맞게 수정 후 실행하세요.



# A) 공통 변수 & 사전 준비

#!/usr/bin/env bash
set -euo pipefail

############################
# A-0. 변수
############################
export REGION= # 서비스 리전 입력
export CLUSTER= "" # 자신의 클러스터 네임 입력
export ACC_ID=$(aws sts get-caller-identity --query 'Account' --output text)

# (추가됨) 바스천 역할 ARN – Access Entry 연결용
export BASTION_ROLE_ARN="arn:aws:iam::${ACC_ID}:role/my-eks-shop-cluster-bastion-role"

# S3 (모니터링)
export THANOS_BUCKET='' # 버킷 네임 입력
export LOKI_BUCKET='' # 버킷 네임 입력

# ServiceAccount
export NS=monitoring
export OBS_SA=obs-sa        # 모니터링용
export EBS_NS=kube-system
export EBS_SA=ebs-csi-controller-sa

# IAM Role
export OBS_ROLE=EKS-Obs-Role
export EBS_ROLE=EKS-EBS-CSI-Role
export MONITORING_POLICY_NAME=monitoring-policy1   # 선택(나중에 최소권한으로 교체 시 사용)



############################
# A-1. kubeconfig / 네임스페이스
############################
aws eks update-kubeconfig --region "$REGION" --name "$CLUSTER"

kubectl get ns "$NS" >/dev/null 2>&1 || kubectl create ns "$NS"
kubectl -n "$NS" get sa "$OBS_SA" >/dev/null 2>&1 || kubectl -n "$NS" create sa "$OBS_SA"

# (추가됨) EKS Access Entry + ClusterAdmin 정책 연결 (system:masters 사용금지)
aws eks create-access-entry \
  --cluster-name "$CLUSTER" --region "$REGION" \
  --principal-arn "$BASTION_ROLE_ARN" \
  --type STANDARD 2>/dev/null || true

aws eks associate-access-policy \
  --cluster-name "$CLUSTER" --region "$REGION" \
  --principal-arn "$BASTION_ROLE_ARN" \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster

aws eks list-associated-access-policies \
  --cluster-name "$CLUSTER" --region "$REGION" \
  --principal-arn "$BASTION_ROLE_ARN" \
  --query 'associatedAccessPolicies[].{policy:policyArn,scope:accessScope}'


# > 주의(유지): system:masters 그룹 매핑은 예약 접두어라 금지. 위 Access Policy 방식 사용.




# B) Pod Identity 신뢰정책(정답본) 준비


cat > /tmp/podid-trust-min.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowEksAuthToAssumeRoleForPodIdentity",
    "Effect": "Allow",
    "Principal": { "Service": "pods.eks.amazonaws.com" },
    "Action": ["sts:AssumeRole","sts:TagSession"]
  }]
}
JSON


# C) IAM 역할 생성/갱신 (idempotent)


############################
# C-1. EKS-Obs-Role (파드가 S3 접근)
############################
aws iam get-role --role-name "$OBS_ROLE" >/dev/null 2>&1 || \
aws iam create-role --role-name "$OBS_ROLE" \
  --assume-role-policy-document file:///tmp/podid-trust-min.json

aws iam update-assume-role-policy --role-name "$OBS_ROLE" \
  --policy-document file:///tmp/podid-trust-min.json

# 테스트 단계: S3 FullAccess (운영 전 최소권한으로 대체)
aws iam attach-role-policy --role-name "$OBS_ROLE" \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess || true

export OBS_ROLE_ARN=$(aws iam get-role --role-name "$OBS_ROLE" --query 'Role.Arn' --output text)
echo "OBS_ROLE_ARN=$OBS_ROLE_ARN"



############################
# C-2. EKS-EBS-CSI-Role (EBS CSI 컨트롤러용)
############################
aws iam get-role --role-name "$EBS_ROLE" >/dev/null 2>&1 || \
aws iam create-role --role-name "$EBS_ROLE" \
  --assume-role-policy-document file:///tmp/podid-trust-min.json

aws iam update-assume-role-policy --role-name "$EBS_ROLE" \
  --policy-document file:///tmp/podid-trust-min.json

aws iam attach-role-policy --role-name "$EBS_ROLE" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy || true

export EBS_ROLE_ARN=$(aws iam get-role --role-name "$EBS_ROLE" --query 'Role.Arn' --output text)
echo "EBS_ROLE_ARN=$EBS_ROLE_ARN"




# D) Pod Identity Association 수동 연결


############################
# D-1. 모니터링 SA ↔ EKS-Obs-Role
############################
OLD_ID=$(aws eks list-pod-identity-associations --region "$REGION" --cluster-name "$CLUSTER" \
  --query 'associations[?namespace==`'"$NS"'` && serviceAccount==`'"$OBS_SA"'`].associationId' --output text) || true
[ -n "${OLD_ID:-}" ] && aws eks delete-pod-identity-association --region "$REGION" \
  --cluster-name "$CLUSTER" --association-id "$OLD_ID" || true

aws eks create-pod-identity-association --region "$REGION" \
  --cluster-name "$CLUSTER" --namespace "$NS" --service-account "$OBS_SA" \
  --role-arn "$OBS_ROLE_ARN"

# (추가됨) roleArn이 정확히 들어갔는지 원자료로 검증
ASSOC_ID=$(aws eks list-pod-identity-associations --region "$REGION" --cluster-name "$CLUSTER" \
  --query 'associations[?namespace==`'"$NS"'` && serviceAccount==`'"$OBS_SA"'`].associationId' --output text)
aws eks describe-pod-identity-association --region "$REGION" --cluster-name "$CLUSTER" \
  --association-id "$ASSOC_ID" \
  --query 'association.{ns:namespace,sa:serviceAccount,role:roleArn,owner:ownerArn}'



############################
# D-2. EBS CSI 애드온 ↔ EKS-EBS-CSI-Role (정석)
############################
aws eks update-addon --region "$REGION" --cluster-name "$CLUSTER" \
  --addon-name aws-ebs-csi-driver \
  --pod-identity-associations "serviceAccount=${EBS_SA},roleArn=${EBS_ROLE_ARN}"

aws eks describe-addon --region "$REGION" --cluster-name "$CLUSTER" \
  --addon-name aws-ebs-csi-driver --query 'addon.status'

aws eks list-pod-identity-associations --region "$REGION" --cluster-name "$CLUSTER" \
  --query 'associations[?namespace==`kube-system` && serviceAccount==`ebs-csi-controller-sa`].{ns:namespace,sa:serviceAccount,role:roleArn,owner:ownerArn}'




# E) StorageClass(gp3) 생성(기본 지정) + 드라이버 상태 확인


cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

kubectl get sc gp3 -o jsonpath='{.provisioner}{" "}{.volumeBindingMode}{"\n"}'
kubectl -n kube-system get deploy ebs-csi-controller -o jsonpath='{.status.readyReplicas}{"\n"}'
kubectl -n kube-system get ds ebs-csi-node -o jsonpath='{.status.numberReady}{"\n"}'




# F) Helm 리포지토리 & 모니터링 스택 설치


helm repo add prometheus-community https://prometheus-community.github.io/helm-charts || true
helm repo add grafana https://grafana.github.io/helm-charts || true
helm repo add bitnami https://charts.bitnami.com/bitnami || true
helm repo update



# Alertmanager Slack 비밀/설정
kubectl -n "$NS" create secret generic slack-webhook \
  --from-literal=url='' #Slack 주소 입력  2>/dev/null || true
kubectl -n "$NS" apply -f alertmanager-config.yaml



# Thanos objstore Secret (Pod Identity와 별개로 Thanos 구성값 필요)
cat >/tmp/objstore.yml <<EOF
type: S3
config:
  bucket: ${THANOS_BUCKET}
  endpoint: s3.${REGION}.amazonaws.com
  region: ${REGION}
  insecure: false
EOF
kubectl -n "$NS" create secret generic thanos-objstore \
  --from-file=objstore.yml=/tmp/objstore.yml --dry-run=client -o yaml | kubectl apply -f -



# kube-prometheus-stack (values에 serviceAccount: obs-sa 반영)
helm upgrade --install kps prometheus-community/kube-prometheus-stack \
  -n "$NS" -f kps-values.yaml

# Prometheus ↔ Thanos gRPC 서비스
kubectl -n "$NS" apply -f prometheus-thanos-grpc.yaml




# Thanos (storegateway/전역 SA 모두 obs-sa로 강제)  ← (추가됨) 핵심 옵션
helm upgrade --install thanos-store bitnami/thanos -n "$NS" \
  -f thanos-store-values.yaml \
  --set serviceAccount.create=false \
  --set serviceAccount.name=obs-sa \
  --set storegateway.serviceAccount.create=false \
  --set storegateway.serviceAccount.name=obs-sa \
  --set existingObjstoreSecret=thanos-objstore \
  --set existingObjstoreSecretItems[0].key=objstore.yml \
  --set existingObjstoreSecretItems[0].path=objstore.yml

helm upgrade --install thanos-compact bitnami/thanos -n "$NS" \
  -f thanos-compact-values.yaml \
  --set serviceAccount.create=false \
  --set serviceAccount.name=obs-sa \
  --set existingObjstoreSecret=thanos-objstore \
  --set existingObjstoreSecretItems[0].key=objstore.yml \
  --set existingObjstoreSecretItems[0].path=objstore.yml

helm upgrade --install thanos-query bitnami/thanos -n "$NS" \
  -f thanos-query-values.yaml \
  --set serviceAccount.create=false \
  --set serviceAccount.name=obs-sa



# Loki (values에 serviceAccount: obs-sa, S3 설정은 values로)
helm upgrade --install loki grafana/loki-distributed -n "$NS" \
  -f loki-aws-values.yaml

helm upgrade --install promtail grafana/promtail -n "$NS" \
  -f promtail-values.yaml




# G) 동작 확인 & 포트포워딩(옵션)


kubectl -n "$NS" get pods -o wide
kubectl -n "$NS" get pvc,pv
kubectl -n "$NS" get svc

# (옵션) 포트포워드
kubectl -n "$NS" port-forward svc/thanos-query-query 10902:10902 &
kubectl -n "$NS" port-forward svc/kps-grafana 3000:80 &
kubectl -n "$NS" port-forward svc/loki-loki-distributed-gateway 3100:80 &
kubectl -n "$NS" port-forward svc/kps-alertmanager 9093:9093 &
# Monitoring
