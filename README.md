# s3-parallel-s3-versions-delete
===

Quick and dirty script to delete s3 object versions in parallel with.

I was able to delete ~1mi docs in 6.5min from an EC2 instance.

```
import boto3, multiprocessing
client = boto3.client("s3")

bucket = 'bucket-name-goes-here'

def consumer(versions):
    delete_response = client.delete_objects(
        Bucket=bucket,
        Delete={
            'Objects': [
                {
                    'Key': v[0],
                    'VersionId': v[1]
                } for v in versions
            ],
            'Quiet': True
        }
    )
    print('response', delete_response['ResponseMetadata']['HTTPStatusCode'], 'docs deleted', len(versions))

if __name__ == '__main__':
    next_key = None
    next_version_id = None
    with multiprocessing.Pool(40) as pool:
        while True:
            if not next_key:
                response = client.list_object_versions(
                    Bucket=bucket,
                    MaxKeys=1000
                )
            else:
                response = client.list_object_versions(
                    Bucket=bucket,
                    MaxKeys=1000,
                    KeyMarker=next_key,
                    VersionIdMarker=next_version_id
                )
            next_key = response['NextKeyMarker']
            next_version_id = response['NextVersionIdMarker']
            deletes = []
            versions = []
            if 'DeleteMarkers' in response:
                deletes = [ (v['Key'], v['VersionId']) for v in response['DeleteMarkers'] ]
            if 'Versions' in response:
                versions = [ (v['Key'], v['VersionId']) for v in response['Versions'] ]
            items = deletes + versions
            pool.apply_async(consumer, (items,))
```
