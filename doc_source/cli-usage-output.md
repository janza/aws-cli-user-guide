# Controlling Command Output from the AWS CLI<a name="cli-usage-output"></a>

This section describes the different ways that you can control the output from the AWS Command Line Interface \(AWS CLI\)\.

**Topics**
+ [How to Select the Output Format](#cli-usage-output-format)
+ [JSON Output Format](#cli-usage-output-json)
+ [Text Output Format](#text-output)
+ [Table Output Format](#table-output)
+ [How to Filter the Output with the `--query` Option](#cli-usage-output-filter)

## How to Select the Output Format<a name="cli-usage-output-format"></a>

The AWS CLI supports three different output formats:
+ JSON \(`json`\)
+ Tab\-delimited text \(`text`\)
+ ASCII\-formatted table \(`table`\)

As explained in the [configuration](cli-chap-configure.md) topic, you can specify the output format in three ways:
+ Using the `output` option in a named profile in the `config` file\. The following example sets the default output format to `text`\.

  ```
  [default]
  output=text
  ```
+ Using the `AWS_DEFAULT_OUTPUT` environment variable\. The following output sets the format to `table` for the commands in this command\-line session until the variable is changed or the session ends\. Using this environment variable overrides any value set in the `config` file\.

  ```
  $ export AWS_DEFAULT_OUTPUT="table"
  ```
+ Using the `--output` option on the command line\. The following example sets the output of only this one command to `json`\. Using this option on the command overrides any currently set environment variable or the value in the `config` file\.

  ```
  $ aws swf list-domains --registration-status REGISTERED --output json
  ```

 [AWS CLI precedence rules](cli-chap-configure.md#config-settings-and-precedence) apply\. For example, using the `AWS_DEFAULT_OUTPUT` environment variable overrides any value set in the `config` file, and a value passed to an AWS CLI command with `--output` overrides any value set in the environment variable or in the `config` file\.

The `json` option is best for handling the output programmatically via various languages or `jq` \(a command\-line JSON processor\)\.

The `table` format is easy to read\.

The `text` format works well with traditional Unix text processing tools, such as `sed`, `grep`, and `awk`, as well as in PowerShell scripts\.

The results in any format can be customized and filtered by using the \-\-query parameter\. For more information, see [How to Filter the Output with the `--query` Option](#cli-usage-output-filter)\. 

## JSON Output Format<a name="cli-usage-output-json"></a>

JSON is the default output format of the AWS CLI\. Most languages can easily decode JSON strings using built\-in functions or with publicly available libraries\. As shown in the previous topic along with output examples, the `--query` option provides powerful ways to filter and format the AWS CLI's JSON\-formatted output\. 

If you need more advanced features that might not be possible with `--query`, you can check out `jq`, a command line JSON processor\. You can download it and find the official tutorial at [http://stedolan\.github\.io/jq/](http://stedolan.github.io/jq/)\.

## Text Output Format<a name="text-output"></a>

The *text* format organizes the AWS CLI's output into tab\-delimited lines\. It works well with traditional Unix text tools such as `grep`, `sed`, and `awk`, as well as the text processing performed by PowerShell\. 

The text output format follows the basic structure shown below\. The columns are sorted alphabetically by the corresponding key names of the underlying JSON object\.

```
IDENTIFIER  sorted-column1 sorted-column2
IDENTIFIER2 sorted-column1 sorted-column2
```

The following is an example of a text output\.

```
$ aws ec2 describe-volumes --output text
VOLUMES us-west-2a      2013-09-17T00:55:03.000Z        30      snap-f23ec1c8   in-use  vol-e11a5288    standard
ATTACHMENTS     2013-09-17T00:55:03.000Z        True    /dev/sda1       i-a071c394      attached        vol-e11a5288
VOLUMES us-west-2a      2013-09-18T20:26:15.000Z        8       snap-708e8348   in-use  vol-2e410a47    standard
ATTACHMENTS     2013-09-18T20:26:16.000Z        True    /dev/sda1       i-4b41a37c      attached        vol-2e410a47
```

**Important**  
*We strongly recommend that if you specify `text` output, you also always use the `--query` option to ensure consistent behavior*\. This is because the text format alphabetically orders output columns by the key name of the underlying JSON object, and similar resources might not have the same key names\. For example, the JSON representation of a Linux\-based EC2 instance might have elements that are not present in the JSON representation of a Windows\-based instance, or vice versa\. Also, resources might have key\-value elements added or removed in future updates, altering the column ordering\. This is where `--query` augments the functionality of the text output to provide you with complete control over the output format\. In the following example, the command specifies which elements to display and *defines the ordering* of the columns with the list notation `[key1, key2, ...]`\. This gives you full confidence that the correct key values are always displayed in the expected column\. Finally, notice how the AWS CLI outputs None as values for keys that don't exist\.

```
$ aws ec2 describe-volumes --query 'Volumes[*].[VolumeId, Attachments[0].InstanceId, AvailabilityZone, Size, FakeKey]' --output text
vol-e11a5288    i-a071c394      us-west-2a      30      None
vol-2e410a47    i-4b41a37c      us-west-2a      8       None
```

The following example show how you can use `grep` and `awk` with the text output from the `aws ec2 describe-instances` command\. The first command displays the Availability Zone, current state, and the instance ID of each instance in text output\. The second command processes that output to display only the instance IDs of all running instances in the `us-west-2a` Availability Zone\.

```
$ aws ec2 describe-instances --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, State.Name, InstanceId]' --output text
us-west-2a      running i-4b41a37c
us-west-2a      stopped i-a071c394
us-west-2b      stopped i-97a217a0
us-west-2a      running i-3045b007
us-west-2a      running i-6fc67758

$ aws ec2 describe-instances --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, State.Name, InstanceId]' --output text | grep us-west-2a | grep running | awk '{print $3}'
i-4b41a37c
i-3045b007
i-6fc67758
```

The following example goes a step further and shows not only how to filter the output, but how to use that output to automate changing instance types for each stopped instance\.

```
$ aws ec2 describe-instances --query 'Reservations[*].Instances[*].[State.Name, InstanceId]' --output text |
> grep stopped |
> awk '{print $2}' |
> while read line;
> do aws ec2 modify-instance-attribute --instance-id $line --instance-type '{"Value": "m1.medium"}';
> done
```

The text output can also be useful in PowerShell\. Because the columns in `text` output is tab\-delimited, it's easily split into an array by using PowerShell's ``t` delimiter\. The following command displays the value of the third column \(`InstanceId`\) if the first column \(`AvailabilityZone`\) matches the string `us-west-2a`\.

```
PS C:\>aws ec2 describe-instances --query 'Reservations[*].Instances[*].[Placement.AvailabilityZone, State.Name, InstanceId]' --output text |
%{if ($_.split("`t")[0] -match "us-west-2a") { $_.split("`t")[2]; } }
i-4b41a37c
i-a071c394
i-3045b007
i-6fc67758
```

**Tip**  
If you output text, and filter the output to a single field using the \-\-query parameter, the output is a single line of tab separated values\. To get each value onto a separate line, you can put the output field in brackets as shown in the following examples:

Tab separated, single\-line output:

```
$ aws iam list-groups-for-user --user-name susan  --output text --query "Groups[].GroupName"
HRDepartment    Developers      SpreadsheetUsers  LocalAdmins
```

Each value on its own line by putting `[GroupName]` in brackets:

```
$ aws iam list-groups-for-user --user-name susan  --output text --query "Groups[].[GroupName]"
HRDepartment
Developers
SpreadsheetUsers
LocalAdmins
```

## Table Output Format<a name="table-output"></a>

The `table` format produces human\-readable representations of complex AWS CLI output in a tabular form\.

```
$ aws ec2 describe-volumes --output table
---------------------------------------------------------------------------------------------------------------------
|                                                  DescribeVolumes                                                  |
+-------------------------------------------------------------------------------------------------------------------+
||                                                     Volumes                                                     ||
|+------------------+---------------------------+-------+----------------+---------+----------------+--------------+|
|| AvailabilityZone |        CreateTime         | Size  |  SnapshotId    |  State  |   VolumeId     | VolumeType   ||
|+------------------+---------------------------+-------+----------------+---------+----------------+--------------+|
||  us-west-2a      |  2013-09-17T00:55:03.000Z |  30   |  snap-f23ec1c8 |  in-use |  vol-e11a5288  |  standard    ||
|+------------------+---------------------------+-------+----------------+---------+----------------+--------------+|
|||                                                  Attachments                                                  |||
||+---------------------------+------------------------+-------------+--------------+------------+----------------+||
|||        AttachTime         |  DeleteOnTermination   |   Device    | InstanceId   |   State    |   VolumeId     |||
||+---------------------------+------------------------+-------------+--------------+------------+----------------+||
|||  2013-09-17T00:55:03.000Z |  True                  |  /dev/sda1  |  i-a071c394  |  attached  |  vol-e11a5288  |||
||+---------------------------+------------------------+-------------+--------------+------------+----------------+||
||                                                     Volumes                                                     ||
|+------------------+---------------------------+-------+----------------+---------+----------------+--------------+|
|| AvailabilityZone |        CreateTime         | Size  |  SnapshotId    |  State  |   VolumeId     | VolumeType   ||
|+------------------+---------------------------+-------+----------------+---------+----------------+--------------+|
||  us-west-2a      |  2013-09-18T20:26:15.000Z |  8    |  snap-708e8348 |  in-use |  vol-2e410a47  |  standard    ||
|+------------------+---------------------------+-------+----------------+---------+----------------+--------------+|
|||                                                  Attachments                                                  |||
||+---------------------------+------------------------+-------------+--------------+------------+----------------+||
|||        AttachTime         |  DeleteOnTermination   |   Device    | InstanceId   |   State    |   VolumeId     |||
||+---------------------------+------------------------+-------------+--------------+------------+----------------+||
|||  2013-09-18T20:26:16.000Z |  True                  |  /dev/sda1  |  i-4b41a37c  |  attached  |  vol-2e410a47  |||
||+---------------------------+------------------------+-------------+--------------+------------+----------------+||
```

You can combine the `--query` option with the table format to display a set of elements preselected from the raw output\. Notice the output differences between dictionary and list notations: column names are alphabetically ordered in the first example, and unnamed columns are ordered as defined by the user in the second example\. For more information about the `--query` option, see [How to Filter the Output with the `--query` Option](#cli-usage-output-filter)\.

```
$ aws ec2 describe-volumes --query 'Volumes[*].{ID:VolumeId,InstanceId:Attachments[0].InstanceId,AZ:AvailabilityZone,Size:Size}' --output table
------------------------------------------------------
|                   DescribeVolumes                  | 
+------------+----------------+--------------+-------+
|     AZ     |      ID        | InstanceId   | Size  |
+------------+----------------+--------------+-------+
|  us-west-2a|  vol-e11a5288  |  i-a071c394  |  30   |
|  us-west-2a|  vol-2e410a47  |  i-4b41a37c  |  8    |
+------------+----------------+--------------+-------+

$ aws ec2 describe-volumes --query 'Volumes[*].[VolumeId,Attachments[0].InstanceId,AvailabilityZone,Size]' --output table
----------------------------------------------------
|                  DescribeVolumes                 |
+--------------+--------------+--------------+-----+
|  vol-e11a5288|  i-a071c394  |  us-west-2a  |  30 |
|  vol-2e410a47|  i-4b41a37c  |  us-west-2a  |  8  |
+--------------+--------------+--------------+-----+
```

## How to Filter the Output with the `--query` Option<a name="cli-usage-output-filter"></a>

The AWS CLI provides built\-in JSON\-based output filtering capabilities with the `--query` option\. The \-\-query parameter accepts strings that are compliant with the [JMESPath specification](http://jmespath.org/)\. 

**Important**  
The output type you specify \(`json`, `text`, or `table`\) impacts how the `--query` option operates\.  
If you specify `--output text`, the output is paginated *before* the `--query` filter is applied and the AWS CLI runs the query once on *each page* of the output\. This can result in unexpected extra output, especially if your filter specifies an array element using something like \[0\], because the output then includes the first matching element on each page\.
If you specify `--output json`, the output is completely processed and converted into a JSON structure before the `--query` filter is applied\. The AWS CLI runs the query only once against the entire output\.
To work around the extra output that can be produced if you use `--output text`, you can specify `--no-paginate`\. This causes the filter to apply only to the complete set of results, but it does remove any pagination, so could result in long output\. You could also use other command line tools such as `head` or `tail` to additionally filter the output to only the values you want\. 

To demonstrate how `--query` works, we first start with the default JSON output below, which describes two Amazon Elastic Block Store \(Amazon EBS\) volumes attached to separate Amazon EC2 instances\.

```
$ aws ec2 describe-volumes
{
    "Volumes": [
        {
            "AvailabilityZone": "us-west-2a",
            "Attachments": [
                {
                    "AttachTime": "2013-09-17T00:55:03.000Z",
                    "InstanceId": "i-a071c394",
                    "VolumeId": "vol-e11a5288",
                    "State": "attached",
                    "DeleteOnTermination": true,
                    "Device": "/dev/sda1"
                }
            ],
            "VolumeType": "standard",
            "VolumeId": "vol-e11a5288",
            "State": "in-use",
            "SnapshotId": "snap-f23ec1c8",
            "CreateTime": "2013-09-17T00:55:03.000Z",
            "Size": 30
        },
        {
            "AvailabilityZone": "us-west-2a",
            "Attachments": [
                {
                    "AttachTime": "2013-09-18T20:26:16.000Z",
                    "InstanceId": "i-4b41a37c",
                    "VolumeId": "vol-2e410a47",
                    "State": "attached",
                    "DeleteOnTermination": true,
                    "Device": "/dev/sda1"
                }
            ],
            "VolumeType": "standard",
            "VolumeId": "vol-2e410a47",
            "State": "in-use",
            "SnapshotId": "snap-708e8348",
            "CreateTime": "2013-09-18T20:26:15.000Z",
            "Size": 8
        }
    ]
}
```

We can choose to display only the first volume from the `Volumes` list by using the following command that [indexes the first volume in the array](http://jmespath.org/specification.html#index-expressions)\.

```
$ aws ec2 describe-volumes --query 'Volumes[0]'
{
    "AvailabilityZone": "us-west-2a",
    "Attachments": [
        {
            "AttachTime": "2013-09-17T00:55:03.000Z",
            "InstanceId": "i-a071c394",
            "VolumeId": "vol-e11a5288",
            "State": "attached",
            "DeleteOnTermination": true,
            "Device": "/dev/sda1"
        }
    ],
    "VolumeType": "standard",
    "VolumeId": "vol-e11a5288",
    "State": "in-use",
    "SnapshotId": "snap-f23ec1c8",
    "CreateTime": "2013-09-17T00:55:03.000Z",
    "Size": 30
}
```

In the next example, we use the [wildcard notation `[*]`](http://jmespath.org/specification.html#wildcard-expressions) to iterate over all of the volumes in the list and also filter out three elements from each: `VolumeId`, `AvailabilityZone`, and `Size`\. The dictionary notation requires that you provide an alias for each JSON key, like this: \{Alias1:JSONKey1,Alias2:JSONKey2\}\. A dictionary is inherently *unordered*, so the ordering of the key\-aliases within a structure might be inconsistent\.

```
$ aws ec2 describe-volumes --query 'Volumes[*].{ID:VolumeId,AZ:AvailabilityZone,Size:Size}'
[
    {
        "AZ": "us-west-2a",
        "ID": "vol-e11a5288",
        "Size": 30
    },
    {
        "AZ": "us-west-2a",
        "ID": "vol-2e410a47",
        "Size": 8
    }
]
```

Using dictionary notation, you can also chain keys together, like `key1.key2[0].key3`, to filter elements deeply nested within the structure\. The following example demonstrates this with the `Attachments[0].InstanceId` key, aliased to simply `InstanceId`\.

```
$ aws ec2 describe-volumes --query 'Volumes[*].{ID:VolumeId,InstanceId:Attachments[0].InstanceId,AZ:AvailabilityZone,Size:Size}'
[
    {
        "InstanceId": "i-a071c394",
        "AZ": "us-west-2a",
        "ID": "vol-e11a5288",
        "Size": 30
    },
    {
        "InstanceId": "i-4b41a37c",
        "AZ": "us-west-2a",
        "ID": "vol-2e410a47",
        "Size": 8
    }
]
```

You can also filter multiple elements using list notation: `[key1, key2]`\. This formats all filtered attributes into a single *ordered* list per object, regardless of type\.

```
$ aws ec2 describe-volumes --query 'Volumes[*].[VolumeId, Attachments[0].InstanceId, AvailabilityZone, Size]'
[
    [
        "vol-e11a5288",
        "i-a071c394",
        "us-west-2a",
        30
    ],
    [
        "vol-2e410a47",
        "i-4b41a37c",
        "us-west-2a",
        8
    ]
]
```

To filter results by the value of a specific field, use the [JMESPath "?" operator](http://jmespath.org/specification.html#filter-expressions)\. The following example query outputs only volumes in the `us-west-2a` Availability Zone\.

```
$ aws ec2 describe-volumes --query 'Volumes[?AvailabilityZone==`us-west-2a`]'
```

**Note**  
When specifying a literal value such as "us\-west\-2" above in a JMESPath query expression, you must surround the value in backticks \(` `\) for it to be read properly\.

Here are some additional examples that illustrate how you can get only the details you want from the output of your commands\.

The following example lists Amazon EC2 volumes\. The service produces a list of all in\-use volumes in the `us-west-2a` Availability Zone\. The `--query` parameter further limits the output to only those volumes with a `Size` value that is larger than 50, and shows only the specified fields with user\-defined names\.

```
$ aws ec2 describe-volumes \
--filter "Name=availability-zone,Values=us-west-2a" "Name=status,Values=attached" \
--query 'Volumes[?Size > `50`].{Id:VolumeId,Size:Size,Type:VolumeType}'
[
    {
        "Id": "vol-0be9bb0bf12345678",
        "Size": 80,
        "Type": "gp2"
    }
]
```

The following example retrieves a list of images that meet several criteria\. It then uses the `--query` parameter to sort the output by `CreationDate`, selecting only the most recent\. It then displays the `ImageId` of that one image\.

```
$ aws ec2 describe-images \
--owners amazon \
--filters "Name=name,Values=amzn*gp2" "Name=virtualization-type,Values=hvm" "Name=root-device-type,Values=ebs" \
--query "sort_by(Images, &CreationDate)[-1].ImageId" \
--output text
ami-00ced3122871a4921
```

The following example uses the `--query` parameter to find a specific item in a list and then extracts information from that item\. The example lists all of the availability zones associated with the specified service endpoint\. It extracts the item from the `ServiceDetails` list that has the specified `ServiceName`, then outputs the `AvailabilityZones` field from that selected item\. 

```
$ aws --region us-east-1 ec2 describe-vpc-endpoint-services --query 'ServiceDetails[?ServiceName==`com.amazonaws.us-east-1.ecs`].AvailabilityZones'
[
    [
        "us-east-1a",
        "us-east-1b",
        "us-east-1c",
        "us-east-1d",
        "us-east-1e",
        "us-east-1f"
    ]
]
```

The `--query` parameter also enables you to count items in the output\. The following example displays the number of available volumes that are more than 1000 IOPS\.

```
$ aws ec2 describe-volumes \
--filter "Name=status,Values=available" \
--query 'length(Volumes[?Iops > `1000`])'
3
```

The following example shows how to list all of your snapshots that were created after a specified date, including only a few of the available fields in the output\.

```
$ aws ec2 describe-snapshots --owner self --output json \
--query 'Snapshots[?StartTime>=`2018-02-07`].{Id:SnapshotId,VId:VolumeId,Size:VolumeSize}' \
[
    {
        "id": "snap-0effb42b7a1b2c3d4",
        "vid": "vol-0be9bb0bf12345678",
        "Size": 8
    }
]
```

The following example lists the five most recent AMIs that you created, sorted from most recent to oldest\.

```
$ aws ec2 describe-images --owners self \
--query 'reverse(sort_by(Images,&CreationDate))[:5].{id:ImageId,date:CreationDate}'
[
    {
        "id": "ami-0a1b2c3d4e5f60001",
        "date": "2018-11-28T17:16:38.000Z"
    },
    {
        "id": "ami-0a1b2c3d4e5f60002",
        "date": "2018-09-15T13:51:22.000Z"
    },
    {
        "id": "ami-0a1b2c3d4e5f60003",
        "date": "2018-08-19T10:22:45.000Z"
    },
    {
        "id": "ami-0a1b2c3d4e5f60004",
        "date": "2018-05-03T12:04:02.000Z"
    },
    {
        "id": "ami-0a1b2c3d4e5f60005",
        "date": "2017-12-13T17:16:38.000Z"
    }

]
```

This following example shows only the `InstanceId` for any unhealthy instances in the specified AutoScaling Group\.

```
$ aws autoscaling describe-auto-scaling-groups --auto-scaling-group-name My-AutoScaling-Group-Name --output text\
--query 'AutoScalingGroups[*].Instances[?HealthStatus==`Unhealthy`].InstanceId'
```

Combined with the three output formats that are explained in more detail in the following sections, the `--query` option is a powerful tool you can use to customize the content and style of outputs\. 

For more examples and the full spec of JMESPath, the underlying JSON\-processing library, see [http://jmespath\.org/specification\.html](http://jmespath.org/specification.html)\.