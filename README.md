# Installacion tools

### Jwt-cli
```bash
brew install mike-engel/jwt-cli/jwt-cli
```

### Kubectl 1.23
```bash
wget kubernetes-client-darwin-amd64.tar.gz https://dl.k8s.io/v1.23.13/kubernetes-client-darwin-amd64.tar.gz
tar -xvzf kubernetes-client-darwin-amd64.tar.gz
mv ./kubernetes/client/bin/kubectl /user/local/bin/
```

### Install eksctl (coge las credenciales de ~/.aws/credentials o en variable de entorno)
```
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
eksctl version
```

# Configure client
```bash
aws eks update-kubeconfig --name cluster-eks02 --region eu-west-1
kubectl config current-context
```

# Create cluster
```bash
eksctl create cluster \
--name eks-oidc-demo \
--region us-east-2
```
Si lo vas a crear a mano y los pods de aws-node no arranca hay que configurar el cni-iam-eksctl (https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html#configure-cni-iam-eksctl)

```bash
eksctl utils associate-iam-oidc-provider --region=eu-west-1 --cluster=cluster-eks02 --approve

eksctl create iamserviceaccount \
    --name aws-node \
    --namespace kube-system \
    --cluster cluster-eks02 \
    --role-name "AmazonEKSVPCCNIRole" \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
    --override-existing-serviceaccounts \
    --approve
```
Recrear pods
```bash
kubectl delete pods -n kube-system -l k8s-app=aws-node
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl exec -n kube-system aws-node-9rgzw -c aws-node -- env | grep AWS
```

#Unauthorized cuando configuramos el kubectl
Solo podemos acceder con el usuario que creo el cluster. Si queremos acceder con otros usuarios hacer

```bash
kubectl edit configmap aws-auth --namespace kube-system
```
Añadir las siguiente linea
```bash
mapUsers: |
- userarn: arn:aws:iam::909383344669:user/jorgehernandezramirez
  username: jorgehernandezramirez
    groups:
       - system:masters
```
https://aws.amazon.com/es/premiumsupport/knowledge-center/eks-api-server-unauthorized-error/

# Crear iamserviceaccount

Se creará
* Service account en k8s
* Role and Trusted policy

```bash
eksctl create iamserviceaccount \
  --name my-sa2 \
  --namespace default \
  --cluster cluster-eks02 \
  --approve \
  --attach-policy-arn $(aws iam list-policies --query 'Policies[?PolicyName==`AmazonS3ReadOnlyAccess`].Arn' --output text) 
 ```

# Execute pods
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: eks-iam-test3
spec:
  serviceAccountName: my-sa 
  containers:
    - name: my-aws-cli
      image: amazon/aws-cli:latest
      command: ['sleep', '36000']
  restartPolicy: Never
EOF
```

# Veriricar que el jwt inyectado es el correcto

```bash
IAM_TOKEN=$(kubectl exec -it eks-iam-test3 -- cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token)
IDP=$(aws eks describe-cluster --name eks-oidc-demo --query cluster.identity.oidc.issuer --output text)
curl -s $IDP/.well-known/openid-configuration | jq -r '.'
```
Finalmente asumimos el rol
```bash
aws sts assume-role-with-web-identity \
--role-arn $(kubectl get sa/my-sa -o json | jq -r '.metadata.annotations."eks.amazonaws.com/role-arn"') \
--role-session-name RoleAssumedJwt \
--web-identity-token $IAM_TOKEN
```

# Crear identity provider a mano. Si lo hacemos a mano especificar
* URL: https://oidc.eks.eu-west-1.amazonaws.com/id/DA048F671E6310F4D8E205302594BABA
* Audience: sts.amazonaws.com

# Referencias
* https://aws.amazon.com/es/blogs/aws-spanish/configurando-una-conexion-vpn-sitio-a-sitio-entre-aws-y-azure/
* https://aws.amazon.com/blogs/containers/diving-into-iam-roles-for-service-accounts/
* https://aws.amazon.com/es/premiumsupport/knowledge-center/eks-api-server-unauthorized-error/