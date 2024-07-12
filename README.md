# aws-iam-authenticator-k3d-demo
Quick and sloppy demo setting up aws-iam-authenticator on k3d.

## Steps

Install aws-iam-authenticator.
```
brew install aws-iam-authenticator
```

Create an IAM role that can be assumed with your AWS creds. Follow guidance [here](https://github.com/kubernetes-sigs/aws-iam-authenticator?tab=readme-ov-file#1-create-an-iam-role).  Save this role arn you created as it will need to be passed into the helm chart.

In order to run aws-iam-authenticator without restarting the kube-api-server, you will need to generate a certificate pair and kubeconfig that needs to be bootstrapped onto the master nodes.  The aws-iam-authenticator helm chart expects the certs to be in specific locations on the master nodes where the kube-apiserver is running:

- `/etc/kubernetes/aws-iam-authenticator/` should hold the kubeconfig
- `/var/aws-iam-authenticator/` should hold the certs

You can generate the certs and kubeconfig with the following command locally:

```bash
aws-iam-authenticator init --cluster-id aws-test # for testing purposes we will use aws-test as the cluster-id
# This command can unfortunately only output the files to the current directory and will need to be moved

# Move the certs to a folder that can be mounted into the k3d cluster
mkdir -p certs
mv cert.pem certs/
mv key.pem certs/
export AWS_IAM_AUTHENTICATOR_CERTS_DIR=$(pwd)/certs

# Move the kubeconfig to a folder that can be mounted into the k3d cluster
mkdir -p kubeconfig
mv aws-iam-authenticator.kubeconfig kubeconfig/kubeconfig.yaml
export AWS_IAM_AUTHENTICATOR_KUBECONFIG_DIR=$(pwd)/kubeconfig
```

We can now launch k3d cluster and mount these certs and specify the kube-apiserver flag to use the aws-iam-authenticator as an authorization webhook.

Take a look and examine the `k3d-config.yaml` file. This config file supplies the extra kube-apiserver flags to use the aws-iam-authenticator as an authentication webhook. 

Create a local cluster with k3d and supply the volume mounts for the certs and kubeconfig:
```bash
k3d cluster create --config k3d-config.yaml --volume "$AWS_IAM_AUTHENTICATOR_CERTS_DIR:/var/aws-iam-authenticator@server:*" --volume "$AWS_IAM_AUTHENTICATOR_KUBECONFIG_DIR:/etc/kubernetes/aws-iam-authenticator@server:*"
```

Now we can deploy the aws-iam-authenticator helm chart.  

You will need to edit the `chart/values.yaml` file to add in the role you want to map.

```yaml
    mapRoles:
    - roleARN: arn:aws:iam::AWS_ACCOUNT:role/your-role # Edit this line to your role arn you created earlier
      username: kubernetes-admin
      groups:
      - system:masters
```

Deploy the helm chart:

```bash
# Before deploying the chart... load docker image into k3d (as it comes from ECR)
# Must have any valid AWS creds to pull from this image down
# Login to the ECR registry that hosts this image
aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 602401143452.dkr.ecr.us-west-2.amazonaws.com

docker pull 602401143452.dkr.ecr.us-west-2.amazonaws.com/amazon/aws-iam-authenticator:v0.5.9
docker tag 602401143452.dkr.ecr.us-west-2.amazonaws.com/amazon/aws-iam-authenticator:v0.5.9 aws-iam-authenticator:v0.5.9
k3d image import -c test-aws aws-iam-authenticator:v0.5.9

# Deploy aws-iam-authenticator helm chart (which deploys daemonset and configmap to cluster)
helm install aws-iam-authenticator ./chart
```

## Test

Set your kubeconfig user and set the context to use this user:

```yaml
users:
- name: kubernetes-admin
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "REPLACE_ME_WITH_YOUR_CLUSTER_ID"
        - "-r"
        - "REPLACE_ME_WITH_YOUR_ROLE_ARN"
```

Test that you can connect to the cluster:

```bash
kubectl get pods
```
