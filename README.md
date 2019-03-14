# athena-sg-analyser

```sql
CREATE DATABASE IF NOT EXISTS cloudtraildb
    COMMENT 'CloudTrail Database'
    LOCATION 's3://cloudtrails3bucket123/AWSLogs'
    WITH DBPROPERTIES ('Creator'='Jeremy Cook', 'Company'='CloudAcademy', 'Created'='2019')
```

```sql
CREATE EXTERNAL TABLE cloudtraildb.cloudtrail_logs (
  eventversion STRING,
  useridentity STRUCT<type:STRING,principalid:STRING,arn:STRING,accountid:STRING,invokedby:STRING,accesskeyid:STRING,username:STRING,sessioncontext:struct<attributes:STRUCT<mfaauthenticated:STRING,creationdate:STRING>,sessionissuer:struct<type:STRING,principalid:STRING,arn:STRING,accountid:STRING,username:STRING>>>,
  eventtime STRING,
  eventsource STRING,
  eventname STRING,
  awsregion STRING,
  sourceipaddress STRING,
  useragent STRING,
  errorcode STRING,
  errormessage STRING,
  requestparameters STRING,
  responseelements STRING,
  additionaleventdata STRING,
  requestid STRING,
  eventid STRING,
  resources ARRAY<STRUCT<ARN:STRING,accountid:STRING,type:STRING>>,
  eventtype STRING,
  apiversion STRING,
  readonly STRING,
  recipientaccountid STRING,
  serviceeventdetails STRING,
  sharedeventid STRING,
  vpcendpointid STRING)
ROW FORMAT SERDE 
  'com.amazon.emr.hive.serde.CloudTrailSerde' 
STORED AS INPUTFORMAT 
  'com.amazon.emr.cloudtrail.CloudTrailInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  's3://cloudtrails3bucket123/AWSLogs'
```

```sql
SELECT * FROM cloudtraildb.cloudtrail_logs limit 10;
```

```sql
SELECT eventname,
        useridentity.username,
        sourceipaddress,
        eventtime,
        requestparameters
FROM cloudtraildb.cloudtrail_logs
```

```sql
SELECT eventname,
        useridentity.username,
        sourceipaddress,
        eventtime,
        requestparameters
FROM cloudtraildb.cloudtrail_logs
WHERE (requestparameters LIKE '%sg-08f82acf7f206bd01%')
ORDER BY eventtime ASC
```

```sql
SELECT eventname,
        useridentity.username,
        sourceipaddress,
        eventtime,
        requestparameters
FROM cloudtraildb.cloudtrail_logs
WHERE (requestparameters LIKE '%sg-08f82acf7f206bd01%')
AND eventname = 'AuthorizeSecurityGroupIngress'
ORDER BY eventtime ASC
```

Note: requestparameters in previous query is JSON and when formattted becomes:

```json
{
  "groupId": "sg-08f82acf7f206bd01",
  "ipPermissions": {
    "items": [
      {
        "ipProtocol": "tcp",
        "fromPort": 1000,
        "toPort": 1000,
        "groups": {},
        "ipRanges": {
          "items": [
            {
              "cidrIp": "1.1.1.1/32"
            }
          ]
        },
        "ipv6Ranges": {},
        "prefixListIds": {}
      },
      {
        "ipProtocol": "tcp",
        "fromPort": 2000,
        "toPort": 2000,
        "groups": {},
        "ipRanges": {
          "items": [
            {
              "cidrIp": "2.2.2.2/32"
            }
          ]
        },
        "ipv6Ranges": {},
        "prefixListIds": {}
      }
    ]
  }
}
```

Let's now query and extract attributes of interest from the JSON requestparameters data:

```sql
SELECT json_extract_scalar(requestparameters, '$.groupId') AS sg_id,
       json_extract_scalar(i.item,'$.ipProtocol') AS ipProtocol,
       json_extract_scalar(i.item,'$.fromPort') AS fromPort,
       json_extract_scalar(i.item,'$.toPort') AS toPort,
       json_extract_scalar(i.item,'$.ipRanges.items[0].cidrIp') AS cidrIp
FROM cloudtraildb.cloudtrail_logs
    CROSS JOIN UNNEST (CAST(json_extract(requestparameters,'$.ipPermissions.items') 
        AS ARRAY(JSON))) AS i (item)
WHERE (requestparameters LIKE '%sg-08f82acf7f206bd01%')
AND eventname = 'AuthorizeSecurityGroupIngress'
```

```sql
DROP TABLE cloudtrail_logs
```

```sql
DROP DATABASE cloudtraildb
```
