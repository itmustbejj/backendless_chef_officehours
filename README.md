# Backendless Chef in AWS
A Chef Server cluster utilizing Amazon services for high availability along with autoscaled frontends.

![Chef Server Backendless Diagram](https://cloud.githubusercontent.com/assets/382062/21002669/26f69096-bce4-11e6-903c-153bb040ae16.png)

## What is CloudFormation?
- It’s a provisioning system
- You write declarative templates that describe AWS resources.
- The new YAML syntax is awesome. Get them brackets outa here.
- Engine that turns those templates into AWS Resources (called Stacks)
- Takes parameters for customization and produces outputs that can be consumed by other templates
- All state is stored by AWS
- It is not a configuration management system or an orchestration tool.
- Not usable on any infrastructure other than AWS
- [The Cloudformation Way Deck](https://docs.google.com/presentation/d/1kCZE0Kk-I8g0GZ5Nxk0XZZxOkeXC1IzcpgHt1zMnFbw/edit#slide=id.p) by Irving Popovetsky

## What does this template provision?
- A "bootstrap" frontend in an Auto Scaling Group of 1.
- A second frontend in an Auto Scaling Group that will scale up to 3 total.
- A Multi AZ Elastic Load Balancer instance.
- A Multi AZ RDS Postgres database.
- An ElasticSearch cluster that defaults to 3 shards.
- Basic CloudWatch alarms. (WIP)
- Various security groups, iam profiles, and various pieces to connect the things.

# Externalizing Chef services in your chef-server.rb
External Postgres settings for chef-server.rb:
```
postgresql['external'] = true
postgresql['vip'] = 'my.vip.here'
postgresql['port'] = '5432'  # not required
postgresql['db_superuser'] = 'chefadmin'
postgresql['db_superuser_password'] = '!@m@dm!nh3@rm3r0@r'
```

External Bookshelf in your Postgres DB:
```
bookshelf['storage_type'] = :sql
```

External Bookshelf in an S3 bucket
```
bookshelf['enable'] = false
# If the host isn't us-east-1, you need to use the region-specific s3 endpoints.
bookshelf['vip'] = 's3-us-west-2.amazonaws.com'
bookshelf['external_url'] = 'https://s3-us-west-2.amazonaws.com'
bookshelf['access_key_id'] = '#{node['aws']['access_key_id']}'
bookshelf['secret_access_key'] = '#{node['aws']['secret_access_key']}'
bookshelf['s3_bucket'] = 'your-bookshelf-bucket'
```
## Spinning up a cluster
- https://github.com/irvingpop/cloudformation_automate
- Bring your own VPC or use the VPC templates included in the repo.
    - Make sure you select the template with the correct number of AZ's for your region.
    - https://github.com/jsonmaur/aws-regions
- TODO: ReadMe example is missing the ContactEmail parameter. Shame on me.
- Go learn a skill or trade while your CloudFormation stack finishes creating.
- Log on to the bootstrap frontend and set up a user and an organization.
    - $ `chef-server-ctl user-create USER_NAME FIRST_NAME LAST_NAME EMAIL 'PASSWORD' --filename FILE_NAME`
    - `$ chef-server-ctl org-create short_name 'full_organization_name' --association_user user_name --filename ORGANIZATION-validator.pem`

## The bootstrap frontend
- Used for initialization of Chef services during installation. 
- Secrets generated during installation are copied to other frontends as well as an s3 bucket.
- There is a database migration state file that tracks current migration level.
-   `/var/opt/opscode/upgrades/migration-level`
-   Only run `chef-server-ctl upgrade` on the bootstrap frontend (more below).

## Upgrading a Chef Server with an External Database
- When it comes to upgrading a Chef server with an external db, [partybus](https://github.com/chef/chef-server/tree/master/omnibus/partybus) is your friend: 
    - `chef-server-ctl upgrade` uses partybus under the hood. 
    -https://github.com/chef/chef-server/blob/608dbe94d15822a31849952e13549744fc40a702/omnibus/files/private-chef-ctl-commands/upgrade.rb#L100
    - partybus has a config file on your chef server at `/opt/opscode/embedded/service/partybus/config.rb`
    - You need to change your `database_connection_string` (and possibly `database_unix_user`) for your external database before running a chef-server-ctl upgrade.
    - Unfortunately, as of this writing, **a chef-server-ctl reconfigure will overwrite this change**. I should totally make a pull request.
    - You can handle this in a cookbook recipe. I should totally write that.

## Needed Improvements
- Submit PR to Chef server to customize partybus database connection string without monkeypatching.
- Handle the Chef server configuration wtih a cookbook. Something something dogfood.
    - Create recipe to upgrade an external Chef server using method above.
- Improve autoscaling behavior of the designated boootstrap frontend.
- Continue to improve Cloudwatch alarms.
- Make an Automate cluster template powered by Backendless Chef stack.
- [Highly available NAT instances](http://www.cakesolutions.net/teamblogs/making-aws-nat-instances-highly-available-without-the-compromises)
- Integrate VPC support for ElasticSearch if/when Amazon implements it, or investigate matrix style permissions:
    - https://aws.amazon.com/blogs/security/how-to-control-access-to-your-amazon-elasticsearch-service-domain/

## Resources:
- https://github.com/irvingpop/cloudformation_automate
- http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-template-resource-type-ref.html
- https://docs.chef.io/server_components.html#external-postgresql

