# "Global" variables

Multiple nested modules can provide power and flexibility but can become brittle if not designed carefully.
Here's an example.

/modules/apps/app1:
```terraform
variable "env" {}
variable "stage" {}
variable "web_instance_count" {}
variable "worker_instance_count" {}

module "web" {
  source = "../../components/web"
  app = "app1"
  instance_count = var.web_instance_count
  env = var.env
  stage = var.stage
}

module "background_worker" {
  source = "../../components/background_worker"
  app = "app1"
  instance_count = var.worker_instance_count
  env = var.env
  stage = var.stage
}

...
```

/modules/components/web:
```terraform
variable "app" {}
variable "env" {}
variable "stage" {}
variable "instance_count" {}

module "web" {
  source = "../../base/aws_instance"
  type = "web"
  app = var.app
  stage = var.stage
  env = var.env
  instance_count = var.instance_count
}

module "elb" {
  ...
}
```

/modules/base/ec2_instance:
```terraform
variable "app" {}
variable "env" {}
variable "stage" {}
variable "instance_count" {}

resource "aws_instance" "instance" {
  count = var.instance_count
  name = "${var.app}-${var.env}-${var.stage}-${count.index}"
  ...
}
```

/state/app1_staging1:
```terraform
module "app1_staging1" {
  source = "../../modules/apps/app1"
  env = "production"
  stage = "penetration-test"
  web_instance_count = 4
  worker_instance_count = 2
}
```

OK. What if we want to shard our application for performance or compliance. We would have to add a "partition" variable to every file!
If we have tons of modules, as most large terraform repos do, this can cause the need for massive reformatting commits.

While terraform doesn't have "global" variables, we can remove this obstacle by putting domain language in a single map variable.

/modules/apps/app1:
```terraform
variable "data" {}

module "web" {
  source = "../../components/web"
  app = "app1"
  instance_count = var.web_instance_count
  data = var.data
}

module "background_worker" {
  source = "../../components/background_worker"
  app = "app1"
  instance_count = var.worker_instance_count
  data = var.data
}

...
```

/modules/components/web:
```terraform
variable "data" {}
variable "app" {}
variable "instance_count" {}

module "web" {
  source = "../../base/aws_instance"
  type = "web"
  app = var.app
  instance_count = var.instance_count
  data = var.data
}

module "elb" {
  ...
}
```

/modules/base/ec2_instance:
```terraform
variable "app" {}
variable "instance_count" {}
variable "data" {}

resource "aws_instance" "instance" {
  count = var.instance_count
  name = "${var.app}-${var.data["env"]}-${var.data["stage"]}-${count.index}"
  ...
}
```

/state/app1_staging1:
```terraform
module "app1_staging1" {
  source = "../../modules/apps/app1"
  web_instance_count = 4
  worker_instance_count = 2
  data = {
    "env": "production",
    "stage": "penetration-test",
  }
}
```

After these changes, it becomes easier to add the partition variable -- we only have to add it to /state/app1_staging1 and /modules/base/ec2_instance.

Additionally, it is possible to seperate the change to the module, and the change to the app, by providing fallbacks:

/modules/base/ec2_instance:
```terraform
...

locals {
  partition = lookup(var.data, "partition", "unzoned")
}
```
