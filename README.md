#Installing kube2iam on PKS 

## pre-reqs

* PKS foundation
* atleast 1 cluster deployed
* using flannel for CNI
* PSP or privledged containers enabled. I opted for PSP. if enabling privledged without PSP you can skip step 2.


## Steps
1. create a namespace and service account for kube2iam

```bash
#create ns
kubectl create ns kube2iam
#create service account and role
kubectl apply -f kube2iam-rbac.yml
```

2. If you are using `PSP`s continue with this step if not skip this step. you will likely already have some policies set up. I will not go through the details of `PSP`s since that is not in scope. PKS will automatically creat a `pks-privledged` `PSP` for you. we will create a cluster role and bidning that allow the service account to use the PSP. You will also need to be sure to have "Allow Privileged" and "PodSecurityPolicies" enabled in your PKS plan.

```bash
kubectl apply -f kube2iam-psp.yml
```


3. add the following permissions to the kuberenetes worker IAM role policy. If you installed PKS using the terraforming-aws this role should be `<env_name>_pks-worker`. When deploying PKS the "Kubernetes Cloud Provider Configuration" will alow you to enter this IAM instance profile name, this will automatically be attached to all workers.  you could very easily add these extra policy permissions to the terraform.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "sts:AssumeRole"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

4. add a trust relationship to the role you want the containers to assume. for this example I have created a simple role named `k8s-pods-role` . edit the trust relationship on the role to allow for the pks worker role to assume the pod role. this should look like:

* go to the role you want to assume.
* click "trust relatiosnhips"
* click "edit trust relationships"
* past the below in after replacing the arn with your k8s worker role arn

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/kubernetes-worker-role"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

5. deploy the k8s daemonset. before runnning the below command you will need to modify the arn in the daemonset yaml to match your base role arn.

```bash
kubectl apply -f kube2iam-daemonset.yml
```

6. test out the functionality with the example deployment. you may need to update the annotation to have your role name. once you exec in you should be able to use the aws cli as if you were in that role.

```bash
kubectl apply -f example-pod.yml

kubectl exec -ti aws-cli /bin/bash
```