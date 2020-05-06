# Load balance between clusters on AWS

Clusters as cattle is a hard thing to implement because the mechanics often just aren't there.

In GKE, you are able to [register services with extant load-balancers](https://github.com/kubernetes/kubernetes/pull/13005/files) using `loadBalancerIP`, but there is [currently no way to do this with AWS](https://github.com/kubernetes/kubernetes/issues/10323#issuecomment-136864967).

This is a really great feature to have, because it allows you to assign services across multiple clusters to the same LB -- adding redundancy at the cluster level and allowing you to add or remove clusters easily.

While we can replicate this on AWS, the mechanics aren't as seamlessly and automated. Here's how.

General Process:

- Create a service of type NodePort or LoadBalancer, and specify a nodePort in the svc definition.
- Create an LB (terraform, console, CF), and ensure nodePort matches targetGroup port

Here's an example

```terraform
resource "aws_security_group" "app1" {
  name        = "app1-${var.stage}"
  description = "app1 LB"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "aws_security_group" "cluster1" {
  filter {
    name   = "tag:aws:eks:cluster-name"
    values = ["cluster1"]
  }
}

data "aws_security_group" "cluster2" {
  filter {
    name   = "tag:aws:eks:cluster-name"
    values = ["cluster2"]
  }
}

resource "aws_lb" "app1" {
  name               = "app1-${var.stage}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.app1.id, data.aws_security_group.cluster1.id, data.aws_security_group.cluster2.id]
  subnets            = var.public_subnet_ids
}

resource "aws_lb_listener" "app1-https" {
  load_balancer_arn = aws_lb.app1.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-2016-08"
  certificate_arn   = var.cert_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app1.arn
  }
}

resource "aws_lb_listener" "app1-http" {
  load_balancer_arn = aws_lb.app1.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app1.arn
  }
}

resource "aws_lb_target_group" "app1" {
  name_prefix  = "app1" # This helps with create_before_destroy, and updates in general
  port         = 30740
  protocol     = "HTTP"
  vpc_id       = var.vpc_id

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_attachment" "app1" {
  count = length(var.auto_scaling_group_names) # These are EKS' ASGs
  autoscaling_group_name = var.auto_scaling_group_names[count.index]
  alb_target_group_arn   = aws_lb_target_group.app1.arn
}
```

Now both ASGs will be registered in the target group, and the target group will manage healthchecks! If you spin up two identical services with the same nodePorts on each
cluster, they will both be load balanced.

### What's next?

The above was a brief overview of load balancing between clusters within a region. This is perfect for any stateful service.

But if you have a stateless service, it may be more advantageous to load balance between clusters in different regions.

We are currently doing this with AWS' Global Accelerator, which allows you to assign an IP to load balancers in different regions. It will then route customers to the closest region, and even do healthchecking on the registered load balancers.

You can use both load balancing in tandem, but there is little benefit to that.
