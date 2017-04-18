# plygrnd-net-deployment-ghost
CloudFormation template and related resources for plygrnd.net

## Why am I doing this?

I recently tried to launch [Ghost](https://ghost.org) on AWS, and it was a complete pain in the arse trying to set it up manually. Ghost, while far easier to configure than say Wordpress, has a few idiosyncrasies that complicate running a production instance scalably. This resulted in me jumping 15 steps ahead (as always) and writing production content on a development instance of Ghost. 

## OK. Wat do?

I'll launch a production-ready instance on Amazon Web Services and ensure it stays up. Then I'll blog about it so I don't forget how I did it.

## Technologies used

* VPC with four subnets:
    * 2 public subnets (10.0.1.0/24, 10.0.2.0/24) in two different Availability Zones (AZs).
    * 2 private subnets (10.0.3.0/24, 10.0.4.0/24) in different AZs
* EC2:
    * 1 Autoscaling group with minimum 2, maximum 4 instances running
    * 1 Elastic Load Balancer to terminate SSL and serve web traffic
    * EC2 application servers running Ghost
* RDS:
    * 1 multi-AZ MySQL database
* SES (email campaigns if I ever write decent content, and password resets)
* CloudFormation
* AWS Certificate Manager
* Route 53

I was going to use AWS Config to check that my resources were configured sanely, but the price tag ($2 per rule) put me off.

## Software setup

Ghost requires a number of variables to be set before it can be used. The basic config file is described [here](http://support.ghost.org/config/); it requires a whole lot of passwords to be stored in it. Since I'm going to bake the configuration into an AMI, I certainly don't want to include any plaintext passwords or API access keys in it!

We have two options here: use [CloudFormation parameters](http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html#creds) to pass creds to the config file, or use a Lambda function to generate the creds, encrypt them with KMS and store them in a DynamoDB table. We could also use something like [Vault](https://vaultproject.io) to store the creds, but that's even more overhead. For right now, I'll generate the database password in Lastpass and pass it to the stack as a parameter.

So, our completed config.js file with all the privacy bits 'n bobs turned on looks like this:

```
var path = require('path'),
    config;

config = {
    production: {
        url: 'myurl',
        mail: {
            transport: 'SMTP',
            options:{
                host: 'email-smtp.eu-west-1.amazonaws.com',
                port: '465',
                service: 'SES',
                auth: {
                    user: 'SESUSERID',
                    pass: 'SESUSERSECRET'
                }
            }
        },
        database:  {
            client: 'mysql',
            connection: {
                host     : 'rdsdbhostname',
                user     : 'rdsusername',
                password : 'rdspassword',
                database : 'ghost',
                charset  : 'utf8'
            }
        },
        privacy:{
            useGravatar: false,
            useUpdatecheck: false,
            useRpcPing: false
        } 
    }
};

module.exports = config;
```

We'll need to fire that into a fleet of worker bees (EC2 instances) running Ghost, which we'll need to install. [This blog post](https://aws.amazon.com/blogs/devops/authenticated-file-downloads-with-cloudformation/) has information on how to download files during template execution.

## Build steps

1. Create a VPC named 'ghost'.
2. Create subnets:
```
|public-1|10.0.1.0/24|eu-west-1a| 
|public-2|10.0.2.0/24|eu-west-1c| 
|private-1|10.0.3.0/24|eu-west-1a| 
|private-2|10.0.4.0/24|eu-west-1b| 
```
3. Create security groups:
    1. public: TCP 80/443 open to internet. Apply to public-1 and public-2.
    2. private: TCP 80 open to 10.0.1.0/24 and 10.0.2.0/24. TCP 3306 open to 10.0.3.0/24 and 10.0.4.0/24. SSH open to my current IP. Apply to private-1 and private-2.
4. Create autoscaling launch configuration:
    1. Amazon Linux ami-e5083683
    2. t2.micro
    3. 8GB EBS storage
    4. Security groups: private
6. Create Ghost configuration file and populate with following values:
    1. SES user ID
    2. SES secret
    3. RDS username
    4. RDS password
    5. URL
5. Create autoscaling group:
    1. Minimum 2 instances
    2. Maximum 4
5. Create ELB:
    1. Listens on port 80 and 443
    1. Terminates SSL using ACM cert
6. Create RDS database:
    1. db.t2.micro
    2. MySQL
    3. Multi-AZ
    4. Security group: private
7. Launch EC2 instances into autoscaling group
8. Install node and npm
9. Install nginx
10. Load Ghost configuration
11. npm install --production
12. npm start --production
13. Update Route 53 hosted zone with alias record pointing to ELB
    