# **AWS Lambda EBS Snapshot Cleanup**

## **1. Project Overview**

This AWS Lambda function automates the cleanup of **stale EBS snapshots** to reduce unnecessary storage costs. It identifies and deletes snapshots that:

- Are **not attached to any volume**.
- Are associated with a volume **that is not attached to any running EC2 instance**.
- Are associated with a **deleted volume**.

## **2. Architecture & Workflow**

1. **Lambda Function Execution**:
   - The Lambda function is triggered manually or through AWS EventBridge.
2. **Fetching EBS Snapshots**:
   - The function retrieves all EBS snapshots owned by the AWS account.
3. **Identifying Active EC2 Instances**:
   - It fetches all running EC2 instances and stores their IDs.
4. **Filtering Stale Snapshots**:
   - The function iterates through snapshots and checks:
     - If the snapshot has no associated volume → **Delete it**.
     - If the associated volume exists but is **not attached to any instance** → **Delete it**.
     - If the associated volume does **not exist** anymore → **Delete it**.
5. **Snapshot Deletion**:
   - The function deletes the identified stale snapshots and logs the deletions.

## **3. AWS Resources Used**

- **AWS Lambda**: Runs the snapshot cleanup logic.
- **AWS IAM Role & Permissions**: Grants necessary access to EC2 resources.
- **Amazon EC2 API**: Used to fetch instance and snapshot details.

## **4. IAM Role & Permissions**

To allow Lambda to interact with EC2 resources, an IAM role was created and attached to the function. The role includes the following permissions:

```json
{
    "Effect": "Allow",
    "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeVolumes",
        "ec2:DeleteSnapshot"
    ],
    "Resource": "*"
}
```

## **5. Lambda Function Code**

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()
    
    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Iterate through each snapshot and delete stale ones
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')

        if not volume_id:
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")
```

## **6. Execution & Testing**

1. **Created an EBS Volume** in AWS Console.

![Screenshot from 2025-03-07 20-04-48](https://github.com/user-attachments/assets/eac9d6a8-94fd-4080-b025-5a661a5e0728)

2. **Took a snapshot** of the volume.

![Screenshot from 2025-03-07 20-05-08](https://github.com/user-attachments/assets/635ca3bd-e392-45fa-8c50-2c4081cbce8f)

3. **Executed the Lambda function** manually.

![Screenshot from 2025-03-07 20-06-09](https://github.com/user-attachments/assets/d3de3435-1811-4eab-9292-608a1e5b82f3)

4. **Verified deletion** in the AWS Console.

![Screenshot from 2025-03-07 20-06-21](https://github.com/user-attachments/assets/a52d52b3-8c83-4b84-ad5d-61363d99b515)

5. **Tested multiple scenarios**, including snapshots from running instances and deleted volumes.
6. **Captured screenshots** of each step.

![Screenshot from 2025-03-07 20-00-48](https://github.com/user-attachments/assets/0205a5dd-249e-497b-a05b-8fe199ecacd2)

## **7. Results & Observations**

✅ Successfully deleted snapshots **not attached** to any volume.
✅ Cleaned up snapshots **from deleted volumes**.
✅ Avoided deletion of snapshots from **active instances**.
✅ Reduced **unnecessary storage costs**.

![Screenshot from 2025-03-07 20-06-21](https://github.com/user-attachments/assets/e5e9488d-8a06-4794-9e4f-3c0e8fb33d03)


## **8. Future Enhancements**

- **Scheduled Execution**: Use AWS EventBridge to trigger Lambda periodically.
- **Logging & Monitoring**: Store logs in CloudWatch for better tracking.
- **Email Notifications**: Integrate with SNS to notify administrators.

---

This documentation provides a complete overview of my project, ensuring clarity and easy reference. 


