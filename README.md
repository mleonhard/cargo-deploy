# cargo-deploy
Infrastructure-as-code tool.

Example:
```toml
# Cargo.toml
# Example of customizing binaries in Cargo.toml.
[[bin]]
name = "server"
test = false
bench = false

[build-dependencies]
deploy-core = "1.0.47"
deploy-aws = "1.0.17"
deploy-ssh = "1.0.4"
```

```rust
// src/bin/server.rs
fn main() {
  // Read env vars...
  // Start server...
}
```

```rust
// deploy.rs
use deploy::*;

const tz = TimeZone::usa::PACIFIC;

fn main() {
  let org_name = OrgName::new("example-com");
  let owner_name = OwnerName::from_env("GIT_BRANCH");
  let instance_name = InstanceName::from_env("GIT_BRANCH");
  let aws_creds = AwsCreds::from_env("AWS_KEY_ID", "AWS_SECRET");
  let region = AwsRegion::from_env("AWS_REGION");
  let backups_region = AwsRegion::from_env("AWS_BACKUPS_REGION");

  let backups_bucket = S3Bucket::new(
    &backups_region,
    &org_name,
    &instance_name,
    "backups",
  )
  .unwrap()
  .with_auto_delete(days(60));
  let backup_binary = Binary::new("backup").unwrap();
  let backup_schedule = Schedule::daily(tz, 3..4).unwrap();
  let server_port = Port::new(8080).unwrap();
  let server = Server::new("server").unwrap();
  let linuxConfig = AlpineLinuxConfig::new()
    .with_env("BACKUPS_BUCKET_NAME", backups_bucket.name())
    .unwrap()
    .with_scheduled_job(&backup_binary, backup_schedule);
    .with_env("SERVER_PORT", &server_port)
    .unwrap()
    .with_server(&server);
  let vm = EC2Instance::new(
    "vm",
    &region,
    EC2InstanceType::R7_Nano,
    &linuxConfig,
  )
    .unwrap()
    .with_perm(backups_bucket.write_perm());
  let eip = Ec2ElasticIp::new(&vm).unwrap();
  let aws_owner = AwsIamAccount::new(format!("{owner_name}@example.com")).unwrap();
  let aws_resources = AWSResourceSet::new(aws_owner, instance_name, (vm,));

  let state_store = S3StateStore(&aws_creds, &org_name, &instance_name);
  deploy::run(state_store, aws_resources);
}
```
