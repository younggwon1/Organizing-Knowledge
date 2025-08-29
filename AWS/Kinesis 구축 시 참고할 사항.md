# Kinesis 구축 시 참고할 사항

### 상황

> Kinesis data stream -> Kinesis data firehose -> external accounts s3 파이프라인에 대한 구축 시 명령어 정리
>
> (콘솔에서는 firehose 설정 중 외부 accounts의 s3 설정이 안되어 cli 로만 가능)
```
$ aws firehose create-delivery-stream --delivery-stream-name {kinesis-data-firehose name} --delivery-stream-type KinesisStreamAsSource --kinesis-stream-source-configuration KinesisStreamARN=arn:aws:kinesis:ap-northeast-2:123456789:stream/{kinesis-data-stream name},RoleARN=arn:aws:iam::123456789:role/{kinesis-data-firehose name} --extended-s3-destination-configuration RoleARN=arn:aws:iam::123456789:role/{kinesis-data-firehose name},BucketARN=arn:aws:s3:::{s3 name}

$ aws firehose describe-delivery-stream --delivery-stream-name {kinesis-data-firehose name}

$ aws firehose update-destination --delivery-stream-name {kinesis-data-firehose name} --current-delivery-stream-version-id 1 --destination-id destinationId-000000000001  --cli-input-json file://firehose.json
```

**firehose.json**

```json
{
    "DeliveryStreamName": "{kinesis-data-firehose name}",
    "ExtendedS3DestinationUpdate": {
        "RoleARN": "arn:aws:iam::123456789:role/{kinesis-data-firehose name}",
        "BucketARN": "arn:aws:s3:::{s3 name}",
        "Prefix": "/prod/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
        "CustomTimeZone": "Asia/Seoul",
        "ErrorOutputPrefix": "/prod/error/!{firehose:error-output-type}/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/",
        "BufferingHints": {
            "SizeInMBs": 5,
            "IntervalInSeconds": 300
        },
        "CompressionFormat": "UNCOMPRESSED",
        "EncryptionConfiguration": {
            "NoEncryptionConfig": "NoEncryption"
        },
        "CloudWatchLoggingOptions": {
            "Enabled": false
        },
        "S3BackupMode": "Disabled",
        "DataFormatConversionConfiguration": {
            "Enabled": false
        }
    }
}
```

**참고 문서**
- [ExtendedS3DestinationUpdate](https://docs.aws.amazon.com/firehose/latest/APIReference/API_ExtendedS3DestinationUpdate.html)
