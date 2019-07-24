---
date: 2019-06-02
title: Upgrading EKS cluster
categories:
   - Admin
description: How to upgrade your EKS cluster, or change EKS worker nodes without losing Spinnaker
type: Document
---

## Upgrade master

This can be achieved directly from AWS console, under `EKS` section. There's a `Upgrade cluster version` button that can be used to upgrade the master:

![](/images/cluster-upgrade-master.png)

If you are using Terraform scripts for managing your cluster, the [version](https://www.terraform.io/docs/providers/aws/r/eks_cluster.html#version) can be specified in `aws_eks_cluster` resource:

```
resource "aws_eks_cluster" "aws-eks" {
  name     = "${var.cluster-name}"
  role_arn = "${aws_iam_role.aws-eks-cluster.arn}"
  
  version  = "${var.cluster-master-version}"

  vpc_config {
    security_group_ids = ["${aws_security_group.aws-eks-cluster.id}"]
    subnet_ids         = ["${aws_subnet.aws-eks-subnet-1.id}", "${aws_subnet.aws-eks-subnet-2.id}"]
  }

  depends_on = [
    "aws_iam_role_policy_attachment.aws-eks-cluster-AmazonEKSClusterPolicy",
    "aws_iam_role_policy_attachment.aws-eks-cluster-AmazonEKSServicePolicy",
    "aws_iam_role.aws-eks-cluster"
  ]
}
```

_Effect: Doing this will only upgrade the master, but EKS worker nodes will not be affected and will continue running normally._


## Upgrade EKS worker nodes

In this step you actually change the AMI used for the worker nodes in EKS launch configuration, and then double the number of nodes in the cluster in their Auto Scaling Group configuration. If the original cluster had 3 nodes, you increase that to 6 nodes:

![](/images/cluster-upgrade-launch-config.png)
![](/images/cluster-upgrade-asg.png)

If you are using Terraform scripts, these settings are available under [image_id](https://www.terraform.io/docs/providers/aws/r/launch_configuration.html#image_id) of `aws_launch_configuration` resource:

```
resource "aws_launch_configuration" "aws-eks" {
  associate_public_ip_address = false
  iam_instance_profile        = "${aws_iam_instance_profile.aws-eks-node.name}"
  image_id                    = "${data.aws_ami.eks-worker.id}"
  instance_type               = "${var.ec2-instance-type}"
  name_prefix                 = "${var.cluster-name}-node"
  security_groups             = ["${aws_security_group.aws-eks-node.id}"]
  user_data_base64            = "${base64encode(local.aws-eks-node-userdata)}"

  lifecycle {
    create_before_destroy = true
  }

  root_block_device {
    volume_size = "50"
  }
}
```

And Auto Scaling Group size is available under [max_size and desired_capacity](https://www.terraform.io/docs/providers/aws/r/autoscaling_group.html#desired_capacity) of `aws_autoscaling_group` resource:

```
resource "aws_autoscaling_group" "aws-eks" {
  desired_capacity     = "${var.desired-ec2-instances}"
  launch_configuration = "${aws_launch_configuration.aws-eks.id}"
  max_size             = "${var.max-ec2-instances}" 
  min_size             = "${var.min-ec2-instances}" 
  name                 = "${var.cluster-name}-asg"
  vpc_zone_identifier  = ["${aws_subnet.aws-eks-subnet-1.id}", "${aws_subnet.aws-eks-subnet-2.id}"]

  tag {
    key                 = "Name"
    value               = "${var.cluster-name}"
    propagate_at_launch = true
  }

  tag {
    key                 = "kubernetes.io/cluster/${var.cluster-name}"
    value               = "owned"
    propagate_at_launch = true
  }

  depends_on = ["aws_eks_cluster.aws-eks"]

}
```

_Effect: Cluster size will be doubled, new worker nodes will be created with the new version, and old worker nodes will still be running and available with the old version._

## Scale down Auto Scaling Group

After the new nodes are correctly registered and available in kubernetes, it's time to decrease the Auto Scaling Group to its original size.

_Effect: All pods in the old nodes will be automatically moved to the new nodes by kubernetes._ After Spinnaker pods are restarted, you should be able to continue using Spinnaker as usual.
