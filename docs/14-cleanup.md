# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the all worker instances, then afterwards delete controller instances:

```sh
echo "Issuing shutdown to worker nodes.. " && \
aws --profile k8s ec2 terminate-instances \
  --instance-ids \
    $(aws --profile k8s ec2 describe-instances --filters \
      "Name=tag:Name,Values=worker-0,worker-1,worker-2" \
      "Name=instance-state-name,Values=running" \
      --output text --query 'Reservations[].Instances[].InstanceId')

echo "Issuing shutdown to master nodes.. " && \
aws --profile k8s ec2 terminate-instances \
  --instance-ids \
    $(aws --profile k8s ec2 describe-instances --filter \
      "Name=tag:Name,Values=controller-0,controller-1,controller-2" \
      "Name=instance-state-name,Values=running" \
      --output text --query 'Reservations[].Instances[].InstanceId')

aws --profile k8s ec2 delete-key-pair --key-name kubernetes
```

## Networking

Delete the external load balancer network resources:

```sh
aws --profile k8s elbv2 delete-load-balancer --load-balancer-arn "${LOAD_BALANCER_ARN}"
aws --profile k8s elbv2 delete-target-group --target-group-arn "${TARGET_GROUP_ARN}"
aws --profile k8s ec2 delete-security-group --group-id "${SECURITY_GROUP_ID}"
ROUTE_TABLE_ASSOCIATION_ID="$(aws --profile k8s ec2 describe-route-tables \
  --route-table-ids "${ROUTE_TABLE_ID}" \
  --output text --query 'RouteTables[].Associations[].RouteTableAssociationId')"
aws --profile k8s ec2 disassociate-route-table --association-id "${ROUTE_TABLE_ASSOCIATION_ID}"
aws --profile k8s ec2 delete-route-table --route-table-id "${ROUTE_TABLE_ID}"
echo "Waiting a minute for all public address(es) to be unmapped.. " && sleep 60

aws --profile k8s ec2 detach-internet-gateway \
  --internet-gateway-id "${INTERNET_GATEWAY_ID}" \
  --vpc-id "${VPC_ID}"
aws --profile k8s ec2 delete-internet-gateway --internet-gateway-id "${INTERNET_GATEWAY_ID}"
aws --profile k8s ec2 delete-subnet --subnet-id "${SUBNET_ID}"
aws --profile k8s ec2 delete-vpc --vpc-id "${VPC_ID}"
```
