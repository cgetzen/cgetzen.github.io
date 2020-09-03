# Terraform Tips

## Never use data blocks in modules

Instead, pull the data blocks outside the module to the root, and pass in the desired attributes from it as variables.
This makes the module more deterministic, and protects it from changes to other resources.
