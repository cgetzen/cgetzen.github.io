# Using remote state as a cache

Note: Terraform only provides identity and access management through sentinel, a paid feature. This post does not address sentinel.

I recently found myself looking at the tradeoff between performance and simplicity in terraform. It looked a little bit like this:


Performant(ish):
modules/module1:
```terraform
variable "var" {}

resource "some_resource" "resource" {
  ...
}
```

states/production:
```terraform
data "huge_performance_hit" {
  ...
}

module "module1a" {
  source = "../modules/module1"
  var = data.huge_performance_hit.resources["user_supplied_info"]
}

module "module1b" {
  ...
}
```

While this worked decently, the data lookup ended up firing off about 50 requests (due to an api that paged results).
Additionally, this terraform repo was supposed to take the place of IT tickets and would be touched by most of the engineers at the org,
meaning that everything had to be as straightforward as possible to reduce confusion, questions, and frustration. Users would only be commiting
to states/production, which means I could move as much as I could out of this folder.

This is the intermediate solution:
modules/module1:
```terraform
variable "user_supplied_info" {}

data "huge_performance_hit" "all_info" {
  ...
}

locals {
  var = data.huge_performance_hit.resources[var.user_supplied_info]
}

resource "some_resource" "resource" {
  ...
}
```

states/production:
```terraform
module "module1a" {
  source = "../modules/module1"
  user_supplied_info = "user_supplied_info"
}

module "module1b" {
  ...
}
```

While this looked good from the point of view of states/production, this will definitely not scale to the whole org -- the performance hit is multiplied by how many modules there are.
By putting the huge_performance_hit in a remote state, I was able to cache it, avoid the 50+ HTTP requests, and keep everything performant:

states/large_performance_hit:
```terraform
data "huge_performance_hit" "all_info" {
  ...
}
output "resources" {
  value = data.huge_performance_hit.resources
}
```

modules/module1:
```terraform
variable "user_supplied_info" {}

data "terraform_remote_state" "cache" {
  ...
}

locals {
  var = data.terraform_remote_state.cache.resources[var.user_supplied_info]
}

resource "some_resource" "resource" {
  ...
}
```

We are currently using S3 as a remote state, and putting a `terraform refresh` for states/large_performance_hit in our nightly CI.
This implementation hits S3 for every module, which is much faster than the provider API. However, as we scale, if we see hitting S3 as an issue for every module, we could instead
make the state local, and simply do a plan on states/large_performance_hit before plan/applying on states/production. This way, there will only be a single lookup.
