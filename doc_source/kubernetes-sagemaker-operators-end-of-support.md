# Old SageMaker Operators for Kubernetes<a name="kubernetes-sagemaker-operators-end-of-support"></a>

This section is based on the original version of [SageMaker Operators for Kubernetes](https://github.com/aws/amazon-sagemaker-operator-for-k8s)\.

**Important**  
We are stopping the development and technical support of [SageMaker Operators for Kubernetes](https://github.com/aws/amazon-sagemaker-operator-for-k8s/tree/master) in its original version\. If you are currently using version `v1.2.2` or below of the original version of SageMaker Operators for Kubernetes, we recommend migrating your resources to the latest SageMaker Operators for Kubernetes, the [ACK service controller for Amazon SageMaker](https://github.com/aws-controllers-k8s/sagemaker-controller) based on [AWS Controllers for Kubernetes \(ACK\)](https://aws-controllers-k8s.github.io/community/ )\.  
For information about the migration steps, see [Migrate resources to the latest Operators](kubernetes-sagemaker-operators-migrate.md)\. For answers to frequently asked questions regarding the end of support of the original version of SageMaker Operators for Kubernetes, see [Announcing the End of Support of the Original Version of SageMaker Operator for Kubernetes](kubernetes-sagemaker-operators-eos-announcement.md)

**Topics**
+ [Install SageMaker Operators for Kubernetes](#kubernetes-sagemaker-operators-eos-install)
+ [Using Amazon SageMaker Jobs](#kubernetes-sagemaker-jobs)

## Install SageMaker Operators for Kubernetes<a name="kubernetes-sagemaker-operators-eos-install"></a>

Use the following steps to install and use SageMaker Operators for Kubernetes to train, tune, and deploy machine learning models with Amazon SageMaker\.

**Topics**
+ [IAM role\-based setup and operator deployment](#iam-role-based-setup-and-operator-deployment)
+ [Clean up resources](#cleanup-operator-resources)
+ [Delete operators](#delete-operators)
+ [Troubleshooting](#troubleshooting)
+ [Images and SMlogs in each Region](#images-and-smlogs-in-each-region)

### IAM role\-based setup and operator deployment<a name="iam-role-based-setup-and-operator-deployment"></a>

The following sections describe the steps to set up and deploy the original version of the operator\.

**Warning**  
**Reminder:** The following steps do not install the latest version of SageMaker Operators for Kubernetes\. To install the new ACK\-based SageMaker Operators for Kubernetes, see [Latest SageMaker Operators for Kubernetes](kubernetes-sagemaker-operators-ack.md)\.

#### Prerequisites<a name="prerequisites"></a>

This guide assumes that you have completed the following prerequisites: 
+ Installed the following tools on the client machine used to access your Kubernetes cluster: 
  + [https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) Version 1\.13 or later\. Use a `kubectl` version that is within one minor version of your Amazon EKS cluster control plane\. For example, a 1\.13 `kubectl` client works with Kubernetes 1\.13 and 1\.14 clusters\. OpenID Connect \(OIDC\) is not supported in versions earlier than 1\.13\. 
  + [https://github.com/weaveworks/eksctl](https://github.com/weaveworks/eksctl) Version 0\.7\.0 or later 
  + [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv1.html) Version 1\.16\.232 or later 
  + \(optional\) [Helm](https://helm.sh/docs/intro/install/) Version 3\.0 or later 
  + [aws\-iam\-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html) 
+ Have IAM permissions to create roles and attach policies to roles\. 
+ Created a Kubernetes cluster on which to run the operators\. It should either be Kubernetes version 1\.13 or 1\.14\. For automated cluster creation using `eksctl`, see [Getting Started with eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)\. It takes 20 to 30 minutes to provision a cluster\. 

#### Cluster\-scoped deployment<a name="cluster-scoped-deployment"></a>

Before you can deploy your operator using an IAM role, associate an OpenID Connect \(OIDC\) provider with your role to authenticate with the IAM service\. 

##### Create an OpenID Connect Provider for Your Cluster<a name="create-an-openid-connect-provider-for-your-cluster"></a>

The following instructions show how to create and associate an OIDC provider with your Amazon EKS cluster\. 

1. Set the local `CLUSTER_NAME` and `AWS_REGION` environment variables as follows: 

   ```
   # Set the Region and cluster
     export CLUSTER_NAME="<your cluster name>"
     export AWS_REGION="<your region>"
   ```

1. Use the following command to associate the OIDC provider with your cluster\. For more information, see [Enabling IAM Roles for Service Accounts on your Cluster\.](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) 

   ```
   eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} \
         --region ${AWS_REGION} --approve
   ```

   Your output should look like the following: 

   ```
   [_]  eksctl version 0.10.1
     [_]  using region us-east-1
     [_]  IAM OpenID Connect provider is associated with cluster "my-cluster" in "us-east-1"
   ```

Now that the cluster has an OIDC identity provider, you can create a role and give a Kubernetes ServiceAccount permission to assume the role\. 

##### Get the OIDC ID<a name="get-the-oidc-id"></a>

To set up the ServiceAccount, obtain the OpenID Connect issuer URL using the following command: 

```
aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} \
      --query cluster.identity.oidc.issuer --output text
```

The command returns a URL like the following: 

```
https://oidc.eks.${AWS_REGION}.amazonaws.com/id/D48675832CA65BD10A532F597OIDCID
```

In this URL, the value `D48675832CA65BD10A532F597OIDCID` is the OIDC ID\. The OIDC ID for your cluster is different\. You need this OIDC ID value to create a role\. 

 If your output is `None`, it means that your client version is old\. To work around this, run the following command: 

```
aws eks describe-cluster --region ${AWS_REGION} --query cluster --name ${CLUSTER_NAME} --output text | grep OIDC
```

The OIDC URL is returned as follows: 

```
OIDC https://oidc.eks.us-east-1.amazonaws.com/id/D48675832CA65BD10A532F597OIDCID
```

##### Create an IAM Role<a name="create-an-iam-role"></a>

1. Create a file named `trust.json` and insert the following trust relationship code block into it\. Be sure to replace all `<OIDC ID>`, `<AWS account number>`, and `<EKS Cluster region>` placeholders with values corresponding to your cluster\. 

   ```
   {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Principal": {
             "Federated": "arn:aws:iam::<AWS account number>:oidc-provider/oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>"
           },
           "Action": "sts:AssumeRoleWithWebIdentity",
           "Condition": {
             "StringEquals": {
               "oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>:aud": "sts.amazonaws.com",
               "oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>:sub": "system:serviceaccount:sagemaker-k8s-operator-system:sagemaker-k8s-operator-default"
             }
           }
         }
       ]
     }
   ```

1. Run the following command to create a role with the trust relationship defined in `trust.json`\. This role enables the Amazon EKS cluster to get and refresh credentials from IAM\. 

   ```
   aws iam create-role --region ${AWS_REGION} --role-name <role name> --assume-role-policy-document file://trust.json --output=text
   ```

   Your output should look like the following: 

   ```
   ROLE    arn:aws:iam::123456789012:role/my-role 2019-11-22T21:46:10Z    /       ABCDEFSFODNN7EXAMPLE   my-role
     ASSUMEROLEPOLICYDOCUMENT        2012-10-17
     STATEMENT       sts:AssumeRoleWithWebIdentity   Allow
     STRINGEQUALS    sts.amazonaws.com       system:serviceaccount:sagemaker-k8s-operator-system:sagemaker-k8s-operator-default
     PRINCIPAL       arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/
   ```

    Take note of `ROLE ARN`; you pass this value to your operator\. 

##### Attach the AmazonSageMakerFullAccess Policy to the Role<a name="attach-the-amazonsagemakerfullaccess-policy-to-the-role"></a>

To give the role access to SageMaker, attach the [AmazonSageMakerFullAccess](https://console.aws.amazon.com/iam/home?#/policies/arn:aws:iam::aws:policy/AmazonSageMakerFullAccess) policy\. If you want to limit permissions to the operator, you can create your own custom policy and attach it\. 

 To attach AmazonSageMakerFullAccess, run the following command: 

```
aws iam attach-role-policy --role-name <role name>  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
```

The Kubernetes ServiceAccount `sagemaker-k8s-operator-default` should have `AmazonSageMakerFullAccess` permissions\. Confirm this when you install the operator\. 

##### Deploy the Operator<a name="deploy-the-operator"></a>

When deploying your operator, you can use either a YAML file or Helm charts\. 

##### Deploy the Operator Using YAML<a name="deploy-the-operator-using-yaml"></a>

This is the simplest way to deploy your operators\. The process is as follows: 

1. Download the installer script using the following command: 

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/release/rolebased/installer.yaml
   ```

1. Edit the `installer.yaml` file to replace `eks.amazonaws.com/role-arn`\. Replace the ARN here with the Amazon Resource Name \(ARN\) for the OIDC\-based role you’ve created\. 

1. Use the following command to deploy the cluster: 

   ```
   kubectl apply -f installer.yaml
   ```

##### Deploy the Operator Using Helm Charts<a name="deploy-the-operator-using-helm-charts"></a>

Use the provided Helm Chart to install the operator\. 

1. Clone the Helm installer directory using the following command: 

   ```
   git clone https://github.com/aws/amazon-sagemaker-operator-for-k8s.git
   ```

1. Navigate to the `amazon-sagemaker-operator-for-k8s/hack/charts/installer` folder\. Edit the `rolebased/values.yaml` file, which includes high\-level parameters for the chart\. Replace the role ARN here with the Amazon Resource Name \(ARN\) for the OIDC\-based role you’ve created\. 

1. Install the Helm Chart using the following command: 

   ```
   kubectl create namespace sagemaker-k8s-operator-system
     helm install --namespace sagemaker-k8s-operator-system sagemaker-operator rolebased/
   ```

   If you decide to install the operator into a namespace other than the one specified, you need to adjust the namespace defined in the IAM role `trust.json` file to match\. 

1. After a moment, the chart is installed with a randomly generated name\. Verify that the installation succeeded by running the following command: 

   ```
   helm ls
   ```

   Your output should look like the following: 

   ```
   NAME                    NAMESPACE                       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
     sagemaker-operator      sagemaker-k8s-operator-system   1               2019-11-20 23:14:59.6777082 +0000 UTC   deployed        sagemaker-k8s-operator-0.1.0
   ```

##### Verify the operator deployment<a name="verify-the-operator-deployment"></a>

1. You should be able to see the SageMaker Custom Resource Definitions \(CRDs\) for each operator deployed to your cluster by running the following command: 

   ```
   kubectl get crd | grep sagemaker
   ```

   Your output should look like the following: 

   ```
   batchtransformjobs.sagemaker.aws.amazon.com         2019-11-20T17:12:34Z
     endpointconfigs.sagemaker.aws.amazon.com            2019-11-20T17:12:34Z
     hostingdeployments.sagemaker.aws.amazon.com         2019-11-20T17:12:34Z
     hyperparametertuningjobs.sagemaker.aws.amazon.com   2019-11-20T17:12:34Z
     models.sagemaker.aws.amazon.com                     2019-11-20T17:12:34Z
     trainingjobs.sagemaker.aws.amazon.com               2019-11-20T17:12:34Z
   ```

1. Ensure that the operator pod is running successfully\. Use the following command to list all pods: 

   ```
   kubectl -n sagemaker-k8s-operator-system get pods
   ```

   You should see a pod named `sagemaker-k8s-operator-controller-manager-*****` in the namespace `sagemaker-k8s-operator-system` as follows: 

   ```
   NAME                                                         READY   STATUS    RESTARTS   AGE
     sagemaker-k8s-operator-controller-manager-12345678-r8abc     2/2     Running   0          23s
   ```

#### Namespace\-scoped deployment<a name="namespace-scoped-deployment"></a>

You have the option to install your operator within the scope of an individual Kubernetes namespace\. In this mode, the controller only monitors and reconciles resources with SageMaker if the resources are created within that namespace\. This allows for finer\-grained control over which controller is managing which resources\. This is useful for deploying to multiple AWS accounts or controlling which users have access to particular jobs\. 

This guide outlines how to install an operator into a particular, predefined namespace\. To deploy a controller into a second namespace, follow the guide from beginning to end and change out the namespace in each step\. 

##### Create an OpenID Connect Provider for Your Amazon EKS cluster<a name="create-an-openid-connect-provider-for-your-eks-cluster"></a>

The following instructions show how to create and associate an OIDC provider with your Amazon EKS cluster\. 

1. Set the local `CLUSTER_NAME` and `AWS_REGION` environment variables as follows: 

   ```
   # Set the region and cluster
     export CLUSTER_NAME="<your cluster name>"
     export AWS_REGION="<your region>"
   ```

1. Use the following command to associate the OIDC provider with your cluster\. For more information, see [Enabling IAM Roles for Service Accounts on your Cluster\.](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) 

   ```
   eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} \
         --region ${AWS_REGION} --approve
   ```

   Your output should look like the following: 

   ```
   [_]  eksctl version 0.10.1
     [_]  using region us-east-1
     [_]  IAM OpenID Connect provider is associated with cluster "my-cluster" in "us-east-1"
   ```

 Now that the cluster has an OIDC identity provider, create a role and give a Kubernetes ServiceAccount permission to assume the role\. 

##### Get your OIDC ID<a name="get-your-oidc-id"></a>

To set up the ServiceAccount, first obtain the OpenID Connect issuer URL using the following command: 

```
aws eks describe-cluster --name ${CLUSTER_NAME} --region ${AWS_REGION} \
      --query cluster.identity.oidc.issuer --output text
```

The command returns a URL like the following: 

```
https://oidc.eks.${AWS_REGION}.amazonaws.com/id/D48675832CA65BD10A532F597OIDCID
```

In this URL, the value D48675832CA65BD10A532F597OIDCID is the OIDC ID\. The OIDC ID for your cluster will be different\. You need this OIDC ID value to create a role\. 

 If your output is `None`, it means that your client version is old\. To work around this, run the following command: 

```
aws eks describe-cluster --region ${AWS_REGION} --query cluster --name ${CLUSTER_NAME} --output text | grep OIDC
```

The OIDC URL is returned as follows: 

```
OIDC https://oidc.eks.us-east-1.amazonaws.com/id/D48675832CA65BD10A532F597OIDCID
```

##### Create your IAM Role<a name="create-your-iam-role"></a>

1. Create a file named `trust.json` and insert the following trust relationship code block into it\. Be sure to replace all `<OIDC ID>`, `<AWS account number>`, `<EKS Cluster region>`, and `<Namespace>` placeholders with values corresponding to your cluster\. For the purposes of this guide, `my-namespace` is used for the `<Namespace>` value\. 

   ```
   {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Principal": {
             "Federated": "arn:aws:iam::<AWS account number>:oidc-provider/oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>"
           },
           "Action": "sts:AssumeRoleWithWebIdentity",
           "Condition": {
             "StringEquals": {
               "oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>:aud": "sts.amazonaws.com",
               "oidc.eks.<EKS Cluster region>.amazonaws.com/id/<OIDC ID>:sub": "system:serviceaccount:<Namespace>:sagemaker-k8s-operator-default"
             }
           }
         }
       ]
     }
   ```

1. Run the following command to create a role with the trust relationship defined in `trust.json`\. This role enables the Amazon EKS cluster to get and refresh credentials from IAM\. 

   ```
   aws iam create-role --region ${AWS_REGION} --role-name <role name> --assume-role-policy-document file://trust.json --output=text
   ```

   Your output should look like the following: 

   ```
   ROLE    arn:aws:iam::123456789012:role/my-role 2019-11-22T21:46:10Z    /       ABCDEFSFODNN7EXAMPLE   my-role
     ASSUMEROLEPOLICYDOCUMENT        2012-10-17
     STATEMENT       sts:AssumeRoleWithWebIdentity   Allow
     STRINGEQUALS    sts.amazonaws.com       system:serviceaccount:my-namespace:sagemaker-k8s-operator-default
     PRINCIPAL       arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/
   ```

Take note of `ROLE ARN`\. You pass this value to your operator\. 

##### Attach the AmazonSageMakerFullAccess Policy to your Role<a name="attach-the-amazonsagemakerfullaccess-policy-to-your-role"></a>

To give the role access to SageMaker, attach the [AmazonSageMakerFullAccess](https://console.aws.amazon.com/iam/home?#/policies/arn:aws:iam::aws:policy/AmazonSageMakerFullAccess) policy\. If you want to limit permissions to the operator, you can create your own custom policy and attach it\. 

 To attach AmazonSageMakerFullAccess, run the following command: 

```
aws iam attach-role-policy --role-name <role name>  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess
```

The Kubernetes ServiceAccount `sagemaker-k8s-operator-default` should have `AmazonSageMakerFullAccess` permissions\. Confirm this when you install the operator\. 

##### Deploy the Operator to Your Namespace<a name="deploy-the-operator-to-your-namespace"></a>

When deploying your operator, you can use either a YAML file or Helm charts\. 

##### Deploy the Operator to Your Namespace Using YAML<a name="deploy-the-operator-to-your-namespace-using-yaml"></a>

There are two parts to deploying an operator within the scope of a namespace\. The first is the set of CRDs that are installed at a cluster level\. These resource definitions only need to be installed once per Kubernetes cluster\. The second part is the operator permissions and deployment itself\. 

 If you have not already installed the CRDs into the cluster, apply the CRD installer YAML using the following command: 

```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/release/rolebased/namespaced/crd.yaml
```

To install the operator onto the cluster: 

1. Download the operator installer YAML using the following command: 

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/release/rolebased/namespaced/operator.yaml
   ```

1. Update the installer YAML to place the resources into your specified namespace using the following command: 

   ```
   sed -i -e 's/PLACEHOLDER-NAMESPACE/<YOUR NAMESPACE>/g' operator.yaml
   ```

1. Edit the `operator.yaml` file to place resources into your `eks.amazonaws.com/role-arn`\. Replace the ARN here with the Amazon Resource Name \(ARN\) for the OIDC\-based role you’ve created\. 

1. Use the following command to deploy the cluster: 

   ```
   kubectl apply -f operator.yaml
   ```

##### Deploy the Operator to Your Namespace Using Helm Charts<a name="deploy-the-operator-to-your-namespace-using-helm-charts"></a>

There are two parts needed to deploy an operator within the scope of a namespace\. The first is the set of CRDs that are installed at a cluster level\. These resource definitions only need to be installed once per Kubernetes cluster\. The second part is the operator permissions and deployment itself\. When using helm charts you have to first create the namespace using `kubectl`\. 

1. Clone the Helm installer directory using the following command: 

   ```
   git clone https://github.com/aws/amazon-sagemaker-operator-for-k8s.git
   ```

1. Navigate to the `amazon-sagemaker-operator-for-k8s/hack/charts/installer/namespaced` folder\. Edit the `rolebased/values.yaml` file, which includes high\-level parameters for the chart\. Replace the role ARN here with the Amazon Resource Name \(ARN\) for the OIDC\-based role you’ve created\. 

1. Install the Helm Chart using the following command: 

   ```
   helm install crds crd_chart/
   ```

1. Create the required namespace and install the operator using the following command: 

   ```
   kubectl create namespace <namespace>
     helm install --n <namespace> op operator_chart/
   ```

1. After a moment, the chart is installed with the name `sagemaker-operator`\. Verify that the installation succeeded by running the following command: 

   ```
   helm ls
   ```

   Your output should look like the following: 

   ```
   NAME                    NAMESPACE                       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
     sagemaker-operator      my-namespace                    1               2019-11-20 23:14:59.6777082 +0000 UTC   deployed        sagemaker-k8s-operator-0.1.0
   ```

##### Verify the operator deployment to your namespace<a name="verify-the-operator-deployment-to-your-namespace"></a>

1. You should be able to see the SageMaker Custom Resource Definitions \(CRDs\) for each operator deployed to your cluster by running the following command: 

   ```
   kubectl get crd | grep sagemaker
   ```

   Your output should look like the following: 

   ```
   batchtransformjobs.sagemaker.aws.amazon.com         2019-11-20T17:12:34Z
     endpointconfigs.sagemaker.aws.amazon.com            2019-11-20T17:12:34Z
     hostingdeployments.sagemaker.aws.amazon.com         2019-11-20T17:12:34Z
     hyperparametertuningjobs.sagemaker.aws.amazon.com   2019-11-20T17:12:34Z
     models.sagemaker.aws.amazon.com                     2019-11-20T17:12:34Z
     trainingjobs.sagemaker.aws.amazon.com               2019-11-20T17:12:34Z
   ```

1. Ensure that the operator pod is running successfully\. Use the following command to list all pods: 

   ```
   kubectl -n my-namespace get pods
   ```

   You should see a pod named `sagemaker-k8s-operator-controller-manager-*****` in the namespace `my-namespace` as follows: 

   ```
   NAME                                                         READY   STATUS    RESTARTS   AGE
     sagemaker-k8s-operator-controller-manager-12345678-r8abc     2/2     Running   0          23s
   ```

#### Install the SageMaker logs `kubectl` plugin<a name="install-the-amazon-sagemaker-logs-kubectl-plugin"></a>

 As part of the SageMaker Operators for Kubernetes, you can use the `smlogs` [plugin](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/) for `kubectl`\. This enables SageMaker CloudWatch logs to be streamed with `kubectl`\. `kubectl` must be installed onto your [PATH](http://www.linfo.org/path_env_var.html)\. The following commands place the binary in the `sagemaker-k8s-bin` directory in your home directory, and add that directory to your `PATH`\. 

```
export os="linux"
  
  wget https://amazon-sagemaker-operator-for-k8s-us-east-1.s3.amazonaws.com/kubectl-smlogs-plugin/v1/${os}.amd64.tar.gz
  tar xvzf ${os}.amd64.tar.gz
  
  # Move binaries to a directory in your homedir.
  mkdir ~/sagemaker-k8s-bin
  cp ./kubectl-smlogs.${os}.amd64/kubectl-smlogs ~/sagemaker-k8s-bin/.
  
  # This line adds the binaries to your PATH in your .bashrc.
  
  echo 'export PATH=$PATH:~/sagemaker-k8s-bin' >> ~/.bashrc
  
  # Source your .bashrc to update environment variables:
  source ~/.bashrc
```

Use the following command to verify that the `kubectl` plugin is installed correctly: 

```
kubectl smlogs
```

If the `kubectl` plugin is installed correctly, your output should look like the following: 

```
View SageMaker logs via Kubernetes
  
  Usage:
    smlogs [command]
  
  Aliases:
    smlogs, SMLogs, Smlogs
  
  Available Commands:
    BatchTransformJob       View BatchTransformJob logs via Kubernetes
    TrainingJob             View TrainingJob logs via Kubernetes
    help                    Help about any command
  
  Flags:
    -h, --help   help for smlogs
  
  Use "smlogs [command] --help" for more information about a command.
```

### Clean up resources<a name="cleanup-operator-resources"></a>

To uninstall the operator from your cluster, you must first make sure to delete all SageMaker resources from the cluster\. Failure to do so causes the operator delete operation to hang\. Run the following commands to stop all jobs: 

```
# Delete all SageMaker jobs from Kubernetes
  kubectl delete --all --all-namespaces hyperparametertuningjob.sagemaker.aws.amazon.com
  kubectl delete --all --all-namespaces trainingjobs.sagemaker.aws.amazon.com
  kubectl delete --all --all-namespaces batchtransformjob.sagemaker.aws.amazon.com
  kubectl delete --all --all-namespaces hostingdeployment.sagemaker.aws.amazon.com
```

You should see output similar to the following: 

```
$ kubectl delete --all --all-namespaces trainingjobs.sagemaker.aws.amazon.com
  trainingjobs.sagemaker.aws.amazon.com "xgboost-mnist-from-for-s3" deleted
  
  $ kubectl delete --all --all-namespaces hyperparametertuningjob.sagemaker.aws.amazon.com
  hyperparametertuningjob.sagemaker.aws.amazon.com "xgboost-mnist-hpo" deleted
  
  $ kubectl delete --all --all-namespaces batchtransformjob.sagemaker.aws.amazon.com
  batchtransformjob.sagemaker.aws.amazon.com "xgboost-mnist" deleted
  
  $ kubectl delete --all --all-namespaces hostingdeployment.sagemaker.aws.amazon.com
  hostingdeployment.sagemaker.aws.amazon.com "host-xgboost" deleted
```

After you delete all SageMaker jobs, see [Delete operators](#delete-operators) to delete the operator from your cluster\.

### Delete operators<a name="delete-operators"></a>

#### Delete cluster\-based operators<a name="delete-cluster-based-operators"></a>

##### Operators installed using YAML<a name="operators-installed-using-yaml"></a>

To uninstall the operator from your cluster, make sure that all SageMaker resources have been deleted from the cluster\. Failure to do so causes the operator delete operation to hang\.

**Note**  
Before deleting your cluster, be sure to delete all SageMaker resources from the cluster\. See [Clean up resources](#cleanup-operator-resources) for more information\.

After you delete all SageMaker jobs, use `kubectl` to delete the operator from the cluster:

```
# Delete the operator and its resources
  kubectl delete -f /installer.yaml
```

You should see output similar to the following: 

```
$ kubectl delete -f raw-yaml/installer.yaml
  namespace "sagemaker-k8s-operator-system" deleted
  customresourcedefinition.apiextensions.k8s.io "batchtransformjobs.sagemaker.aws.amazon.com" deleted
  customresourcedefinition.apiextensions.k8s.io "endpointconfigs.sagemaker.aws.amazon.com" deleted
  customresourcedefinition.apiextensions.k8s.io "hostingdeployments.sagemaker.aws.amazon.com" deleted
  customresourcedefinition.apiextensions.k8s.io "hyperparametertuningjobs.sagemaker.aws.amazon.com" deleted
  customresourcedefinition.apiextensions.k8s.io "models.sagemaker.aws.amazon.com" deleted
  customresourcedefinition.apiextensions.k8s.io "trainingjobs.sagemaker.aws.amazon.com" deleted
  role.rbac.authorization.k8s.io "sagemaker-k8s-operator-leader-election-role" deleted
  clusterrole.rbac.authorization.k8s.io "sagemaker-k8s-operator-manager-role" deleted
  clusterrole.rbac.authorization.k8s.io "sagemaker-k8s-operator-proxy-role" deleted
  rolebinding.rbac.authorization.k8s.io "sagemaker-k8s-operator-leader-election-rolebinding" deleted
  clusterrolebinding.rbac.authorization.k8s.io "sagemaker-k8s-operator-manager-rolebinding" deleted
  clusterrolebinding.rbac.authorization.k8s.io "sagemaker-k8s-operator-proxy-rolebinding" deleted
  service "sagemaker-k8s-operator-controller-manager-metrics-service" deleted
  deployment.apps "sagemaker-k8s-operator-controller-manager" deleted
  secrets "sagemaker-k8s-operator-abcde" deleted
```

##### Operators installed using Helm Charts<a name="operators-installed-using-helm-charts"></a>

To delete the operator CRDs, first delete all the running jobs\. Then delete the Helm Chart that was used to deploy the operators using the following commands: 

```
# get the helm charts
  helm ls
  
  # delete the charts
  helm delete <chart_name>
```

#### Delete namespace\-based operators<a name="delete-namespace-based-operators"></a>

##### Operators installed with YAML<a name="operators-installed-with-yaml"></a>

To uninstall the operator from your cluster, first make sure that all SageMaker resources have been deleted from the cluster\. Failure to do so causes the operator delete operation to hang\.

**Note**  
Before deleting your cluster, be sure to delete all SageMaker resources from the cluster\. See [Clean up resources](#cleanup-operator-resources) for more information\.

After you delete all SageMaker jobs, use `kubectl` to first delete the operator from the namespace and then the CRDs from the cluster\. Run the following commands to delete the operator from the cluster: 

```
# Delete the operator using the same yaml file that was used to install the operator
  kubectl delete -f operator.yaml
  
  # Now delete the CRDs using the CRD installer yaml
  kubectl delete -f https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/release/rolebased/namespaced/crd.yaml
  
  # Now you can delete the namespace if you want
  kubectl delete namespace <namespace>
```

##### Operators installed with Helm Charts<a name="operators-installed-with-helm-charts"></a>

To delete the operator CRDs, first delete all the running jobs\. Then delete the Helm Chart that was used to deploy the operators using the following commands: 

```
# Delete the operator
  helm delete <chart_name>
  
  # delete the crds
  helm delete crds
  
  # optionally delete the namespace
  kubectl delete namespace <namespace>
```

### Troubleshooting<a name="troubleshooting"></a>

#### Debugging a Failed Job<a name="debugging-a-failed-job"></a>
+ Check the job status by running the following: 

  ```
  kubectl get <CRD Type> <job name>
  ```
+ If the job was created in SageMaker, you can use the following command to see the `STATUS` and the `SageMaker Job Name`: 

  ```
  kubectl get <crd type> <job name>
  ```
+ You can use `smlogs` to find the cause of the issue using the following command: 

  ```
  kubectl smlogs <crd type> <job name>
  ```
+  You can also use the `describe` command to get more details about the job using the following command\. The output has an `additional` field that has more information about the status of the job\. 

  ```
  kubectl describe <crd type> <job name>
  ```
+ If the job was not created in SageMaker, then use the logs of the operator’s pod to find the cause of the issue as follows: 

  ```
  $ kubectl get pods -A | grep sagemaker
    # Output:
    sagemaker-k8s-operator-system   sagemaker-k8s-operator-controller-manager-5cd7df4d74-wh22z   2/2     Running   0          3h33m
    
    $ kubectl logs -p <pod name> -c manager -n sagemaker-k8s-operator-system
  ```

#### Deleting an Operator CRD<a name="deleting-an-operator-crd"></a>

If deleting a job is not working, check if the operator is running\. If the operator is not running, then you have to delete the finalizer using the following steps: 

1. In a new terminal, open the job in an editor using `kubectl edit` as follows: 

   ```
   kubectl edit <crd type> <job name>
   ```

1. Edit the job to delete the finalizer by removing the following two lines from the file\. Save the file and the job is be deleted\. 

   ```
   finalizers:
     - sagemaker-operator-finalizer
   ```

### Images and SMlogs in each Region<a name="images-and-smlogs-in-each-region"></a>

The following table lists the available operator images and SMLogs in each region\. 


|  Region  |  Controller Image  |  Linux SMLogs  | 
| --- | --- | --- | 
|  us\-east\-1  |  957583890962\.dkr\.ecr\.us\-east\-1\.amazonaws\.com/amazon\-sagemaker\-operator\-for\-k8s:v1  |  [https://s3\.us\-east\-1\.amazonaws\.com/amazon\-sagemaker\-operator\-for\-k8s\-us\-east\-1/kubectl\-smlogs\-plugin/v1/linux\.amd64\.tar\.gz](https://s3.us-east-1.amazonaws.com/amazon-sagemaker-operator-for-k8s-us-east-1/kubectl-smlogs-plugin/v1/linux.amd64.tar.gz)  | 
|  us\-east\-2  |  922499468684\.dkr\.ecr\.us\-east\-2\.amazonaws\.com/amazon\-sagemaker\-operator\-for\-k8s:v1  |  [https://s3\.us\-east\-2\.amazonaws\.com/amazon\-sagemaker\-operator\-for\-k8s\-us\-east\-2/kubectl\-smlogs\-plugin/v1/linux\.amd64\.tar\.gz](https://s3.us-east-2.amazonaws.com/amazon-sagemaker-operator-for-k8s-us-east-2/kubectl-smlogs-plugin/v1/linux.amd64.tar.gz)  | 
|  us\-west\-2  |  640106867763\.dkr\.ecr\.us\-west\-2\.amazonaws\.com/amazon\-sagemaker\-operator\-for\-k8s:v1  |  [https://s3\.us\-west\-2\.amazonaws\.com/amazon\-sagemaker\-operator\-for\-k8s\-us\-west\-2/kubectl\-smlogs\-plugin/v1/linux\.amd64\.tar\.gz](https://s3.us-west-2.amazonaws.com/amazon-sagemaker-operator-for-k8s-us-west-2/kubectl-smlogs-plugin/v1/linux.amd64.tar.gz)  | 
|  eu\-west\-1  |  613661167059\.dkr\.ecr\.eu\-west\-1\.amazonaws\.com/amazon\-sagemaker\-operator\-for\-k8s:v1  |  [https://s3\.eu\-west\-1\.amazonaws\.com/amazon\-sagemaker\-operator\-for\-k8s\-eu\-west\-1/kubectl\-smlogs\-plugin/v1/linux\.amd64\.tar\.gz](https://s3.eu-west-1.amazonaws.com/amazon-sagemaker-operator-for-k8s-eu-west-1/kubectl-smlogs-plugin/v1/linux.amd64.tar.gz)  | 

## Using Amazon SageMaker Jobs<a name="kubernetes-sagemaker-jobs"></a>

This section is based on the original version of [SageMaker Operators for Kubernetes](https://github.com/aws/amazon-sagemaker-operator-for-k8s)\.

**Important**  
We are stopping the development and technical support of [SageMaker Operators for Kubernetes](https://github.com/aws/amazon-sagemaker-operator-for-k8s/tree/master) in its original version\. If you are currently using version `v1.2.2` or below of the original version of SageMaker Operators for Kubernetes, we recommend migrating your resources to the latest SageMaker Operators for Kubernetes, the [ACK service controller for Amazon SageMaker](https://github.com/aws-controllers-k8s/sagemaker-controller) based on [AWS Controllers for Kubernetes \(ACK\)](https://aws-controllers-k8s.github.io/community/ )\.  
For information about the migration steps, see [Migrate resources to the latest Operators](kubernetes-sagemaker-operators-migrate.md)\. For answers to frequently asked questions regarding the end of support of the original version of SageMaker Operators for Kubernetes, see [Announcing the End of Support of the Original Version of SageMaker Operator for Kubernetes](kubernetes-sagemaker-operators-eos-announcement.md)

To run an Amazon SageMaker job using the Operators for Kubernetes, you can either apply a YAML file or use the supplied Helm Charts\. 

All sample operator jobs in the following tutorials use sample data taken from a public MNIST dataset\. In order to run these samples, download the dataset into your Amazon S3 bucket\. You can find the dataset in [Download the MNIST Dataset\.](https://docs.aws.amazon.com/sagemaker/latest/dg/ex1-preprocess-data-pull-data.html) 

**Topics**
+ [TrainingJob operator](#trainingjob-operator)
+ [HyperParameterTuningJob operator](#hyperparametertuningjobs-operator)
+ [BatchTransformJob operator](#batchtransformjobs-operator)
+ [HostingDeployment operator](#hosting-deployment-operator)
+ [ProcessingJob operator](#kubernetes-processing-job-operator)
+ [HostingAutoscalingPolicy \(HAP\) Operator](#kubernetes-hap-operator)

### TrainingJob operator<a name="trainingjob-operator"></a>

Training job operators reconcile your specified training job spec to SageMaker by launching it for you in SageMaker\. You can learn more about SageMaker training jobs in the SageMaker [CreateTrainingJob API documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/API_CreateTrainingJob.html)\. 

**Topics**
+ [Create a TrainingJob Using a YAML File](#create-a-trainingjob-using-a-simple-yaml-file)
+ [Create a TrainingJob Using a Helm Chart](#create-a-trainingjob-using-a-helm-chart)
+ [List TrainingJobs](#list-training-jobs)
+ [Describe a TrainingJob](#describe-a-training-job)
+ [View Logs from TrainingJobs](#view-logs-from-training-jobs)
+ [Delete TrainingJobs](#delete-training-jobs)

#### Create a TrainingJob Using a YAML File<a name="create-a-trainingjob-using-a-simple-yaml-file"></a>

1. Download the sample YAML file for training using the following command: 

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/xgboost-mnist-trainingjob.yaml
   ```

1. Edit the `xgboost-mnist-trainingjob.yaml` file to replace the `roleArn` parameter with your `<sagemaker-execution-role>`, and `outputPath` with your Amazon S3 bucket that the SageMaker execution role has write access to\. The `roleArn` must have permissions so that SageMaker can access Amazon S3, Amazon CloudWatch, and other services on your behalf\. For more information on creating an SageMaker ExecutionRole, see [SageMaker Roles](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html#sagemaker-roles-createtrainingjob-perms)\. Apply the YAML file using the following command: 

   ```
   kubectl apply -f xgboost-mnist-trainingjob.yaml
   ```

#### Create a TrainingJob Using a Helm Chart<a name="create-a-trainingjob-using-a-helm-chart"></a>

You can use Helm Charts to run TrainingJobs\. 

1. Clone the GitHub repository to get the source using the following command: 

   ```
   git clone https://github.com/aws/amazon-sagemaker-operator-for-k8s.git
   ```

1. Navigate to the `amazon-sagemaker-operator-for-k8s/hack/charts/training-jobs/` folder and edit the `values.yaml` file to replace values like `rolearn` and `outputpath` with values that correspond to your account\. The RoleARN must have permissions so that SageMaker can access Amazon S3, Amazon CloudWatch, and other services on your behalf\. For more information on creating an SageMaker ExecutionRole, see [SageMaker Roles](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html#sagemaker-roles-createtrainingjob-perms)\. 

##### Create the TrainingJob<a name="create-the-training-job"></a>

With the roles and Amazon S3 buckets replaced with appropriate values in `values.yaml`, you can create a training job using the following command: 

```
helm install . --generate-name
```

Your output should look like the following: 

```
NAME: chart-12345678
LAST DEPLOYED: Wed Nov 20 23:35:49 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thanks for installing the sagemaker-k8s-trainingjob.
```

##### Verify Your Training Helm Chart<a name="verify-your-training-helm-chart"></a>

To verify that the Helm Chart was created successfully, run: 

```
helm ls
```

Your output should look like the following: 

```
NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
chart-12345678        default         1               2019-11-20 23:35:49.9136092 +0000 UTC   deployed        sagemaker-k8s-trainingjob-0.1.0
rolebased-12345678    default         1               2019-11-20 23:14:59.6777082 +0000 UTC   deployed        sagemaker-k8s-operator-0.1.0
```

`helm install` creates a `TrainingJob` Kubernetes resource\. The operator launches the actual training job in SageMaker and updates the `TrainingJob` Kubernetes resource to reflect the status of the job in SageMaker\. You incur charges for SageMaker resources used during the duration of your job\. You do not incur any charges once your job completes or stops\. 

**Note**: SageMaker does not allow you to update a running training job\. You cannot edit any parameter and re\-apply the file/config\. Either change the metadata name or delete the existing job and create a new one\. Similar to existing training job operators like TFJob in Kubeflow, `update` is not supported\. 

#### List TrainingJobs<a name="list-training-jobs"></a>

Use the following command to list all jobs created using the Kubernetes operator: 

```
kubectl get TrainingJob
```

The output listing all jobs should look like the following: 

```
kubectl get trainingjobs
NAME                        STATUS       SECONDARY-STATUS   CREATION-TIME          SAGEMAKER-JOB-NAME
xgboost-mnist-from-for-s3   InProgress   Starting           2019-11-20T23:42:35Z   xgboost-mnist-from-for-s3-examplef11eab94e0ed4671d5a8f
```

A training job continues to be listed after the job has completed or failed\. You can remove a `TrainingJob` job from the list by following the Delete a Training Job steps\. Jobs that have completed or stopped do not incur any charges for SageMaker resources\. 

##### TrainingJob Status Values<a name="training-job-status-values"></a>

The `STATUS` field can be one of the following values: 
+ `Completed` 
+ `InProgress` 
+ `Failed` 
+ `Stopped` 
+ `Stopping` 

These statuses come directly from the SageMaker official [API documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/API_DescribeTrainingJob.html#SageMaker-DescribeTrainingJob-response-TrainingJobStatus)\. 

In addition to the official SageMaker status, it is possible for `STATUS` to be `SynchronizingK8sJobWithSageMaker`\. This means that the operator has not yet processed the job\. 

##### Secondary Status Values<a name="secondary-status-values"></a>

The secondary statuses come directly from the SageMaker official [API documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/API_DescribeTrainingJob.html#SageMaker-DescribeTrainingJob-response-SecondaryStatus)\. They contain more granular information about the status of the job\. 

#### Describe a TrainingJob<a name="describe-a-training-job"></a>

You can get more details about the training job by using the `describe` `kubectl` command\. This is typically used for debugging a problem or checking the parameters of a training job\. To get information about your training job, use the following command: 

```
kubectl describe trainingjob xgboost-mnist-from-for-s3
```

The output for your training job should look like the following: 

```
Name:         xgboost-mnist-from-for-s3
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  sagemaker.aws.amazon.com/v1
Kind:         TrainingJob
Metadata:
  Creation Timestamp:  2019-11-20T23:42:35Z
  Finalizers:
    sagemaker-operator-finalizer
  Generation:        2
  Resource Version:  23119
  Self Link:         /apis/sagemaker.aws.amazon.com/v1/namespaces/default/trainingjobs/xgboost-mnist-from-for-s3
  UID:               6d7uiui-0bef-11ea-b94e-0ed467example
Spec:
  Algorithm Specification:
    Training Image:       8256416981234.dkr.ecr.us-east-2.amazonaws.com/xgboost:1
    Training Input Mode:  File
  Hyper Parameters:
    Name:   eta
    Value:  0.2
    Name:   gamma
    Value:  4
    Name:   max_depth
    Value:  5
    Name:   min_child_weight
    Value:  6
    Name:   num_class
    Value:  10
    Name:   num_round
    Value:  10
    Name:   objective
    Value:  multi:softmax
    Name:   silent
    Value:  0
  Input Data Config:
    Channel Name:      train
    Compression Type:  None
    Content Type:      text/csv
    Data Source:
      S 3 Data Source:
        S 3 Data Distribution Type:  FullyReplicated
        S 3 Data Type:               S3Prefix
        S 3 Uri:                     https://s3-us-east-2.amazonaws.com/my-bucket/sagemaker/xgboost-mnist/train/
    Channel Name:                    validation
    Compression Type:                None
    Content Type:                    text/csv
    Data Source:
      S 3 Data Source:
        S 3 Data Distribution Type:  FullyReplicated
        S 3 Data Type:               S3Prefix
        S 3 Uri:                     https://s3-us-east-2.amazonaws.com/my-bucket/sagemaker/xgboost-mnist/validation/
  Output Data Config:
    S 3 Output Path:  s3://my-bucket/sagemaker/xgboost-mnist/xgboost/
  Region:             us-east-2
  Resource Config:
    Instance Count:     1
    Instance Type:      ml.m4.xlarge
    Volume Size In GB:  5
  Role Arn:             arn:aws:iam::12345678910:role/service-role/AmazonSageMaker-ExecutionRole
  Stopping Condition:
    Max Runtime In Seconds:  86400
  Training Job Name:         xgboost-mnist-from-for-s3-6d7fa0af0bef11eab94e0example
Status:
  Cloud Watch Log URL:           https://us-east-2.console.aws.amazon.com/cloudwatch/home?region=us-east-2#logStream:group=/aws/sagemaker/TrainingJobs;prefix=<example>;streamFilter=typeLogStreamPrefix
  Last Check Time:               2019-11-20T23:44:29Z
  Sage Maker Training Job Name:  xgboost-mnist-from-for-s3-6d7fa0af0bef11eab94eexample
  Secondary Status:              Downloading
  Training Job Status:           InProgress
Events:                          <none>
```

#### View Logs from TrainingJobs<a name="view-logs-from-training-jobs"></a>

Use the following command to see the logs from the `kmeans-mnist` training job: 

```
kubectl smlogs trainingjob xgboost-mnist-from-for-s3
```

Your output should look similar to the following\. The logs from instances are ordered chronologically\. 

```
"xgboost-mnist-from-for-s3" has SageMaker TrainingJobName "xgboost-mnist-from-for-s3-123456789" in region "us-east-2", status "InProgress" and secondary status "Starting"
xgboost-mnist-from-for-s3-6d7fa0af0bef11eab94e0ed46example/algo-1-1574293123 2019-11-20 23:45:24.7 +0000 UTC Arguments: train
xgboost-mnist-from-for-s3-6d7fa0af0bef11eab94e0ed46example/algo-1-1574293123 2019-11-20 23:45:24.7 +0000 UTC [2019-11-20:23:45:22:INFO] Running standalone xgboost training.
xgboost-mnist-from-for-s3-6d7fa0af0bef11eab94e0ed46example/algo-1-1574293123 2019-11-20 23:45:24.7 +0000 UTC [2019-11-20:23:45:22:INFO] File size need to be processed in the node: 1122.95mb. Available memory size in the node: 8586.0mb
xgboost-mnist-from-for-s3-6d7fa0af0bef11eab94e0ed46example/algo-1-1574293123 2019-11-20 23:45:24.7 +0000 UTC [2019-11-20:23:45:22:INFO] Determined delimiter of CSV input is ','
xgboost-mnist-from-for-s3-6d7fa0af0bef11eab94e0ed46example/algo-1-1574293123 2019-11-20 23:45:24.7 +0000 UTC [23:45:22] S3DistributionType set as FullyReplicated
```

#### Delete TrainingJobs<a name="delete-training-jobs"></a>

Use the following command to stop a training job on Amazon SageMaker: 

```
kubectl delete trainingjob xgboost-mnist-from-for-s3
```

This command removes the SageMaker training job from Kubernetes\. This command returns the following output: 

```
trainingjob.sagemaker.aws.amazon.com "xgboost-mnist-from-for-s3" deleted
```

If the job is still in progress on SageMaker, the job stops\. You do not incur any charges for SageMaker resources after your job stops or completes\. 

**Note**: SageMaker does not delete training jobs\. Stopped jobs continue to show on the SageMaker console\. The `delete` command takes about 2 minutes to clean up the resources from SageMaker\. 

### HyperParameterTuningJob operator<a name="hyperparametertuningjobs-operator"></a>

Hyperparameter tuning job operators reconcile your specified hyperparameter tuning job spec to SageMaker by launching it in SageMaker\. You can learn more about SageMaker hyperparameter tuning jobs in the SageMaker [CreateHyperParameterTuningJob API documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/API_CreateHyperParameterTuningJob.html)\. 

**Topics**
+ [Create a HyperparameterTuningJob Using a YAML File](#create-a-hyperparametertuningjob-using-a-simple-yaml-file)
+ [Create a HyperparameterTuningJob using a Helm Chart](#create-a-hyperparametertuningjob-using-a-helm-chart)
+ [List HyperparameterTuningJobs](#list-hyperparameter-tuning-jobs)
+ [Describe a HyperparameterTuningJob](#describe-a-hyperparameter-tuning-job)
+ [View Logs from HyperparameterTuningJobs](#view-logs-from-hyperparametertuning-jobs)
+ [Delete a HyperparameterTuningJob](#delete-hyperparametertuning-jobs)

#### Create a HyperparameterTuningJob Using a YAML File<a name="create-a-hyperparametertuningjob-using-a-simple-yaml-file"></a>

1. Download the sample YAML file for the hyperparameter tuning job using the following command: 

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/xgboost-mnist-hpo.yaml
   ```

1. Edit the `xgboost-mnist-hpo.yaml` file to replace the `roleArn` parameter with your `sagemaker-execution-role`\. For the hyperparameter tuning job to succeed, you must also change the `s3InputPath` and `s3OutputPath` to values that correspond to your account\. Apply the updates YAML file using the following command: 

   ```
   kubectl apply -f xgboost-mnist-hpo.yaml
   ```

#### Create a HyperparameterTuningJob using a Helm Chart<a name="create-a-hyperparametertuningjob-using-a-helm-chart"></a>

You can use Helm Charts to run hyperparameter tuning jobs\. 

1. Clone the GitHub repository to get the source using the following command: 

   ```
   git clone https://github.com/aws/amazon-sagemaker-operator-for-k8s.git
   ```

1. Navigate to the `amazon-sagemaker-operator-for-k8s/hack/charts/hyperparameter-tuning-jobs/` folder\. 

1. Edit the `values.yaml` file to replace the `roleArn` parameter with your `sagemaker-execution-role`\. For the hyperparameter tuning job to succeed, you must also change the `s3InputPath` and `s3OutputPath` to values that correspond to your account\. 

##### Create the HyperparameterTuningJob<a name="create-the-hpo-job"></a>

With the roles and Amazon S3 paths replaced with appropriate values in `values.yaml`, you can create a hyperparameter tuning job using the following command: 

```
helm install . --generate-name
```

Your output should look similar to the following: 

```
NAME: chart-1574292948
LAST DEPLOYED: Wed Nov 20 23:35:49 2019
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thanks for installing the sagemaker-k8s-hyperparametertuningjob.
```

##### Verify Chart Installation<a name="verify-chart-installation"></a>

To verify that the Helm Chart was created successfully, run the following command: 

```
helm ls
```

Your output should look like the following: 

```
NAME                    NAMESPACE       REVISION        UPDATED
chart-1474292948        default         1               2019-11-20 23:35:49.9136092 +0000 UTC   deployed        sagemaker-k8s-hyperparametertuningjob-0.1.0                               STATUS          CHART                           APP VERSION
chart-1574292948        default         1               2019-11-20 23:35:49.9136092 +0000 UTC   deployed        sagemaker-k8s-trainingjob-0.1.0
rolebased-1574291698    default         1               2019-11-20 23:14:59.6777082 +0000 UTC   deployed        sagemaker-k8s-operator-0.1.0
```

`helm install` creates a `HyperParameterTuningJob` Kubernetes resource\. The operator launches the actual hyperparameter optimization job in SageMaker and updates the `HyperParameterTuningJob` Kubernetes resource to reflect the status of the job in SageMaker\. You incur charges for SageMaker resources used during the duration of your job\. You do not incur any charges once your job completes or stops\. 

**Note**: SageMaker does not allow you to update a running hyperparameter tuning job\. You cannot edit any parameter and re\-apply the file/config\. You must either change the metadata name or delete the existing job and create a new one\. Similar to existing training job operators like TFJob in Kubeflow, `update` is not supported\. 

#### List HyperparameterTuningJobs<a name="list-hyperparameter-tuning-jobs"></a>

Use the following command to list all jobs created using the Kubernetes operator: 

```
kubectl get hyperparametertuningjob
```

Your output should look like the following: 

```
NAME         STATUS      CREATION-TIME          COMPLETED   INPROGRESS   ERRORS   STOPPED   BEST-TRAINING-JOB                               SAGEMAKER-JOB-NAME
xgboost-mnist-hpo   Completed   2019-10-17T01:15:52Z   10          0            0        0         xgboostha92f5e3cf07b11e9bf6c06d6-009-4c7a123   xgboostha92f5e3cf07b11e9bf6c123
```

A hyperparameter tuning job continues to be listed after the job has completed or failed\. You can remove a `hyperparametertuningjob` from the list by following the steps in Delete a Hyperparameter Tuning Job\. Jobs that have completed or stopped do not incur any charges for SageMaker resources\. 

##### Hyperparameter Tuning Job Status Values<a name="hyperparameter-tuning-job-status-values"></a>

The `STATUS` field can be one of the following values: 
+ `Completed` 
+ `InProgress` 
+ `Failed` 
+ `Stopped` 
+ `Stopping` 

These statuses come directly from the SageMaker official [API documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/API_DescribeHyperParameterTuningJob.html#SageMaker-DescribeHyperParameterTuningJob-response-HyperParameterTuningJobStatus)\. 

In addition to the official SageMaker status, it is possible for `STATUS` to be `SynchronizingK8sJobWithSageMaker`\. This means that the operator has not yet processed the job\. 

##### Status Counters<a name="status-counters"></a>

The output has several counters, like `COMPLETED` and `INPROGRESS`\. These represent how many training jobs have completed and are in progress, respectively\. For more information about how these are determined, see [TrainingJobStatusCounters](https://docs.aws.amazon.com/sagemaker/latest/dg/API_TrainingJobStatusCounters.html) in the SageMaker API documentation\. 

##### Best TrainingJob<a name="best-training-job"></a>

This column contains the name of the `TrainingJob` that best optimized the selected metric\. 

To see a summary of the tuned hyperparameters, run: 

```
kubectl describe hyperparametertuningjob xgboost-mnist-hpo
```

To see detailed information about the `TrainingJob`, run: 

```
kubectl describe trainingjobs <job name>
```

##### Spawned TrainingJobs<a name="spawned-training-jobs"></a>

You can also track all 10 training jobs in Kubernetes launched by `HyperparameterTuningJob` by running the following command: 

```
kubectl get trainingjobs
```

#### Describe a HyperparameterTuningJob<a name="describe-a-hyperparameter-tuning-job"></a>

You can obtain debugging details using the `describe` `kubectl` command\.

```
kubectl describe hyperparametertuningjob xgboost-mnist-hpo
```

In addition to information about the tuning job, the SageMaker Operator for Kubernetes also exposes the [best training job](https://docs.aws.amazon.com/sagemaker/latest/dg/automatic-model-tuning-monitor.html#automatic-model-tuning-best-training-job) found by the hyperparameter tuning job in the `describe` output as follows: 

```
Name:         xgboost-mnist-hpo
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"sagemaker.aws.amazon.com/v1","kind":"HyperparameterTuningJob","metadata":{"annotations":{},"name":"xgboost-mnist-hpo","namespace":...
API Version:  sagemaker.aws.amazon.com/v1
Kind:         HyperparameterTuningJob
Metadata:
  Creation Timestamp:  2019-10-17T01:15:52Z
  Finalizers:
    sagemaker-operator-finalizer
  Generation:        2
  Resource Version:  8167
  Self Link:         /apis/sagemaker.aws.amazon.com/v1/namespaces/default/hyperparametertuningjobs/xgboost-mnist-hpo
  UID:               a92f5e3c-f07b-11e9-bf6c-06d6f303uidu
Spec:
  Hyper Parameter Tuning Job Config:
    Hyper Parameter Tuning Job Objective:
      Metric Name:  validation:error
      Type:         Minimize
    Parameter Ranges:
      Integer Parameter Ranges:
        Max Value:     20
        Min Value:     10
        Name:          num_round
        Scaling Type:  Linear
    Resource Limits:
      Max Number Of Training Jobs:     10
      Max Parallel Training Jobs:      10
    Strategy:                          Bayesian
    Training Job Early Stopping Type:  Off
  Hyper Parameter Tuning Job Name:     xgboostha92f5e3cf07b11e9bf6c06d6
  Region:                              us-east-2
  Training Job Definition:
    Algorithm Specification:
      Training Image:       12345678910.dkr.ecr.us-east-2.amazonaws.com/xgboost:1
      Training Input Mode:  File
    Input Data Config:
      Channel Name:  train
      Content Type:  text/csv
      Data Source:
        s3DataSource:
          s3DataDistributionType:  FullyReplicated
          s3DataType:              S3Prefix
          s3Uri:                   https://s3-us-east-2.amazonaws.com/my-bucket/sagemaker/xgboost-mnist/train/
      Channel Name:                validation
      Content Type:                text/csv
      Data Source:
        s3DataSource:
          s3DataDistributionType:  FullyReplicated
          s3DataType:              S3Prefix
          s3Uri:                   https://s3-us-east-2.amazonaws.com/my-bucket/sagemaker/xgboost-mnist/validation/
    Output Data Config:
      s3OutputPath:  https://s3-us-east-2.amazonaws.com/my-bucket/sagemaker/xgboost-mnist/xgboost
    Resource Config:
      Instance Count:     1
      Instance Type:      ml.m4.xlarge
      Volume Size In GB:  5
    Role Arn:             arn:aws:iam::123456789012:role/service-role/AmazonSageMaker-ExecutionRole
    Static Hyper Parameters:
      Name:   base_score
      Value:  0.5
      Name:   booster
      Value:  gbtree
      Name:   csv_weights
      Value:  0
      Name:   dsplit
      Value:  row
      Name:   grow_policy
      Value:  depthwise
      Name:   lambda_bias
      Value:  0.0
      Name:   max_bin
      Value:  256
      Name:   max_leaves
      Value:  0
      Name:   normalize_type
      Value:  tree
      Name:   objective
      Value:  reg:linear
      Name:   one_drop
      Value:  0
      Name:   prob_buffer_row
      Value:  1.0
      Name:   process_type
      Value:  default
      Name:   rate_drop
      Value:  0.0
      Name:   refresh_leaf
      Value:  1
      Name:   sample_type
      Value:  uniform
      Name:   scale_pos_weight
      Value:  1.0
      Name:   silent
      Value:  0
      Name:   sketch_eps
      Value:  0.03
      Name:   skip_drop
      Value:  0.0
      Name:   tree_method
      Value:  auto
      Name:   tweedie_variance_power
      Value:  1.5
    Stopping Condition:
      Max Runtime In Seconds:  86400
Status:
  Best Training Job:
    Creation Time:  2019-10-17T01:16:14Z
    Final Hyper Parameter Tuning Job Objective Metric:
      Metric Name:        validation:error
      Value:
    Objective Status:     Succeeded
    Training End Time:    2019-10-17T01:20:24Z
    Training Job Arn:     arn:aws:sagemaker:us-east-2:123456789012:training-job/xgboostha92f5e3cf07b11e9bf6c06d6-009-4sample
    Training Job Name:    xgboostha92f5e3cf07b11e9bf6c06d6-009-4c7a3059
    Training Job Status:  Completed
    Training Start Time:  2019-10-17T01:18:35Z
    Tuned Hyper Parameters:
      Name:                                    num_round
      Value:                                   18
  Hyper Parameter Tuning Job Status:           Completed
  Last Check Time:                             2019-10-17T01:21:01Z
  Sage Maker Hyper Parameter Tuning Job Name:  xgboostha92f5e3cf07b11e9bf6c06d6
  Training Job Status Counters:
    Completed:            10
    In Progress:          0
    Non Retryable Error:  0
    Retryable Error:      0
    Stopped:              0
    Total Error:          0
Events:                   <none>
```

#### View Logs from HyperparameterTuningJobs<a name="view-logs-from-hyperparametertuning-jobs"></a>

Hyperparameter tuning jobs do not have logs, but all training jobs launched by them do have logs\. These logs can be accessed as if they were a normal training job\. For more information, see View Logs from Training Jobs\. 

#### Delete a HyperparameterTuningJob<a name="delete-hyperparametertuning-jobs"></a>

Use the following command to stop a hyperparameter job in SageMaker\. 

```
kubectl delete hyperparametertuningjob xgboost-mnist-hpo
```

This command removes the hyperparameter tuning job and associated training jobs from your Kubernetes cluster and stops them in SageMaker\. Jobs that have stopped or completed do not incur any charges for SageMaker resources\. SageMaker does not delete hyperparameter tuning jobs\. Stopped jobs continue to show on the SageMaker Console\. 

Your output should look like the following: 

```
hyperparametertuningjob.sagemaker.aws.amazon.com "xgboost-mnist-hpo" deleted
```

**Note**: The delete command takes about 2 minutes to clean up the resources from SageMaker\. 

### BatchTransformJob operator<a name="batchtransformjobs-operator"></a>

Batch transform job operators reconcile your specified batch transform job spec to SageMaker by launching it in SageMaker\. You can learn more about SageMaker batch transform job in the SageMaker [CreateTransformJob API documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/API_CreateTransformJob.html)\. 

**Topics**
+ [Create a BatchTransformJob Using a YAML File](#create-a-batchtransformjob-using-a-simple-yaml-file)
+ [Create a BatchTransformJob Using a Helm Chart](#create-a-batchtransformjob-using-a-helm-chart)
+ [List BatchTransformJobs](#list-batch-transform-jobs)
+ [Describe a BatchTransformJob](#describe-a-batch-transform-job)
+ [View Logs from BatchTransformJobs](#view-logs-from-batch-transform-jobs)
+ [Delete a BatchTransformJob](#delete-a-batch-transform-job)

#### Create a BatchTransformJob Using a YAML File<a name="create-a-batchtransformjob-using-a-simple-yaml-file"></a>

1. Download the sample YAML file for the batch transform job using the following command: 

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/xgboost-mnist-batchtransform.yaml
   ```

1. Edit the file `xgboost-mnist-batchtransform.yaml` to change necessary parameters to replace the `inputdataconfig` with your input data and `s3OutputPath` with your Amazon S3 buckets that the SageMaker execution role has write access to\. 

1. Apply the YAML file using the following command: 

   ```
   kubectl apply -f xgboost-mnist-batchtransform.yaml
   ```

#### Create a BatchTransformJob Using a Helm Chart<a name="create-a-batchtransformjob-using-a-helm-chart"></a>

You can use Helm Charts to run batch transform jobs\. 

##### Get the Helm installer directory<a name="get-the-helm-installer-directory"></a>

Clone the GitHub repository to get the source using the following command: 

```
git clone https://github.com/aws/amazon-sagemaker-operator-for-k8s.git
```

##### Configure the Helm Chart<a name="configure-the-helm-chart"></a>

Navigate to the `amazon-sagemaker-operator-for-k8s/hack/charts/batch-transform-jobs/` folder\. 

Edit the `values.yaml` file to replace the `inputdataconfig` with your input data and outputPath with your S3 buckets to which the SageMaker execution role has write access\. 

##### Create a BatchTransformJob<a name="create-a-batch-transform-job"></a>

1. Use the following command to create a batch transform job: 

   ```
   helm install . --generate-name
   ```

   Your output should look like the following: 

   ```
   NAME: chart-1574292948
   LAST DEPLOYED: Wed Nov 20 23:35:49 2019
   NAMESPACE: default
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   Thanks for installing the sagemaker-k8s-batch-transform-job.
   ```

1. To verify that the Helm Chart was created successfully, run the following command: 

   ```
   helm ls
   NAME                    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
   chart-1474292948        default         1               2019-11-20 23:35:49.9136092 +0000 UTC   deployed        sagemaker-k8s-batchtransformjob-0.1.0
   chart-1474292948        default         1               2019-11-20 23:35:49.9136092 +0000 UTC   deployed        sagemaker-k8s-hyperparametertuningjob-0.1.0
   chart-1574292948        default         1               2019-11-20 23:35:49.9136092 +0000 UTC   deployed        sagemaker-k8s-trainingjob-0.1.0
   rolebased-1574291698    default         1               2019-11-20 23:14:59.6777082 +0000 UTC   deployed        sagemaker-k8s-operator-0.1.0
   ```

   This command creates a `BatchTransformJob` Kubernetes resource\. The operator launches the actual transform job in SageMaker and updates the `BatchTransformJob` Kubernetes resource to reflect the status of the job in SageMaker\. You incur charges for SageMaker resources used during the duration of your job\. You do not incur any charges once your job completes or stops\. 

**Note**: SageMaker does not allow you to update a running batch transform job\. You cannot edit any parameter and re\-apply the file/config\. You must either change the metadata name or delete the existing job and create a new one\. Similar to existing training job operators like TFJob in Kubeflow, `update` is not supported\. 

#### List BatchTransformJobs<a name="list-batch-transform-jobs"></a>

Use the following command to list all jobs created using the Kubernetes operator: 

```
kubectl get batchtransformjob
```

Your output should look like the following: 

```
NAME                                STATUS      CREATION-TIME          SAGEMAKER-JOB-NAME
xgboost-mnist-batch-transform       Completed   2019-11-18T03:44:00Z   xgboost-mnist-a88fb19809b511eaac440aa8axgboost
```

A batch transform job will continue to be listed after the job has completed or failed\. You can remove a `hyperparametertuningjob` from the list by following the Delete a Batch Transform Job steps\. Jobs that have completed or stopped do not incur any charges for SageMaker resources\. 

##### Batch Transform Status Values<a name="batch-transform-status-values"></a>

The `STATUS` field can be one of the following values: 
+ `Completed` 
+ `InProgress` 
+ `Failed` 
+ `Stopped` 
+ `Stopping` 

These statuses come directly from the SageMaker official [API documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/API_DescribeHyperParameterTuningJob.html#SageMaker-DescribeHyperParameterTuningJob-response-HyperParameterTuningJobStatus)\. 

In addition to the official SageMaker status, it is possible for `STATUS` to be `SynchronizingK8sJobWithSageMaker`\. This means that the operator has not yet processed the job\.

#### Describe a BatchTransformJob<a name="describe-a-batch-transform-job"></a>

You can obtain debugging details using the `describe` `kubectl` command\.

```
kubectl describe batchtransformjob xgboost-mnist-batch-transform
```

Your output should look like the following: 

```
Name:         xgboost-mnist-batch-transform
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"sagemaker.aws.amazon.com/v1","kind":"BatchTransformJob","metadata":{"annotations":{},"name":"xgboost-mnist","namespace"...
API Version:  sagemaker.aws.amazon.com/v1
Kind:         BatchTransformJob
Metadata:
  Creation Timestamp:  2019-11-18T03:44:00Z
  Finalizers:
    sagemaker-operator-finalizer
  Generation:        2
  Resource Version:  21990924
  Self Link:         /apis/sagemaker.aws.amazon.com/v1/namespaces/default/batchtransformjobs/xgboost-mnist
  UID:               a88fb198-09b5-11ea-ac44-0aa8a9UIDNUM
Spec:
  Model Name:  TrainingJob-20190814SMJOb-IKEB
  Region:      us-east-1
  Transform Input:
    Content Type:  text/csv
    Data Source:
      S 3 Data Source:
        S 3 Data Type:  S3Prefix
        S 3 Uri:        s3://my-bucket/mnist_kmeans_example/input
  Transform Job Name:   xgboost-mnist-a88fb19809b511eaac440aa8a9SMJOB
  Transform Output:
    S 3 Output Path:  s3://my-bucket/mnist_kmeans_example/output
  Transform Resources:
    Instance Count:  1
    Instance Type:   ml.m4.xlarge
Status:
  Last Check Time:                2019-11-19T22:50:40Z
  Sage Maker Transform Job Name:  xgboost-mnist-a88fb19809b511eaac440aaSMJOB
  Transform Job Status:           Completed
Events:                           <none>
```

#### View Logs from BatchTransformJobs<a name="view-logs-from-batch-transform-jobs"></a>

Use the following command to see the logs from the `xgboost-mnist` batch transform job: 

```
kubectl smlogs batchtransformjob xgboost-mnist-batch-transform
```

#### Delete a BatchTransformJob<a name="delete-a-batch-transform-job"></a>

Use the following command to stop a batch transform job in SageMaker\. 

```
kubectl delete batchTransformJob xgboost-mnist-batch-transform
```

Your output should look like the following: 

```
batchtransformjob.sagemaker.aws.amazon.com "xgboost-mnist" deleted
```

This command removes the batch transform job from your Kubernetes cluster, as well as stops them in SageMaker\. Jobs that have stopped or completed do not incur any charges for SageMaker resources\. Delete takes about 2 minutes to clean up the resources from SageMaker\. 

**Note**: SageMaker does not delete batch transform jobs\. Stopped jobs continue to show on the SageMaker console\. 

### HostingDeployment operator<a name="hosting-deployment-operator"></a>

HostingDeployment operators support creating and deleting an endpoint, as well as updating an existing endpoint, for real\-time inference\. The hosting deployment operator reconciles your specified hosting deployment job spec to SageMaker by creating models, endpoint\-configs and endpoints in SageMaker\. You can learn more about SageMaker inference in the SageMaker [CreateEndpoint API documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/API_CreateEndpoint.html)\. 

**Topics**
+ [Configure a HostingDeployment Resource](#configure-a-hostingdeployment-resource)
+ [Create a HostingDeployment](#create-a-hostingdeployment)
+ [List HostingDeployments](#list-hostingdeployments)
+ [Describe a HostingDeployment](#describe-a-hostingdeployment)
+ [Invoking the Endpoint](#invoking-the-endpoint)
+ [Update HostingDeployment](#update-hostingdeployment)
+ [Delete the HostingDeployment](#delete-the-hostingdeployment)

#### Configure a HostingDeployment Resource<a name="configure-a-hostingdeployment-resource"></a>

Download the sample YAML file for the hosting deployment job using the following command: 

```
wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/xgboost-mnist-hostingdeployment.yaml
```

The `xgboost-mnist-hostingdeployment.yaml` file has the following components that can be edited as required: 
+ *ProductionVariants*\. A production variant is a set of instances serving a single model\. SageMaker load\-balances between all production variants according to set weights\. 
+ *Models*\. A model is the containers and execution role ARN necessary to serve a model\. It requires at least a single container\. 
+ *Containers*\. A container specifies the dataset and serving image\. If you are using your own custom algorithm instead of an algorithm provided by SageMaker, the inference code must meet SageMaker requirements\. For more information, see [Using Your Own Algorithms with SageMaker](https://docs.aws.amazon.com/sagemaker/latest/dg/your-algorithms.html)\. 

#### Create a HostingDeployment<a name="create-a-hostingdeployment"></a>

To create a HostingDeployment, use `kubectl` to apply the file `hosting.yaml` with the following command: 

```
kubectl apply -f hosting.yaml
```

SageMaker creates an endpoint with the specified configuration\. You incur charges for SageMaker resources used during the lifetime of your endpoint\. You do not incur any charges once your endpoint is deleted\. 

The creation process takes approximately 10 minutes\. 

#### List HostingDeployments<a name="list-hostingdeployments"></a>

To verify that the HostingDeployment was created, use the following command: 

```
kubectl get hostingdeployments
```

Your output should look like the following: 

```
NAME           STATUS     SAGEMAKER-ENDPOINT-NAME
host-xgboost   Creating   host-xgboost-def0e83e0d5f11eaaa450aSMLOGS
```

##### HostingDeployment Status Values<a name="hostingdeployment-status-values"></a>

The status field can be one of several values: 
+ `SynchronizingK8sJobWithSageMaker`: The operator is preparing to create the endpoint\. 
+ `ReconcilingEndpoint`: The operator is creating, updating, or deleting endpoint resources\. If the HostingDeployment remains in this state, use `kubectl describe` to see the reason in the `Additional` field\. 
+ `OutOfService`: Endpoint is not available to take incoming requests\. 
+ `Creating`: [CreateEndpoint](https://docs.aws.amazon.com/sagemaker/latest/dg/API_CreateEndpoint.html) is executing\. 
+ `Updating`: [UpdateEndpoint](https://docs.aws.amazon.com/sagemaker/latest/dg/API_UpdateEndpoint.html) or [UpdateEndpointWeightsAndCapacities](https://docs.aws.amazon.com/sagemaker/latest/dg/API_UpdateEndpointWeightsAndCapacities.html) is executing\. 
+ `SystemUpdating`: Endpoint is undergoing maintenance and cannot be updated or deleted or re\-scaled until it has completed\. This maintenance operation does not change any customer\-specified values such as VPC config, KMS encryption, model, instance type, or instance count\. 
+ `RollingBack`: Endpoint fails to scale up or down or change its variant weight and is in the process of rolling back to its previous configuration\. Once the rollback completes, endpoint returns to an `InService` status\. This transitional status only applies to an endpoint that has autoscaling enabled and is undergoing variant weight or capacity changes as part of an [UpdateEndpointWeightsAndCapacities](https://docs.aws.amazon.com/sagemaker/latest/dg/API_UpdateEndpointWeightsAndCapacities.html) call or when the [UpdateEndpointWeightsAndCapacities](https://docs.aws.amazon.com/sagemaker/latest/dg/API_UpdateEndpointWeightsAndCapacities.html) operation is called explicitly\. 
+ `InService`: Endpoint is available to process incoming requests\. 
+ `Deleting`: [DeleteEndpoint](https://docs.aws.amazon.com/sagemaker/latest/dg/API_DeleteEndpoint.html) is executing\. 
+ `Failed`: Endpoint could not be created, updated, or re\-scaled\. Use [DescribeEndpoint:FailureReason](https://docs.aws.amazon.com/sagemaker/latest/dg/API_DescribeEndpoint.html#SageMaker-DescribeEndpoint-response-FailureReason) for information about the failure\. [DeleteEndpoint](https://docs.aws.amazon.com/sagemaker/latest/dg/API_DeleteEndpoint.html) is the only operation that can be performed on a failed endpoint\. 

#### Describe a HostingDeployment<a name="describe-a-hostingdeployment"></a>

You can obtain debugging details using the `describe` `kubectl` command\.

```
kubectl describe hostingdeployment
```

Your output should look like the following: 

```
Name:         host-xgboost
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"sagemaker.aws.amazon.com/v1","kind":"HostingDeployment","metadata":{"annotations":{},"name":"host-xgboost","namespace":"def..."
API Version:  sagemaker.aws.amazon.com/v1
Kind:         HostingDeployment
Metadata:
  Creation Timestamp:  2019-11-22T19:40:00Z
  Finalizers:
    sagemaker-operator-finalizer
  Generation:        1
  Resource Version:  4258134
  Self Link:         /apis/sagemaker.aws.amazon.com/v1/namespaces/default/hostingdeployments/host-xgboost
  UID:               def0e83e-0d5f-11ea-aa45-0a3507uiduid
Spec:
  Containers:
    Container Hostname:  xgboost
    Image:               123456789012.dkr.ecr.us-east-2.amazonaws.com/xgboost:latest
    Model Data URL:      s3://my-bucket/inference/xgboost-mnist/model.tar.gz
  Models:
    Containers:
      xgboost
    Execution Role Arn:  arn:aws:iam::123456789012:role/service-role/AmazonSageMaker-ExecutionRole
    Name:                xgboost-model
    Primary Container:   xgboost
  Production Variants:
    Initial Instance Count:  1
    Instance Type:           ml.c5.large
    Model Name:              xgboost-model
    Variant Name:            all-traffic
  Region:                    us-east-2
Status:
  Creation Time:         2019-11-22T19:40:04Z
  Endpoint Arn:          arn:aws:sagemaker:us-east-2:123456789012:endpoint/host-xgboost-def0e83e0d5f11eaaaexample
  Endpoint Config Name:  host-xgboost-1-def0e83e0d5f11e-e08f6c510d5f11eaaa450aexample
  Endpoint Name:         host-xgboost-def0e83e0d5f11eaaa450a350733ba06
  Endpoint Status:       Creating
  Endpoint URL:          https://runtime.sagemaker.us-east-2.amazonaws.com/endpoints/host-xgboost-def0e83e0d5f11eaaaexample/invocations
  Last Check Time:       2019-11-22T19:43:57Z
  Last Modified Time:    2019-11-22T19:40:04Z
  Model Names:
    Name:   xgboost-model
    Value:  xgboost-model-1-def0e83e0d5f11-df5cc9fd0d5f11eaaa450aexample
Events:     <none>
```

The status field provides more information using the following fields: 
+ `Additional`: Additional information about the status of the hosting deployment\. This field is optional and only gets populated in case of error\. 
+ `Creation Time`: When the endpoint was created in SageMaker\. 
+ `Endpoint ARN`: The SageMaker endpoint ARN\. 
+ `Endpoint Config Name`: The SageMaker name of the endpoint configuration\. 
+ `Endpoint Name`: The SageMaker name of the endpoint\. 
+ `Endpoint Status`: The status of the endpoint\. 
+ `Endpoint URL`: The HTTPS URL that can be used to access the endpoint\. For more information, see [Deploy a Model on SageMaker Hosting Services](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model.html)\. 
+ `FailureReason`: If a create, update, or delete command fails, the cause is shown here\. 
+ `Last Check Time`: The last time the operator checked the status of the endpoint\. 
+ `Last Modified Time`: The last time the endpoint was modified\. 
+ `Model Names`: A key\-value pair of HostingDeployment model names to SageMaker model names\. 

#### Invoking the Endpoint<a name="invoking-the-endpoint"></a>

Once the endpoint status is `InService`, you can invoke the endpoint in two ways: using the AWS CLI, which does authentication and URL request signing, or using an HTTP client like cURL\. If you use your own client, you need to do AWS v4 URL signing and authentication on your own\. 

To invoke the endpoint using the AWS CLI, run the following command\. Make sure to replace the Region and endpoint\-name with your endpoint's Region and SageMaker endpoint name\. This information can be obtained from the output of `kubectl describe`\. 

```
# Invoke the endpoint with mock input data.
aws sagemaker-runtime invoke-endpoint \
  --region us-east-2 \
  --endpoint-name <endpoint name> \
  --body $(seq 784 | xargs echo | sed 's/ /,/g') \
  >(cat) \
  --content-type text/csv > /dev/null
```

For example, if your Region is `us-east-2` and your endpoint config name is `host-xgboost-f56b6b280d7511ea824b129926example`, then the following command would invoke the endpoint: 

```
aws sagemaker-runtime invoke-endpoint \
  --region us-east-2 \
  --endpoint-name host-xgboost-f56b6b280d7511ea824b1299example \
  --body $(seq 784 | xargs echo | sed 's/ /,/g') \
  >(cat) \
  --content-type text/csv > /dev/null
4.95847082138
```

Here, `4.95847082138` is the prediction from the model for the mock data\. 

#### Update HostingDeployment<a name="update-hostingdeployment"></a>

1. Once a HostingDeployment has a status of `InService`, it can be updated\. It might take about 10 minutes for HostingDeployment to be in service\. To verify that the status is `InService`, use the following command: 

   ```
   kubectl get hostingdeployments
   ```

1. The HostingDeployment can be updated before the status is `InService`\. The operator waits until the SageMaker endpoint is `InService` before applying the update\. 

   To apply an update, modify the `hosting.yaml` file\. For example, change the `initialInstanceCount` field from 1 to 2 as follows: 

   ```
   apiVersion: sagemaker.aws.amazon.com/v1
   kind: HostingDeployment
   metadata:
     name: host-xgboost
   spec:
       region: us-east-2
       productionVariants:
           - variantName: all-traffic
             modelName: xgboost-model
             initialInstanceCount: 2
             instanceType: ml.c5.large
       models:
           - name: xgboost-model
             executionRoleArn: arn:aws:iam::123456789012:role/service-role/AmazonSageMaker-ExecutionRole
             primaryContainer: xgboost
             containers:
               - xgboost
       containers:
           - containerHostname: xgboost
             modelDataUrl: s3://my-bucket/inference/xgboost-mnist/model.tar.gz
             image: 123456789012.dkr.ecr.us-east-2.amazonaws.com/xgboost:latest
   ```

1. Save the file, then use `kubectl` to apply your update as follows\. You should see the status change from `InService` to `ReconcilingEndpoint`, then `Updating`\. 

   ```
   $ kubectl apply -f hosting.yaml
   hostingdeployment.sagemaker.aws.amazon.com/host-xgboost configured
   
   $ kubectl get hostingdeployments
   NAME           STATUS                SAGEMAKER-ENDPOINT-NAME
   host-xgboost   ReconcilingEndpoint   host-xgboost-def0e83e0d5f11eaaa450a350abcdef
   
   $ kubectl get hostingdeployments
   NAME           STATUS     SAGEMAKER-ENDPOINT-NAME
   host-xgboost   Updating   host-xgboost-def0e83e0d5f11eaaa450a3507abcdef
   ```

SageMaker deploys a new set of instances with your models, switches traffic to use the new instances, and drains the old instances\. As soon as this process begins, the status becomes `Updating`\. After the update is complete, your endpoint becomes `InService`\. This process takes approximately 10 minutes\. 

#### Delete the HostingDeployment<a name="delete-the-hostingdeployment"></a>

1. Use `kubectl` to delete a HostingDeployment with the following command: 

   ```
   kubectl delete hostingdeployments host-xgboost
   ```

   Your output should look like the following: 

   ```
   hostingdeployment.sagemaker.aws.amazon.com "host-xgboost" deleted
   ```

1. To verify that the hosting deployment has been deleted, use the following command: 

   ```
   kubectl get hostingdeployments
   No resources found.
   ```

Endpoints that have been deleted do not incur any charges for SageMaker resources\. 

### ProcessingJob operator<a name="kubernetes-processing-job-operator"></a>

ProcessingJob operators are used to launch Amazon SageMaker processing jobs\. For more information on SageMaker processing jobs, see [CreateProcessingJob](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_CreateProcessingJob.html)\. 

**Topics**
+ [Create a ProcessingJob Using a YAML File](#kubernetes-processing-job-yaml)
+ [List ProcessingJobs](#kubernetes-processing-job-list)
+ [Describe a ProcessingJob](#kubernetes-processing-job-description)
+ [Delete a ProcessingJob](#kubernetes-processing-job-delete)

#### Create a ProcessingJob Using a YAML File<a name="kubernetes-processing-job-yaml"></a>

Follow these steps to create an Amazon SageMaker processing job by using a YAML file:

1. Download the `kmeans_preprocessing.py` pre\-processing script\.

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/kmeans_preprocessing.py
   ```

1. In one of your Amazon Simple Storage Service \(Amazon S3\) buckets, create a `mnist_kmeans_example/processing_code` folder and upload the script to the folder\.

1. Download the `kmeans-mnist-processingjob.yaml` file\.

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/kmeans-mnist-processingjob.yaml
   ```

1. Edit the YAML file to specify your `sagemaker-execution-role` and replace all instances of `my-bucket` with your S3 bucket\.

   ```
   ...
   metadata:
     name: kmeans-mnist-processing
   ...
     roleArn: arn:aws:iam::<acct-id>:role/service-role/<sagemaker-execution-role>
     ...
     processingOutputConfig:
       outputs:
         ...
             s3Output:
               s3Uri: s3://<my-bucket>/mnist_kmeans_example/output/
     ...
     processingInputs:
       ...
           s3Input:
             s3Uri: s3://<my-bucket>/mnist_kmeans_example/processing_code/kmeans_preprocessing.py
   ```

   The `sagemaker-execution-role` must have permissions so that SageMaker can access your S3 bucket, Amazon CloudWatch, and other services on your behalf\. For more information on creating an execution role, see [SageMaker Roles](https://docs.aws.amazon.com/sagemaker/latest/dg/sagemaker-roles.html#sagemaker-roles-createtrainingjob-perms)\.

1. Apply the YAML file using one of the following commands\.

   For cluster\-scoped installation:

   ```
   kubectl apply -f kmeans-mnist-processingjob.yaml
   ```

   For namespace\-scoped installation:

   ```
   kubectl apply -f kmeans-mnist-processingjob.yaml -n <NAMESPACE>
   ```

#### List ProcessingJobs<a name="kubernetes-processing-job-list"></a>

Use one of the following commands to list all the jobs created using the ProcessingJob operator\. `SAGEMAKER-JOB-NAME ` comes from the `metadata` section of the YAML file\.

For cluster\-scoped installation:

```
kubectl get ProcessingJob kmeans-mnist-processing
```

For namespace\-scoped installation:

```
kubectl get ProcessingJob -n <NAMESPACE> kmeans-mnist-processing
```

Your output should look similar to the following:

```
NAME                    STATUS     CREATION-TIME        SAGEMAKER-JOB-NAME
kmeans-mnist-processing InProgress 2020-09-22T21:13:25Z kmeans-mnist-processing-7410ed52fd1811eab19a165ae9f9e385
```

The output lists all jobs regardless of their status\. To remove a job from the list, see [Delete a Processing Job](https://docs.aws.amazon.com/sagemaker/latest/dg/kubernetes-processing-job-operator.html#kubernetes-processing-job-delete)\.

**ProcessingJob Status**
+ `SynchronizingK8sJobWithSageMaker` – The job is first submitted to the cluster\. The operator has received the request and is preparing to create the processing job\.
+ `Reconciling` – The operator is initializing or recovering from transient errors, along with others\. If the processing job remains in this state, use the `kubectl` `describe` command to see the reason in the `Additional` field\.
+ `InProgress | Completed | Failed | Stopping | Stopped` – Status of the SageMaker processing job\. For more information, see [DescribeProcessingJob](https://docs.aws.amazon.com/sagemaker/latest/APIReference/API_DescribeProcessingJob.html#sagemaker-DescribeProcessingJob-response-ProcessingJobStatus)\.
+ `Error` – The operator cannot recover by reconciling\.

Jobs that have completed, stopped, or failed do not incur further charges for SageMaker resources\.

#### Describe a ProcessingJob<a name="kubernetes-processing-job-description"></a>

Use one of the following commands to get more details about a processing job\. These commands are typically used for debugging a problem or checking the parameters of a processing job\.

For cluster\-scoped installation:

```
kubectl describe processingjob kmeans-mnist-processing
```

For namespace\-scoped installation:

```
kubectl describe processingjob kmeans-mnist-processing -n <NAMESPACE>
```

The output for your processing job should look similar to the following\.

```
$ kubectl describe ProcessingJob kmeans-mnist-processing
Name:         kmeans-mnist-processing
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"sagemaker.aws.amazon.com/v1","kind":"ProcessingJob","metadata":{"annotations":{},"name":"kmeans-mnist-processing",...
API Version:  sagemaker.aws.amazon.com/v1
Kind:         ProcessingJob
Metadata:
  Creation Timestamp:  2020-09-22T21:13:25Z
  Finalizers:
    sagemaker-operator-finalizer
  Generation:        2
  Resource Version:  21746658
  Self Link:         /apis/sagemaker.aws.amazon.com/v1/namespaces/default/processingjobs/kmeans-mnist-processing
  UID:               7410ed52-fd18-11ea-b19a-165ae9f9e385
Spec:
  App Specification:
    Container Entrypoint:
      python
      /opt/ml/processing/code/kmeans_preprocessing.py
    Image Uri:  763104351884.dkr.ecr.us-west-2.amazonaws.com/pytorch-training:1.5.0-cpu-py36-ubuntu16.04
  Environment:
    Name:   MYVAR
    Value:  my_value
    Name:   MYVAR2
    Value:  my_value2
  Network Config:
  Processing Inputs:
    Input Name:  mnist_tar
    s3Input:
      Local Path:   /opt/ml/processing/input
      s3DataType:   S3Prefix
      s3InputMode:  File
      s3Uri:        s3://<s3bucket>-us-west-2/algorithms/kmeans/mnist/mnist.pkl.gz
    Input Name:     source_code
    s3Input:
      Local Path:   /opt/ml/processing/code
      s3DataType:   S3Prefix
      s3InputMode:  File
      s3Uri:        s3://<s3bucket>/mnist_kmeans_example/processing_code/kmeans_preprocessing.py
  Processing Output Config:
    Outputs:
      Output Name:  train_data
      s3Output:
        Local Path:    /opt/ml/processing/output_train/
        s3UploadMode:  EndOfJob
        s3Uri:         s3://<s3bucket>/mnist_kmeans_example/output/
      Output Name:     test_data
      s3Output:
        Local Path:    /opt/ml/processing/output_test/
        s3UploadMode:  EndOfJob
        s3Uri:         s3://<s3bucket>/mnist_kmeans_example/output/
      Output Name:     valid_data
      s3Output:
        Local Path:    /opt/ml/processing/output_valid/
        s3UploadMode:  EndOfJob
        s3Uri:         s3://<s3bucket>/mnist_kmeans_example/output/
  Processing Resources:
    Cluster Config:
      Instance Count:     1
      Instance Type:      ml.m5.xlarge
      Volume Size In GB:  20
  Region:                 us-west-2
  Role Arn:               arn:aws:iam::<acct-id>:role/m-sagemaker-role
  Stopping Condition:
    Max Runtime In Seconds:  1800
  Tags:
    Key:    tagKey
    Value:  tagValue
Status:
  Cloud Watch Log URL:             https://us-west-2.console.aws.amazon.com/cloudwatch/home?region=us-west-2#logStream:group=/aws/sagemaker/ProcessingJobs;prefix=kmeans-mnist-processing-7410ed52fd1811eab19a165ae9f9e385;streamFilter=typeLogStreamPrefix
  Last Check Time:                 2020-09-22T21:14:29Z
  Processing Job Status:           InProgress
  Sage Maker Processing Job Name:  kmeans-mnist-processing-7410ed52fd1811eab19a165ae9f9e385
Events:                            <none>
```

#### Delete a ProcessingJob<a name="kubernetes-processing-job-delete"></a>

When you delete a processing job, the SageMaker processing job is removed from Kubernetes but the job isn't deleted from SageMaker\. If the job status in SageMaker is `InProgress` the job is stopped\. Processing jobs that are stopped do not incur any charges for SageMaker resources\. Use one of the following commands to delete a processing job\. 

For cluster\-scoped installation:

```
kubectl delete processingjob kmeans-mnist-processing
```

For namespace\-scoped installation:

```
kubectl delete processingjob kmeans-mnist-processing -n <NAMESPACE>
```

The output for your processing job should look similar to the following\.

```
processingjob.sagemaker.aws.amazon.com "kmeans-mnist-processing" deleted
```



**Note**  
SageMaker does not delete the processing job\. Stopped jobs continue to show in the SageMaker console\. The `delete` command takes a few minutes to clean up the resources from SageMaker\.

### HostingAutoscalingPolicy \(HAP\) Operator<a name="kubernetes-hap-operator"></a>

The HostingAutoscalingPolicy \(HAP\) operator takes a list of resource IDs as input and applies the same policy to each of them\. Each resource ID is a combination of an endpoint name and a variant name\. The HAP operator performs two steps: it registers the resource IDs and then applies the scaling policy to each resource ID\. `Delete` undoes both actions\. You can apply the HAP to an existing SageMaker endpoint or you can create a new SageMaker endpoint using the [HostingDeployment operator](https://docs.aws.amazon.com/sagemaker/latest/dg/hosting-deployment-operator.html#create-a-hostingdeployment)\. You can read more about SageMaker autoscaling in the [ Application Autoscaling Policy documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/endpoint-auto-scaling.html)\.

**Note**  
In your `kubectl` commands, you can use the short form, `hap`, in place of `hostingautoscalingpolicy`\.

**Topics**
+ [Create a HostingAutoscalingPolicy Using a YAML File](#kubernetes-hap-job-yaml)
+ [List HostingAutoscalingPolicies](#kubernetes-hap-list)
+ [Describe a HostingAutoscalingPolicy](#kubernetes-hap-describe)
+ [Update a HostingAutoscalingPolicy](#kubernetes-hap-update)
+ [Delete a HostingAutoscalingPolicy](#kubernetes-hap-delete)
+ [Update or Delete an Endpoint with a HostingAutoscalingPolicy](#kubernetes-hap-update-delete-endpoint)

#### Create a HostingAutoscalingPolicy Using a YAML File<a name="kubernetes-hap-job-yaml"></a>

Use a YAML file to create a HostingAutoscalingPolicy \(HAP\) that applies a predefined or custom metric to one or multiple SageMaker endpoints\.

Amazon SageMaker requires specific values in order to apply autoscaling to your variant\. If these values are not specified in the YAML spec, the HAP operator applies the following default values\.

```
# Do not change
Namespace                    = "sagemaker"
# Do not change
ScalableDimension            = "sagemaker:variant:DesiredInstanceCount"
# Only one supported
PolicyType                   = "TargetTrackingScaling"
# This is the default policy name but can be changed to apply a custom policy
DefaultAutoscalingPolicyName = "SageMakerEndpointInvocationScalingPolicy"
```

Use the following samples to create a HAP that applies a predefined or custom metric to one or multiple endpoints\.

##### Sample 1: Apply a Predefined Metric to a Single Endpoint Variant<a name="kubernetes-hap-predefined-metric"></a>

1. Download the sample YAML file for a predefined metric using the following command:

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/hap-predefined-metric.yaml
   ```

1. Edit the YAML file to specify your `endpointName`, `variantName`, and `region`\.

1. Use one of the following commands to apply a predefined metric to a single resource ID \(endpoint name and variant name combination\)\.

   For cluster\-scoped installation:

   ```
   kubectl apply -f hap-predefined-metric.yaml
   ```

   For namespace\-scoped installation:

   ```
   kubectl apply -f hap-predefined-metric.yaml -n <NAMESPACE>
   ```

##### Sample 2: Apply a Custom Metric to a Single Endpoint Variant<a name="kubernetes-hap-custom-metric"></a>

1. Download the sample YAML file for a custom metric using the following command:

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/hap-custom-metric.yaml
   ```

1. Edit the YAML file to specify your `endpointName`, `variantName`, and `region`\.

1. Use one of the following commands to apply a custom metric to a single resource ID \(endpoint name and variant name combination\) in place of the recommended `SageMakerVariantInvocationsPerInstance`\.
**Note**  
Amazon SageMaker does not check the validity of your YAML spec\.

   For cluster\-scoped installation:

   ```
   kubectl apply -f hap-custom-metric.yaml
   ```

   For namespace\-scoped installation:

   ```
   kubectl apply -f hap-custom-metric.yaml -n <NAMESPACE>
   ```

##### Sample 3: Apply a Scaling Policy to Multiple Endpoints and Variants<a name="kubernetes-hap-scaling-policy"></a>

You can use the HAP operator to apply the same scaling policy to multiple resource IDs\. A separate `scaling_policy` request is created for each resource ID \(endpoint name and variant name combination\)\.

1. Download the sample YAML file for a predefined metric using the following command:

   ```
   wget https://raw.githubusercontent.com/aws/amazon-sagemaker-operator-for-k8s/master/samples/hap-predefined-metric.yaml
   ```

1. Edit the YAML file to specify your `region` and multiple `endpointName` and `variantName` values\.

1. Use one of the following commands to apply a predefined metric to multiple resource IDs \(endpoint name and variant name combinations\)\.

   For cluster\-scoped installation:

   ```
   kubectl apply -f hap-predefined-metric.yaml
   ```

   For namespace\-scoped installation:

   ```
   kubectl apply -f hap-predefined-metric.yaml -n <NAMESPACE>
   ```

##### Considerations for HostingAutoscalingPolicies for Multiple Endpoints and Variants<a name="kubernetes-hap-scaling-considerations"></a>

The following considerations apply when you use multiple resource IDs:
+ If you apply a single policy across multiple resource IDs, one PolicyARN is created per resource ID\. Five endpoints have five PolicyARNs\. When you run the `describe` command on the policy, the responses show up as one job and include a single job status\.
+ If you apply a custom metric to multiple resource IDs, the same dimension or value is used for all the resource ID \(variant\) values\. For example, if you apply a customer metric for instances 1\-5, and the endpoint variant dimension is mapped to variant 1, when variant 1 exceeds the metrics, all endpoints are scaled up or down\.
+ The HAP operator supports updating the list of resource IDs\. If you modify, add, or delete resource IDs to the spec, the autoscaling policy is removed from the previous list of variants and applied to the newly specified resource ID combinations\. Use the [https://docs.aws.amazon.com/sagemaker/latest/dg/kubernetes-hap-operator.html#kubernetes-hap-describe](https://docs.aws.amazon.com/sagemaker/latest/dg/kubernetes-hap-operator.html#kubernetes-hap-describe) command to list the resource IDs to which the policy is currently applied\.

#### List HostingAutoscalingPolicies<a name="kubernetes-hap-list"></a>

Use one of the following commands to list all HostingAutoscalingPolicies \(HAPs\) created using the HAP operator\.

For cluster\-scoped installation:

```
kubectl get hap
```

For namespace\-scoped installation:

```
kubectl get hap -n <NAMESPACE>
```

Your output should look similar to the following:

```
NAME             STATUS   CREATION-TIME
hap-predefined   Created  2021-07-13T21:32:21Z
```

Use the following command to check the status of your HostingAutoscalingPolicy \(HAP\)\.

```
kubectl get hap <job-name>
```

One of the following values is returned:
+ `Reconciling` – Certain types of errors show the status as `Reconciling` instead of `Error`\. Some examples are server\-side errors and endpoints in the `Creating` or `Updating` state\. Check the `Additional` field in status or operator logs for more details\.
+ `Created`
+ `Error`

**To view the autoscaling endpoint to which you applied the policy**

1. Open the Amazon SageMaker console at [https://console\.aws\.amazon\.com/sagemaker/](https://console.aws.amazon.com/sagemaker/)\.

1. In the left side panel, expand **Inference**\.

1. Choose **Endpoints**\.

1. Select the name of the endpoint of interest\.

1. Scroll to the **Endpoint runtime settings** section\.

#### Describe a HostingAutoscalingPolicy<a name="kubernetes-hap-describe"></a>

Use the following command to get more details about a HostingAutoscalingPolicy \(HAP\)\. These commands are typically used for debugging a problem or checking the resource IDs \(endpoint name and variant name combinations\) of a HAP\.

```
kubectl describe hap <job-name>
```

#### Update a HostingAutoscalingPolicy<a name="kubernetes-hap-update"></a>

The HostingAutoscalingPolicy \(HAP\) operator supports updates\. You can edit your YAML spec to change the values and then reapply the policy\. The HAP operator deletes the existing policy and applies the new policy\.

#### Delete a HostingAutoscalingPolicy<a name="kubernetes-hap-delete"></a>

Use one of the following commands to delete a HostingAutoscalingPolicy \(HAP\) policy\.

For cluster\-scoped installation:

```
kubectl delete hap hap-predefined
```

For namespace\-scoped installation:

```
kubectl delete hap hap-predefined -n <NAMESPACE>
```

This command deletes the scaling policy and deregisters the scaling target from Kubernetes\. This command returns the following output:

```
hostingautoscalingpolicies.sagemaker.aws.amazon.com "hap-predefined" deleted
```

#### Update or Delete an Endpoint with a HostingAutoscalingPolicy<a name="kubernetes-hap-update-delete-endpoint"></a>

To update an endpoint that has a HostingAutoscalingPolicy \(HAP\), use the `kubectl` `delete` command to remove the HAP, update the endpoint, and then reapply the HAP\.

To delete an endpoint that has a HAP, use the `kubectl` `delete` command to remove the HAP before you delete the endpoint\.