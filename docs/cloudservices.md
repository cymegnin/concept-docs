## Cloud Services
Defined by the `config.rb` file.

The `config.rb` file resides in the `<repository-dir>/services` directory and exists to define interactions within the cloud provider in which you are operating. The [API Documentation](http://docs.cloudcoreo.com/docs/frames/index) provides specifics.

We define a ***resource*** as the actual cloud service you are interacting with, such as a [*Route53*](https://aws.amazon.com/route53/) entry or [*Autoscaling Group*](https://aws.amazon.com/autoscaling/). Resources are defined by the desired configuration within your cloud provider account.
In general, the structure is:
```
<resource> "<name>" do
    <property1> <value>
    <property2> <value>
    <property3> <value>
    <property4> <value>
end
```
For Example:
```
coreo_aws_vpc_vpc "my-vpc" do
  action :sustain
  cidr "12.0.0.0/16"
  internet_gateway true
end
```
In the above example, CloudCoreo would create a VPC with the CIDR address of "12.0.0.0/16" and attach an *Internet gateway* to the VPC to allow traffic to the public subnets (which haven't been defined in this specific example).

This is all very inflexible in terms of cloning, which is why ***templating*** has been built into the system.

### Templating System
Templating allows variables to be inserted into the system at runtime, thus allowing a single template to be used in many different environments. The substitution comes from one of:

1. The value provided by the user in the CloudCoreo UI
1. The values set during the run of the CLI tool
1. The default set in the config.yaml file

This is the order in which they are applied. e.g. If there is no UI or CLI variable set, CloudCoreo will apply the default.

The look of the variables is similar to *bash* but many of the bash-isms to not apply. Use a variable as follows `${VARIABLE_NAME}`. For instance, the above example of creating a VPC would be more useful with the following templating.
```
coreo_aws_vpc_vpc "${VPC_NAME}" do
  action :sustain
  cidr "${VPC_CIDR}"
  internet_gateway true
end
```
Within your config.yaml file, you can now call out a `VPC_NAME` variable and apply defaults. This variable name will show up in the web UI for any users to modify.
### Overriding Resources
The `services/` directory honors the hierarchy we have discussed elsewhere. The difference, however, is that in the `config.rb` file each ***resource*** is over-ridable on an individual basis. This is best explained with an example.

Lets say we have the following directory structure and files:

* `<repository-dir>/extends/services/config.rb`
and
* `<repository-dir>/services/config.rb`

We know from that anything in the `extends/services` directory will be run ***before*** the root `services` directory. This is true, however the resources are compiled before running. What does this mean?

**Any *resource* in the root level `services/config.rb` file will override any *resource* located in the `extends/services/config.rb` file, where the key is the `<resource> "<name>" do` line.** This still adheres to our previous rule we have been discussing in an interesting way. Since the resources are effectively keys, as we get closer and close to the root, the keys are added to the compilation stack in order, thus overwriting (overriding) the previously added resources. 

Once again, an example will work much better for this description.

Take, for instance, the directory and file structure we have already described:

* `<repository-dir>/extends/services/config.rb`
and
* `<repository-dir>/services/config.rb`

Lets assume we have a ***VPC*** resource in the `extends/services/config.rb` file as follows:
```
coreo_aws_vpc_vpc "${VPC_NAME}" do
  action :sustain
  cidr "12.0.0.0/16"
  internet_gateway true
end
```
This doesn't do us any good because we can't clone this to different environments. We need to have templating and variables, and we want to add some tags to the VPC. We can override this resource by keying off the `<resource> "<name>" do` line and placing the new block in the root `services/config.rb` file. So, within the `<repository-dir>/services/config.rb` file we call out a new ***VPC*** as follows:
```
coreo_aws_vpc_vpc "${VPC_NAME}" do
  action :sustain
  cidr "${VPC_CIDR}"
  internet_gateway true
  tags [
     "Name=${VPC_NAME_TAG}",
     "Team=${VPC_TEAM}"
  ]
end
```
The point of this is that the *key* is the first line, in this case `coreo_aws_vpc_vpc "${VPC_NAME}" do`. This means that as CloudCoreo is compiling the resources, it will see the ***VPC*** resource in the root ***last*** and will replace any existing blocks with this same resource key. The important part is that CloudCoreo will honor the positioning of the resources that it is overriding and simply override the block. We call this ***anchoring***.

### Anchoring
Anchoring is what CloudCoreo does with overridden resources. The key (`<resource> "<name>" do`) that is being overridden becomes an anchor for all pre-requisite resources found in an overriding file. That's a mouthful. Here's an abstract example. Please do not pay attention to the blocks because they are irrelevant and shortened for clarity.

Begin again with our standard directory structure we have been using:

* `<repository-dir>/extends/services/config.rb`
and
* `<repository-dir>/services/config.rb`

In our `extends/services/config.rb` file we will have a ***subnet definition*** and an ***elastic load balancer***.
```
coreo_aws_vpc_subnet 'public-subnet' do
   vpc 'my-vpc'
   map_public_ip_on_launch true
end

coreo_aws_ec2_elb 'my-elb' do
   type 'public'
   subnet 'public-subnet'
end
```
We decide this is not adequate. First of all we want the elb to contain a security group and we want it to be an `internal` load balancer and not public. To achieve this we move to our root `services/config.rb` file and add our key resource. This will become an anchor and we can modify anything other than the key itself.
```
coreo_aws_ec2_elb 'my-elb' do
   type 'internal'
   subnet 'private-subnet'
   security_groups ['my-elb-sg']
end
```
This one is modified correctly, but we are missing some pieces, namely the private subnet and the security groups. We re-edit our file knowing our anchor bring the pieces along that it requires. Our new root-level `services/config.rb` file looks as follows.
```
coreo_aws_vpc_subnet 'private-subnet' do
   vpc 'my-vpc'
   map_public_ip_on_launch false
end

coreo_aws_ec2_securityGroup 'my-elb-sg' do
   vpc 'my-vpc'
   allows [ 
            { 
              :direction => :ingress,
              :protocol => :tcp,
              :ports => ${INGRESS_PORTS},
              :cidrs => ${INGRESS_CIDRS}
            }
          ]
end

coreo_aws_ec2_elb 'my-elb' do
   type 'internal'
   subnet 'private-subnet'
   security_groups ['my-elb-sg']
end
```

The ***anchor*** in this is the elb resource. What this means is that anything above that has not already overridden something will be considered a prerequisite of the resource doing the overriding (the key). This means that the root level `services/config.rb` file's coreo_aws_ec2_elb resource block will override the one found in the `extends/services/config.rb` file. Not only that, but all the resources above the key in the root level will maintain order and override with the key. The final compilation will be a merge of the two files with the *anchor* located in the same place as the original, with any other non-overriding resources preceding it from the overriding file. Here is what the compiled version would become.
```
coreo_aws_vpc_subnet 'public-subnet' do
   vpc 'my-vpc'
   map_public_ip_on_launch true
end

coreo_aws_vpc_subnet 'private-subnet' do
   vpc 'my-vpc'
   map_public_ip_on_launch false
end

coreo_aws_ec2_securityGroup 'my-elb-sg' do
   vpc 'my-vpc'
   allows [ 
            { 
              :direction => :ingress,
              :protocol => :tcp,
              :ports => ${INGRESS_PORTS},
              :cidrs => ${INGRESS_CIDRS}
            }
          ]
end

coreo_aws_ec2_elb 'my-elb' do
   type 'internal'
   subnet 'private-subnet'
   security_groups ['my-elb-sg']
end
```
The '*public-subnet*' resource has been included from the `extends/services/config.rb` file, while the '*my-elb*' from the `extends/services/config.rb` essentially *becomes* the '*private-subnet*' + '*my-elb-sg*' + '*my-elb*'.

### API Docs
If you are working on services within CloudCoreo, you need to know what options, parameters etc you can use. For those head over to our [API Documentation](http://docs.cloudcoreo.com/docs/frames/index).
