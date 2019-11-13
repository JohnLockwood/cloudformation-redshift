
# cloudformation-redshift

The file ```redshift.yaml``` is a Redshift Cloudformation Template that I use to easily set up and tear down a cluster for testing and proof of concept purposes. 

## Usage

If you have the AWS cli [installed](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html), you can use the instructions below.  If not, a fairly frictionless way to get started is to simply create an [Amazon Cloud9](https://aws.amazon.com/cloud9/) instance in the region where you want to the cluster to live.  The environment this creates will have a aws command line interface all set up for you, which you can verify with:

```
aws configure list
```

### Creating the stack

I generally use a shell script or batch file to do this.  **IMPORTANT**:  You should add an entry to .gitignore to exclude this file so you don't check it in, because it contains your password.  This is acceptable for personal development purposes.  In a production environment a better approach would be to use AWS secrets or Key Management Service.

I create the script in the same directory as the redshift file, with the contents below.  You replace "redshift2019", "MyAdminUserName", and "MyAdminPassword" with appropriately secure values.  AA general best practice for passwords is to use special characters, but what I found is that they cause problems you try to connect via JDBC.  So it's better in this case to use a longer AlphaNumeric password to make it more secure, not special characters.

```
aws cloudformation create-stack --stack-name redshift2019 --template-body file://./redshift.yaml --parameters ParameterKey=MasterUsername,ParameterValue=MyAdminUserName ParameterKey=MasterUserPassword,ParameterValue=MyAdminPassword
```

### Watching it spin up

I like to watch the stack get created both in the Redshift section of the console, but you can also check on the progress using the command below, replacing the stack name as appropriate.

```
aws cloudformation describe-stacks --stack-name=redshift2019 
```

### Deleting the stack

Delete the stack with the following command.  See the "Source and Modifications" section below for how to do this to save money while being able to restore the stack later.

```
aws cloudformation delete-stack --stack-name=redshift2019 
```

## Source and Modifications

The redshift.yaml file is based on [this Amazon Sample](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/quickref-redshift.html).  Like the original, the current script both creates a VPC but then wires up the security group, route etc to make the cluster available generally on the Internet.  Again, in a production environment it could be further locked down.

I've modified the original sample script as follows:

* I've added dc2.large as the default node type, to make a cluster that's eligible to take advantage of the new [Redshift Query Editor](https://docs.aws.amazon.com/redshift/latest/mgmt/query-editor.html).  However, the result of that effort was a bit of a disappointment, because the online query editor doesn't support executing more than one query at a time (and if you're setting ).  A better all-around choice was to use a JDBC client -- the one that worked best for me was also a free choice - [Squirrel](http://squirrel-sql.sourceforge.net/).
* This script creates a multi-node cluster using 2 dc2.large instances.
* I added a DeletionPolicy of Snapshot to the Redshift Cluster. This means that when you delete the Cloudformation Stack, a snapshot is created first.  To take advantage of that snapshot and a, I also added a "SnapshotName" parameter.  Set this to "NONE" when you first use the script, and it will create a new cluster not based on a snapshot. After that, when you delete the CloudFormation stack as shown under "Deleting the Stack", above, you can set SnapshotName to the identifier of the snapshot that gets created when the stack is deleted. In this way you can set up the cluster but only pay for it while you're working on it.
* I've created an IAM role, RedshiftReadAccessS3, based on the AWS Managed Policy, AmazonS3ReadOnlyAccess.  This role is then assigned to the cluster, so as soon as it comes up you can for example run copy queries from delimited files in an S3 bucket.