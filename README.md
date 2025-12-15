
# Guidance for rolling back changes to datasets in Amazon S3

#### Recovery of data in Amazon S3, at scale, using S3 Versioning.

>  **Disclaimer:** Code is provided as-is, to demonstrate a concept or workflow to AWS customers. You should ensure it meets your requirements, and carefully review the S3 Batch Operations manifests and tasks before running any jobs against non-test data.

## Table of Contents

- [Overview](#overview)
  - [Cost](#cost)
- [Prerequisites](#prerequisites)
- [Deploying and Running the Guidance](#deploying-and-running-the-guidance)
- [Rolling back changes at scale](#rolling-back-changes-at-scale)
- [Scenarios covered](#scenarios-covered)
  - [Bucket Rollback mode](#bucket-rollback-mode)
  - [Delete Marker Removal mode](#delete-marker-removal-mode)
  - [Copy to Bucket mode](#copy-to-bucket-mode)
- [Deployment workflow detail](#deployment-workflow-detail)
  - [Bucket Rollback mode](#bucket-rollback-mode-1)
  - [Delete Marker Removal mode](#delete-marker-removal-mode-1)
  - [Mode comparison](#mode-comparison)
  - [Copy to Bucket mode](#copy-to-bucket-mode-1)
- [Deployment Validation](#deployment-validation)
- [Creating a real-time inventory using the ListObjectVersions API](#creating-a-real-time-inventory-using-the-listobjectversions-api)
- [Simple demo](#simple-demo)
- [Ensuring recoverability](#ensuring-recoverability)
- [KMS permissions](#kms-permissions)
- [AWS Lambda concurrency reservations](#aws-lambda-concurrency-reservations)
- [FAQs](#faqs)
- [Cleanup](#cleanup)
- [Tenets](#tenets)
- [Additional resources](#additional-resources)
- [Revisions](#revisions)
- [Notices](#notices)
- [Authors](#authors)

## Overview
If you want to revert undesired changes to a dataset in Amazon S3, as quickly as possible, this tool is for you. It can detect and revert thousands of changes in under 15 minutes, 10 million changes in under an hour, or 100 million changes in under 5 hours. Alternatively, it can recreate a desired point in time into an empty bucket.

The only [Prerequisites](#prerequisites) are that [S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html) is enabled, and the desired versions still exist. If your bucket, or the prefix in scope, has up to 10 million objects, you can get started in under an hour after the undesired event by [creating a real-time inventory](#creating-a-real-time-inventory-using-the-listobjectversions-api). For buckets with [S3 Metadata live inventory tables](https://aws.amazon.com/blogs/aws/amazon-s3-metadata-now-supports-metadata-for-all-your-s3-objects/) enabled and up to 3 billion objects in scope for rollback, you can get started in only 15 minutes. If you don't have S3 Metadata enabled, or have even more objects in scope, you can get started in under 48 hours with an [S3 Inventory report](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html).

[Watch a 10 minute demo here:](https://www.youtube.com/watch?v=2XR2trZvv7w) [![Recorded demo of the tool](https://img.youtube.com/vi/2XR2trZvv7w/mqdefault.jpg)](https://www.youtube.com/watch?v=2XR2trZvv7w)

Undesired ‘soft DELETE’ (such as from [S3 Lifecycle expiry](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-expire-general-considerations.html)), ‘overwrite PUT’ and ‘non-overwrite PUT’ operations are all in scope - **you choose which of these to revert**. Changes to storage class by [S3 Lifecycle transitions](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html), or to [object tags](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-tagging.html), are not in scope as these do not create new object versions.

The tool will not permanently delete any data. In the default 'Bucket Rollback' mode it adds and removes delete markers, and copies object versions where necessary, to revert a dataset to an earlier time with the fewest possible operations. This is illustrated in the following animation:

![Rollback tool for Amazon S3 animated overview](images/s3rollbacktool-animated.gif) 

> In 'Delete Marker Removal' mode, it will simply remove delete markers placed after the specified time, down to the next regular object version, from objects where a delete marker is the current version. 

![Architecture diagram](images/arch-diagram.png)

### Cost

You are responsible for the cost of the AWS services used while running this Guidance. 

For example, as of 2025-11-17, if you use this tool in 'Bucket Rollback' mode against an entire bucket in the US East (N. Virginia) Region containing 1 billion objects and with S3 Metadata live inventory enabled or an existing Amazon S3 Inventory, to roll back 1 million non-overwrite PUTs, 1 million overwrite PUTs (of objects in Standard or Intelligent-Tiering classes`*`) *and* 1 million DELETEs since the desired point-in-time, **the total cost would be approximately $11**, detailed in the following table: 

| AWS service | Dimensions | Cost [USD] |
|-------------|------------|------------|
| [Amazon Athena](https://aws.amazon.com/athena/pricing/) | Data scanned | $1 |
| [Amazon S3](https://aws.amazon.com/s3/pricing/) | COPY requests | $5 |
| [Amazon S3 Batch Operations](https://aws.amazon.com/s3/pricing/) | Jobs and objects processed | $4 |
| [AWS Lambda](https://aws.amazon.com/lambda/pricing/) | Compute time and requests | $1 |

- `*` Objects in these S3 storage classes are not subject to a per-GB retrieval charge when copied. 
- Additional storage charges for copies are not included in the above estimates.
- Athena charges can be expected to scale with objects in the inventory. 
- Following the [simple demo](#simple-demo) costs $1. 
- In testing, deleting 100 million delete markers incurred $36 in Lambda charges.  Lambda compute time is higher for COPY than DELETE operations, and scales with object size.
- 'Copy to Bucket' mode copies every object (that was current at the timestamp) using S3 Batch Operations and Lambda. Cross-region copies will incur cross-region data transfer charges, and the lower throughput may increase Lambda compute time.
- S3 does not charge for DELETE operations.
- If the CloudFormation stack is not deleted after rollback is complete, ongoing S3 storage charges will apply for the temporary S3 bucket. See [Cleanup](#cleanup).


## Prerequisites

1. [S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html) must be enabled on your Amazon S3 bucket, and must have been enabled prior to the undesired event.

2. To return the state of a dataset to an earlier time, this tool exposes or copies desired versions of objects. **These desired versions must still exist**. See the section [Ensuring recoverability](#ensuring-recoverability) for more information.

3. A current inventory list of the bucket is required.
    - Ideally, [S3 Metadata live inventory](https://aws.amazon.com/blogs/aws/amazon-s3-metadata-now-supports-metadata-for-all-your-s3-objects/) is enabled on your bucket. For buckets with over a million objects, this is the fastest way for the tool to learn the current state of the bucket. S3 Metadata support has been tested up to 3 billion objects.
        - [Amazon S3 table buckets integration with AWS analytics services](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-aws.html) must be enabled.
        - The S3 Metadata table configuration must not use KMS encryption. If it does, or you have any other challenges with using S3 Metadata, you can choose not to use it during deployment.
        - *If you have more than 3B objects in scope, and want to use S3 Metadata with this tool, please contact the [authors](#authors) either directly or via your AWS account team.*
    - If S3 Metadata is not configured on your bucket, or you chose not to use it, the tool will attempt to detect and use an existing [S3 Inventory](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html).
        - S3 Inventory reports are supported in [Parquet or Apache ORC](https://docs.aws.amazon.com/athena/latest/ug/columnar-storage.html) format, when including all versions as well as all [additional metadata](https://docs.aws.amazon.com/AmazonS3/latest/userguide/configure-inventory.html#configure-inventory-console). They must not be stored with [KMS encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html). See [Deployment workflow](#deployment-workflow) for additional detail.
    - An inventory list can be provided manually, and will be used in preference of the above. See the section [Creating a real-time inventory using the ListObjectVersions API](#creating-a-real-time-inventory-using-the-listobjectversions-api).

4. **The operations you want to roll back must be included in the inventory**. 
    - S3 Inventory reports are delivered daily (or weekly), are eventually consistent, and might not include recently added or deleted objects. This can add more than a day to the start of your recovery time.
    - S3 Metadata support combines the live inventory table with the latest updates from the journal table, for the fastest start to your rollback.
5. For 'Copy to Bucket' mode, the destination bucket must already exist and have S3 Versioning enabled. This destination bucket, or the prefix in scope, **should be empty**. It can be in any AWS Region. If your destination bucket is in a different AWS Account, see the note in [Copy to Bucket mode](#copy-to-bucket-mode-1).

## Deploying and Running the Guidance

Before reverting changes, you may wish to prevent further changes taking place, for example with an update to your [bucket policy](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html) to deny PUT and DELETE operations (other than by roles created by this tool). You should also [temporarily disable any lifecycle expiry rules](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-configuration-examples.html#lifecycle-config-conceptual-ex2) to prevent objects being deleted during recovery. Then, deploy the [CloudFormation template](s3-rollback.yaml) in the same AWS Region as the S3 bucket you want to roll back or copy from. During deployment, specify:

1. **Bucket**: The name of the bucket to roll back or copy from.
2. **TimeStamp**: The date and time to roll back to *in the UTC timezone*, in ISO `yyyy-mm-ddThh:mm:ss` format. For example: `2025-08-30T02:00:00`.
3. **Prefix**: If you want to limit recovery to a specific prefix, specify it here, or leave blank to roll back the entire bucket. This will be used to select from available S3 Inventories, and is required if S3 Inventory is being used, unless there is an inventory report with the whole bucket in scope.
4. **Execution Mode**: 
    - **Bucket Rollback** is the default mode that creates S3 Batch Operations jobs to revert all changes in the bucket. This mode adds delete markers to new objects created after the timestamp, removes delete markers for existing objects deleted since the timestamp, and copies older object versions that have been overwritten since the point in time.
    - **Delete Marker Removal** mode simply removes delete markers placed after the timestamp, from objects where a delete marker is the current version. It will recover all objects that have been soft-deleted after the timestamp, including objects written or overwritten since the timestamp, back to their most recent version. The parameters **Copy to storage class** and **Copy using KMS key** are not used in this mode.
    - **Copy to Bucket** makes no changes to the original bucket. It recreates the precise state from an earlier point in time in *a different* bucket, by copying only the object versions which were current at the timestamp.
4. **Start S3 Batch Operations jobs**: The tool creates one or more S3 Batch Operations jobs to carry out S3 operations, but by default it will *not* start them. If you are working with a test dataset and do not need to validate the operations or adjust permissions (e.g. [KMS](#kms-permissions)), change this to `Yes`.
5. **Use S3 Metadata tables**: Change this to `NO` to skip the check for S3 Metadata journal and live inventory tables. This is helpful if you have KMS encryption on either table, or any other errors (such as permissions).
5. **Copy to storage class**: The Amazon S3 storage class to copy object versions into. [If in doubt, use the Intelligent-Tiering storage class](https://catalog.workshops.aws/awscff/en-US/playbooks/storage/s3/2-choosing-storage-class). See [the S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/sc-howtoset.html) for more information.
6. **Copy using KMS key (optional)**: If object versions should be created using a KMS encryption key, specify it here. Leave blank for bucket default. *Permissions to KMS keys are not updated by this tool - see the* [**KMS permissions**](#kms-permissions) *section.*
7. **Specify CSV inventory (optional)**: The S3 location of a CSV containing a current inventory of the bucket. This optional field allows you to provide a list of object versions, instead of using [S3 Metadata](https://aws.amazon.com/blogs/aws/amazon-s3-metadata-now-supports-metadata-for-all-your-s3-objects/) or [S3 Inventory](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html). See the section [Creating a real-time inventory using the ListObjectVersions API](#creating-a-real-time-inventory-using-the-listobjectversions-api).
8. **Destination Bucket (optional)**: For use only with 'Copy to Bucket' mode. The tool will copy all objects to this bucket that were current at the timestamp.

<details>
<summary><strong>Expand this section for detailed instructions for deploying the CloudFormation template.</strong></summary>

1. Download the [s3-rollback.yaml](s3-rollback.yaml) file from GitHub to your computer.
2. In the AWS Management Console, in the same region as your S3 bucket, find and select **CloudFormation**.
3. Choose **Create stack - With new resources (standard)**
4. For **Template Source**, choose, **Upload a template file**, **Choose file**, and select the `s3-rollback.yaml` file you downloaded. Choose **Next**. ![CloudFormation prerequisite - prepare template](images/cf1.png)
5. Enter a name for the stack. This must be unique for your account and region. Enter and choose parameters as described above and in the [simple demo](#simple-demo) below, then choose **Next**. ![CloudFormation stack details](images/cf2.png)
6. Scroll to the end of the **Configure stack options** screen, check the box **I acknowledge that AWS CloudFormation might create IAM resources**, and choose **Next**. ![Configure stack options](images/cf3.png)
7. Scroll to the end of the **Configure stack options** screen and choose **Submit**.

---

</details>

1. Once the CloudFormation deployment is complete, review the [S3 Batch Operations jobs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops-managing-jobs.html) and their manifests using the Amazon S3 Console. 

2. If you are working in a production account, review the section on [AWS Lambda concurrency reservations](#aws-lambda-concurrency-reservations) and make any necessary changes.

3. If using KMS encryption, ensure the roles created by this tool have the necessary permissions before running the jobs (see [KMS permissions](#kms-permissions)).

**CAUTION: [S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html) must be enabled when the Batch Operations Jobs are run. The tool will check that versioning is enabled on your bucket during deployment, but data loss can occur if versioning is suspended prior to the jobs being run.**

4. When you are ready, **run the desired S3 Batch Operations job(s).**

> There may be object versions that need to be copied to complete the recovery, but are in an asynchronous [S3 storage class](https://aws.amazon.com/s3/storage-classes/) (i.e. Glacier Flexible Retrieval or Glacier Deep Archive). This tool will not retrieve and copy these, but will output their details in a CSV manifest `scenario3a.csv` or  `scenario5a.csv`, in the S3 path referenced in the `Manifests` output, that can be used with the solution [**Guidance for Automated Restore and Copy for Amazon S3 Glacier Objects**](https://github.com/aws-solutions-library-samples/guidance-for-automated-restore-and-copy-for-amazon-s3-glacier-objects).

When you no longer need the manifests and other artifacts created by the tool, see the [Cleaning up](#cleaning-up) section.

## Rolling back changes at scale   

Users can [see the version history of a single object key in the S3 console](https://docs.aws.amazon.com/AmazonS3/latest/userguide/list-obj-version-enabled-bucket.html), and reverting a single object to an earlier version is as simple as [making a copy of the desired version to the same object key name](https://docs.aws.amazon.com/AmazonS3/latest/userguide/RestoringPreviousVersions.html), or [deleting a delete marker](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ManagingDelMarkers.html). If an undesired new object has been placed (i.e. a ‘non-overwrite PUT’, not an overwrite of an existing object), it can have a delete marker placed on it with a DELETE operation.

The Rollback Tool for Amazon S3 enables restoration at scale, finding changes within buckets containing billions of objects in minutes (using [Amazon Athena](https://aws.amazon.com/athena/)). It then determines the most efficient operations needed to revert those changes, and creates [S3 Batch Operations](https://aws.amazon.com/s3/features/batch-operations/) jobs to enact them. *By default, these jobs are not started automatically*. Users should review the jobs and their manifests to ensure the rollback operations are desired, before proceeding to run the jobs. The jobs can be found in the S3 Batch Operations console, or the CloudFormation output.

If you prefer to copy your dataset into an empty bucket or prefix, as it was at a point in time, choose the 'Copy to Bucket' mode. Note that this will require copying *every* object that was a current version at the chosen timestamp, and is likely to be significantly slower and more expensive than rolling back in place. 


## Scenarios covered

Scenarios differ depending on the selected mode. For each object key (name) in scope of the prefix filter:

### Bucket Rollback mode

1. **Revert ‘non-overwrite PUTs’:** At the desired point in time (tD), there was either no object with this name, or the current version was a delete marker. Since tD, a new object has been written (and potentially overwritten by further versions). The current version of the object is not a delete marker. 
    - **Action: Add a delete marker as the current version.**
2. **Revert DELETE operations:** At tD, a regular object (not a delete marker) was current. Since tD there has been at least one DELETE operation, but no regular objects have been PUT to the key. 
    - **Action: Delete all delete markers written since tD.**
3. **Revert ‘overwrite PUTs’:** At tD, a regular object (not a delete marker) was current. Since tD, there has been at least one PUT to the same key. 
    - **Action: Copy the desired version, creating a new current version.**
    - **Exception:** If the the desired version is in an an asynchronous S3 storage class (Glacier Flexible Retrieval or Glacier Deep Archive), include this in the `scenario3a.csv` manifest. You can then use [**Guidance for Automated Restore and Copy for Amazon S3 Glacier Objects**](https://github.com/aws-solutions-library-samples/guidance-for-automated-restore-and-copy-for-amazon-s3-glacier-objects) to retrieve those object versions and copy them in place.
    - **Note:** In this scenario you may prefer to permanently delete the newer object versions. Due to the potential for data loss, the tool does *not* include this capability.

### Delete Marker Removal mode

4. **Revert *all* DELETE operations since tD:** The current version of the object is a delete marker. 
    - **Action: Delete all delete markers written since tD.** In the event of an object version having been placed since tD, only delete markers written after the latest object version will be deleted.

### Copy to Bucket mode
5. **Recreate tD in the destination bucket:**
    - **Action: Copy all objects that were current version at tD to the destination bucket.**
    - **Exception:** If the the desired version is in an an asynchronous S3 storage class (Glacier Flexible Retrieval or Glacier Deep Archive), include this in the `scenario5a.csv` manifest. You can then use [**Guidance for Automated Restore and Copy for Amazon S3 Glacier Objects**](https://github.com/aws-solutions-library-samples/guidance-for-automated-restore-and-copy-for-amazon-s3-glacier-objects) to retrieve those object versions and copy them in place.

## Deployment workflow detail

The tool requires an inventory of the bucket or prefix in scope. If none is available, the deployment will fail.

- If the **Specify CSV inventory** CloudFormation stack parameter has an entry, the tool will read it from the specified S3 location. If it cannot read or process this file, the deployment will fail.
- If not, the tool will look at the configuration of the selected bucket. If the bucket has an [S3 Metadata live inventory table](https://aws.amazon.com/blogs/aws/amazon-s3-metadata-now-supports-metadata-for-all-your-s3-objects/) configured and in active status, the tool will query the bucket's S3 Metadata tables (using any specified prefix as a filter).
    - [Amazon S3 table buckets integration with AWS analytics services](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables-integrating-aws.html) must be enabled.
    - The S3 Metadata table configuration must not use KMS encryption. If it does, or you have any other challenges with using S3 Metadata, you can choose not to use it during deployment by changing **Use S3 Metadata tables** to `NO`.
- Otherwise, the tool looks at the bucket's S3 Inventory configuration. It evaluates the available S3 Inventory reports, and chooses the latest one that is in [Parquet or Apache ORC](https://docs.aws.amazon.com/athena/latest/ug/columnar-storage.html) format and includes all versions as well as all [additional metadata](https://docs.aws.amazon.com/AmazonS3/latest/userguide/configure-inventory.html#configure-inventory-console). If a prefix has been specified in the CloudFormation deployment, the tool will prioritize inventory configurations matching or containing the specified prefix. If no prefix was specified, there must be a valid S3 Inventory configuration containing all objects in the bucket, or the deployment will fail. 


The CloudFormation template creates the following resources:

* A temporary S3 bucket to store Athena output, associated [AWS Glue database](https://docs.aws.amazon.com/glue/latest/dg/define-database.html), Athena [workgroup](https://docs.aws.amazon.com/athena/latest/ug/workgroups-create-update-delete.html) and [tables](https://docs.aws.amazon.com/athena/latest/ug/creating-tables.html), plus S3 Batch Operations manifests and reports.
   - By default, new S3 buckets have [Block Public Access](https://aws.amazon.com/s3/features/block-public-access/) enabled and [new objects are encrypted at rest with SSE-S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-encryption-faq.html).
* Lambda functions to create the above resources, check versioning is enabled, determine which inventory to use, orchestrate Athena queries, create manifests in the correct formats, create S3 Batch Operations jobs, and clean up the temporary S3 bucket when the CloudFormation stack is deleted.
* IAM roles for use by the tool.

The tool outputs manifest files for S3 Batch Operations jobs, in the S3 path referenced in the `Manifests` output, depending on the mode selected:

### Bucket Rollback mode

* **Scenario 1**:  Where all versions of a key have last_modified after the desired Point In Time (tD), or the current version at tD was a delete marker, and current version is not a delete marker. **Add a delete marker.**
* **Scenario 2:**  Keys where there are only delete markers (no new objects) after tD. **These delete markers will be deleted.**
    * `scenario2-undo.csv` will also be created so that these can be recreated if needed. See [FAQ #3](#faqs).
* **Scenarios 3a, b and c:** Keys where there was an object (not a delete marker) at the tD, and there is current version newer than at tD. Excludes keys covered by scenarios 1 and 2. **The desired VersionID will be copied to the same key, making it the current version.**
    * **3a:** Desired VersionID is in Glacier Flexible Retrieval or Deep Archive class and needs to be restored from async. **This tool will not copy these objects**, but output its details in the `scenario3a.csv` manifest that can be used with [**Guidance for Automated Restore and Copy for Amazon S3 Glacier Objects**](https://github.com/aws-solutions-library-samples/guidance-for-automated-restore-and-copy-for-amazon-s3-glacier-objects).
        * Note that as ListObjectVersions and S3 Metadata don't report Intelligent Tiering tier, objects versions that need to be retrieved before being copied from the *opt-in* asynchronous Archive and Deep Archive tiers won’t be included here unless an S3 Inventory source is used. Instead they will be included in scenario 3b or 3c, and an attempt made to copy them. These copy attempts will fail with `403: InvalidObjectState` 
    * **3b:** Desired VersionID <= 5 GiB. These objects will be copied with the CopyObject API, with an S3 Batch Operations Copy job.
    * **3c:** Desired VersionID > 5 GiB. These objects require special handling. A separate S3 Batch Operations Lambda job will perform these copies, reusing code from the solution at https://aws.amazon.com/blogs/storage/copying-objects-greater-than-5-gb-with-amazon-s3-batch-operations/ .
            

**You are in control of the above operations. For example, if you don't want to revert non-overwrite PUTs, don't run the `Scenario 1` Batch Operations job.**

The following flow diagram illustrates the Bucket Rollback mode workflow:

<img src="images/s3rollbackflow.png" alt="Flow diagram" width="65%">

### Delete Marker Removal mode

* **Scenario 4**:  At tD, a regular object (not a delete marker) was current, and now the current version of the object is a delete marker. **All delete markers written since tD will be deleted**. In the event of an object version having been placed since tD, only delete markers written after the latest object version will be deleted.

### Mode comparison

The table below compares the behaviour of the Bucket Rollback and Delete Marker Removal modes for each key, where 'V' are object versions (PUT operations), and 'DM' are delete markers (DELETE operations):

| Operations before tD | Operations after tD | Bucket Rollback Mode behaviour | Remove Delete Marker mode behaviour |
|---|---|---|---|
| V1 | DM1 | Delete DM1, exposing V1 | Delete DM1, exposing V1
| *None* | V1, DM1 | *No action* | Delete DM1, exposing V1 |
| V1, DM1 | V2, DM2 | *No action* | Delete DM2, exposing V2 |
| V1, DM1 | DM2 | *No action* | Delete DM2, object remains soft-deleted due to DM1 |
| V1 | DM1, V2, DM2 | Copy V1 | Delete DM2, exposing V2 |
| V1 | V2 | Copy V1 (as current version V3) | *No action*
| *None*| V1 | Place DM, key is soft deleted | *No action*


### Copy to Bucket mode

* **Scenario 5**: Every object that was a current version at tD will be copied to the destination bucket. Objects where the current version was a delete marker will not be copied. All copies are done using a Lambda function, to allow for cross-region copies.

>Note: If the destination S3 bucket is in another AWS account, then you must also apply a resource level bucket policy to the S3 bucket with the required permissions. See the following S3 bucket policy example with the minimum required permissions:

```
{
    "Version": "2012-10-17",
    "Id": "PolicyName",
    "Statement": [
        {
            "Sid": "Allow Cross Account Copy",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::1234567890:root"
            },
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl",
	 "s3:PutObjectTagging"
            ],
            "Resource": "arn:aws:s3:::DESTINATION_BUCKET/*"
        }
    ]
}
```
>`1234567890` in the bucket policy Principal is the source account AWS Account-ID. It is recommended to set Object Ownership on the destination account bucket to Bucket owner enforced. If you have a special use case that requires using ACLs, set it to Bucket owner preferred to ensure that the destination account owns the objects.



## Deployment Validation

All S3 Batch Operations jobs created by this tool create completion reports, including both success and failure for each task. If you find you have some failures and wish to manually address these (as opposed to re-running the tool), follow the [instructions in this article](https://repost.aws/articles/ARq3WkGPYmSmiT3xC8Me_mvA/querying-s3-batch-operations-completion-reports-with-amazon-athena) to find the failures.


## Creating a real-time inventory using the ListObjectVersions API

The S3 rollback tool has been designed to operate with up to 10 billion objects in scope. At this scale, S3 Inventory or S3 Metadata inventory tables are necessary in order to obtain a listing of all the object versions and their details.

If you have up to 100 million objects in your bucket, or are looking to perform rapid iterative testing of this tool, you may prefer to list all the objects in scope rather than use S3 Inventory or S3 Metadata. The example script provided at https://github.com/aws-samples/sample-s3-listobjectversions-to-csv/ uses [list_object_versions with boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3/client/list_object_versions.html) to output all object versions in a bucket (and optionally prefix), into a CSV format that can be used with this tool (with the following headers: `bucket_name, key_name, version_id, is_latest, delete_marker, size, last_modified, storage_class, e_tag`). The example script lists 1 million object versions in 5 minutes (on a t2.micro instance), and 50 million in approximately 5 hours.

Copy your output CSV into another S3 bucket in the region you are working in. You can do this using the [AWS CLI](https://aws.amazon.com/cli/) with `aws s3 cp s3_object_versions.csv s3://my-s3-bucket/prefix/name.csv` - note the object name must end `.csv`.



## Simple demo

For a simple demonstration example, costing $0.75-$1.00 (for the three to four S3 Batch Operations jobs), perform the following:

1. Create a new S3 bucket for testing, with versioning enabled. Keep all other defaults.
    - Optionally enable S3 [S3 Metadata live inventory tables](https://aws.amazon.com/blogs/aws/amazon-s3-metadata-now-supports-metadata-for-all-your-s3-objects/) for the bucket, without KMS encryption, and wait for the live inventory table to have status 'Active'.
2. Create text files `a` and `b`, and upload them to the new bucket. **This is your desired (good) state**. Note the time, adjusted to the UTC timezone.
   - The bucket will look like this in the console:
   
   ![DT state](images/dt.png)

   - And like this with **Show versions** enabled:
   
   ![DT state with versions](images/dt(versions).png)

   - Disable **Show versions** before proceeding.

3. Delete object `a` (ensure this is done without **Show versions** enabled - you should be typing `delete` not `permanently delete`), overwrite object `b` (by editing the text file and then uploading it to the same key name), and write new object `c`. **This is your undesired state, that you want to roll back**.
   - The bucket will look like this in the console: 
   
   ![Undesired state](images/badstate.png)

   - And like this with **Show versions** enabled:
   
   ![DT state with versions](images/badstate(versions).png)

4. If you did not enable S3 Metadata live inventory tables, instead create a real-time inventory (see [Creating a real-time inventory using the ListObjectVersions API](#creating-a-real-time-inventory-using-the-listobjectversions-api)), and copy the output CSV into S3.
5. Deploy the [CloudFormation template](s3-rollback.yaml).
   - Enter the name of the test bucket you just created.
   - Enter a timestamp (in UTC) after step 2 but before step 3. In this example, it was `2025-08-22T11:01:00`.
   - Leave the default **Bucket Rollback** mode selected.
   - Set **Start S3 Batch Operations jobs** to `YES`.
   - Set **Specify CSV inventory** to the URI of the real-time CSV inventory object you uploaded.
   - **Create Stack**.
6. Observe the CloudFormation template deployment. When complete, observe the S3 Batch Operations jobs referenced in the output, and review their manifests. The whole process should complete in around 2 minutes.
   - The bucket will look like this in the console - back to your desired state: 
   
   ![Rolled-back state](images/rolledback.png)

   - And like this with **Show versions** enabled:
   
   ![Rolled-back state with versions](images/rolledback(versions).png)

   - Note that the newest version of object `b` is a copy of the oldest version. 


## Ensuring recoverability

[Amazon S3](https://aws.amazon.com/s3/) is an object storage service with industry-leading scalability, data availability, security, performance, and 99.999999999% (11 9s) of data durability. [S3 Versioning](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html) protects against accidental deletions and overwrites by keeping multiple variants of an object in the same S3 bucket, and placing a [delete marker as the current version in response to simple DELETE requests](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DeletingObjectVersions.html) (i.e. without specifying a VersionID). 

To return the state of a dataset to an earlier time, this tool exposes or copies desired versions of objects. **These desired versions must still exist**. Administrators can prevent the deletion of specific versions in Amazon S3 by denying use of the [DeleteObjectVersion](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html) and [PutLifeCycleConfiguration](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutBucketLifecycleConfiguration.html) APIs with [bucket policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html), [access point policies](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points-policies.html), [service control policies (SCPs)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html), or [resource control policies (RCPs)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html), and/or by using [S3 Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html). For a complementary solution to assist with this, see [**Maintaining object immutability by automatically extending Amazon S3 Object Lock retention periods**](https://aws.amazon.com/blogs/storage/maintaining-object-immutability-by-automatically-extending-amazon-s3-object-lock-retention-periods/). You can also use [S3 Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html) to make and maintain a copy of your data in another bucket (see [FAQ #9](#faqs)).


## KMS permissions

The roles created by this tool **will not have IAM or KMS permissions to any KMS keys**. If you have objects in scope that are encrypted with KMS, or you have specified that the solution should use a KMS key when copying objects, or your bucket has default encryption set to a KMS key, you will need to update permissions before starting the Batch Operations copy jobs (Scenarios 3 and 5). The ARNs that require permissions are shown in the `CopyRole3b`, `CopyRole3c` and `CopyRole5b` output of the CloudFormation stack.

Permissions required:
- KMS decrypt for object versions to be copied
- KMS encrypt for the key you selected copies to be encrypted with (or the bucket default)

KMS permissions are *not* required for scenario 1, 2 and 4 jobs, as DELETE operations do not encrypt or decrypt data.

## AWS Lambda concurrency reservations

**This tool does not assign [reserved concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html)** to the Lambda functions it creates, and may consume all the available [AWS Lambda concurrent execution quota](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html#compute-and-storage). **If you have concerns about this, review the S3 Batch Operations tasks that use Lambda (scenarios 1, 2, 3c, 4 and 5), and [adjust the Lambda functions accordingly](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html#configuring-concurrency-reserved) before running the jobs**. We made this design decision because 1/ at the time the functions are created the tool does not know the number of objects (if any) in scope for the relevant scenarios, 2/ we don't know the Lambda concurrent execution quota, or how much it is appropriate to consume, and 3/ when a function has reserved concurrency, no other function can use that concurrency. Any reservation assigned by the tool would have been held until the CloudFormation stack was deleted.

The Lambda functions created by this tool (for use with S3 Batch Operations) have been tested with up to 1000 concurrent executions. Please see [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) in the AWS documentation.


## FAQs

1. What did I need to have in place prior to an event that I want to recover from, in order to use this tool?
    - See [Prerequisites](#prerequisites).
2. What about changes that took place after the inventory was created, or after the rollback process was started?
    - This tool will take the appropriate action to revert changes, based on the available inventory information at the time of deployment. The impact of a subsequent change depends on its nature, and whether it took place before or after the specific S3 Batch Operations job (created  by this rollback tool) took its action. However, in all cases, running the rollback tool *again*, with an inventory that includes new changes, will correctly revert the new changes. Note: If the new inventory includes copies made by this tool (scenarios 3 and 5), those copies will be repeated as it isn't possible to be certain (from an inventory) that the current object version's data is identical to the desired version.
3. I used this tool to roll back my bucket, and want to undo the changes. Can I do that?
    - Yes, provided you have not yet deleted the CloudFormation stack from the original deployment. 
    - For Bucket Rollback mode, This is a 2-step process:
      1. Once you have an inventory that includes the changes made by this tool, deploy the tool again in **Bucket Rollback** mode to roll back to the point in time just prior to the initial deployment. If you have S3 Metadata live inventory enabled, simply wait 15 minutes to ensure all changes have been written to the journal table, as the tool incorporates uncoalesced journal entries. 
      2. Once the above is complete, you need to re-create any delete markers removed by the original deployment, as the updated inventory has no knowledge of these. To do this, create an S3 Batch Operations job, choose the CSV manifest `scenario2_undo.csv` from the *original deployment*, leaving **Manifest includes version IDs** unchecked. Choose **Invoke AWS Lambda function** as the operation type, the function titled `<original stack name>-S3BatchOpsDeleteFunction-<unique-id>` (you can find this name in the **Resources** output of the deployment) and **Invocation schema version 2.0**. Run the job with the role titled `<original stack name>-S3BatchOpsExecutorRole--<unique-id>`.
    - For Delete Marker Removal mode, you need to re-create any delete markers removed by the original deployment. To do this, create an S3 Batch Operations job, choose the CSV manifest `scenario4_undo.csv` from the *original deployment*, leaving **Manifest includes version IDs** unchecked. Choose **Invoke AWS Lambda function** as the operation type, the function titled `<original stack name>-S3BatchOpsDeleteFunction-<unique-id>` (you can find this name in the **Resources** output of the deployment) and **Invocation schema version 2.0**. Run the job with the role titled `<original stack name>-S3BatchOpsExecutorRole--<unique-id>`.
4. What happens when I delete the CloudFormation stack?
    - Any S3 Batch Operations jobs not in COMPLETE state will be cancelled.
    - All created resources will be deleted, including the temporary S3 bucket.
5. I don’t want to delete delete markers, but instead copy the desired object version to the top of the version stack.
    - As this increases both cost and time, delete markers contain no data, and deletion of delete markers is reversible, we made the decision not to prioritize providing this as an option. We may consider adding this capability based on feedback - see [Issues](https://github.com/aws-samples/sample-s3-rollback-tool/issues).
6. I have [ACLs enabled](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ensure-object-ownership.html) on my bucket, can I still use this tool?
    - Yes, however the copy operations carried out by this tool will overwrite the ownerID, which may cause access issues for the original uploading account.
7. I have objects in my bucket with `null` version IDs from when S3 Versioning was disabled or suspended. Will this tool still work?
    - Yes, the tool will correctly copy objects and delete delete markers that have a `null` version ID. Versioning must be enabled when this tool is deployed. When versioning is suspended, objects and delete markers are written with `null` version IDs, and will permanently and immediately overwrite any existing object with the same version ID.
8. I only want to remove delete markers, not any PUT operations. Is this tool still useful?
    - Yes:
        - To simply delete all delete markers placed after tD, select the dedicated **Remove Delete Markers** mode during deployment.
        - Or, if you only want to remove delete markers from objects that were present (with a current version that was not a delete marker) at tD, select the default **Bucket Rollback** mode, leave **Start S3 Batch Operations Jobs** as `FALSE`, and then only run the scenario 2 job. Note this will not act on keys where an object has been PUT after tD, as this falls under scenario 3.
9. Does this work with S3 Replication?
    - Yes. If you have used S3 Replication to make a copy of your data in another bucket, you could use this tool to revert either the source or destination bucket. Note that permanent delete operations (including of delete markers) are not replicated, and replication of new delete markers is [optional and not available for tag-filtered replication rules](https://docs.aws.amazon.com/AmazonS3/latest/userguide/delete-marker-replication.html).
    - If you want to roll back both source and destination buckets:
        1. Ensure replication of delete markers is enabled in your replication rule.
        2. Roll back the source bucket to tD, running all Batch Operations jobs. The operations carried out by the Scenario 1 and 3 jobs will be replicated.
        3. Roll back the destination bucket to tD, running only the Scenario 2 job, as these operations will *not* have been replicated.
    - If you only want to roll back the destination (for example, you are failing over to using the destination bucket):
        1. [Disable the replication rule](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-add-config.html#replication-config-min-rule-config) to prevent further replication from source to destination.
        2. Using an inventory that includes all replicated changes, roll back the destination bucket to tD.
    - You may also want to consider using the [Replication Validation Tool for S3](https://github.com/aws-samples/sample-RVT-S3) to continuously validate that your destination bucket (or prefix) is a complete copy of your source.
    - Note that the tool uses the 'last modified' date of each object (i.e. when it was created), and this date is maintained when objects are replicated. Therefore, when rolling back a destination bucket you need to be conscious that actions are calculated according to the date an object was *created*, **not** the date it was *replicated*.
10. For 'Copy to Bucket' mode, why does the destination bucket need S3 Versioning enabled?
    - This tool will copy objects into the destination bucket without considering the current state of objects in the destination. If versioning is not enabled, these copies could result in overwrites causing permanent and irreversable data loss.


## Cleanup

As the tool is deployed once per use, it should be removed once the jobs are complete and the reports, tables and manifests are no longer required.

To clean up, delete the CloudFormation stack. This will delete any CSV manifests generated by the tool (which might be required for undo - see FAQ 3), as well as all the resources it created.

## Tenets

* **Works at any scale**: Designed to support 10B object buckets with 1B changes. Efficiency is key to achieving this. Re-running with a new inventory (to catch up, or following failures/cancellations) will not repeat completed operations (where possible, see FAQ 2).
* **Fast at any scale:** Revert 1M changes in a 10B object bucket in under 1 hour, end to end, using S3 Inventory. Delete 100M delete markers in 3.5 hours (with 1000 concurrent lambda executions, and where S3 prefix partitioning permits). Revert a few thousand changes in a 1M object bucket in under 15 minutes, including real-time CSV inventory creation.
* **Do no harm:** Do not delete data (never regular object versions, only delete markers). Create S3 Batch Operations jobs but do not start them by default. Ensure S3 Versioning is enabled.
* **Flexible**: Use any inventory. Give users control of prefix and which operations to carry out. 
* **No assumptions**: Don’t assume which operations the user wants to run, which storage class to copy to, or encryption to use for copies. We can’t know what’s ‘correct’, so we ask.

## Additional resources

* [AWS CloudFormation product page](https://aws.amazon.com/cloudformation/)
* [Amazon S3 Batch Operations documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/batch-ops.html)
* [AWS Lambda product page](https://aws.amazon.com/lambda/)
* [AWS Lambda Performance Guidelines](https://docs.aws.amazon.com/lambda/latest/operatorguide/perf-optimize.html)
* [Serverless Data Integration – AWS Glue – Amazon Web Services](https://aws.amazon.com/glue/)
* [Interactive SQL - Serverless Query Service - Amazon Athena - AWS](https://aws.amazon.com/athena/)
* [Amazon S3 Inventory](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-inventory.html)
* [Amazon S3 Metadata tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/metadata-tables-overview.html)
* [Amazon S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)

## Revisions

- 2025-09-15 - Initial release
- 2025-10-31 
    - Added Remove Delete Marker mode
    - 4x improvement in Lambda function performance
    - Updated readme.md to comply with new repository requirements
- 2025-11-06
    - Added 'Copy to Bucket' mode
    - Updated GitHub references following the move to https://github.com/aws-solutions-library-samples/
- 2025-11-27 - Updated to include support for S3 Metadata tables

## Notices

Customers are responsible for making their own independent assessment of the information in this Guidance. This Guidance: (a) is for informational purposes only, (b) represents AWS current product offerings and practices, which are subject to change without notice, and (c) does not create any commitments or assurances from AWS and its affiliates, suppliers or licensors. AWS products or services are provided “as is” without warranties, representations, or conditions of any kind, whether express or implied. AWS responsibilities and liabilities to its customers are controlled by AWS agreements, and this Guidance is not part of, nor does it modify, any agreement between AWS and its customers.

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.

## Authors

* Ed Gummett, Senior Storage Specialist Solutions Architect, AWS. [Connect on LinkedIn.](https://www.linkedin.com/in/egummett/)
* Paul Gargan, Senior Solutions Architect, AWS.
* Tom Bailey, Senior Technical Account Manager, AWS. [Connect on LinkedIn.](https://www.linkedin.com/in/tom-bailey-1633866/)

