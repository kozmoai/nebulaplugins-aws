# `nebula-aws`

<p align="center">
    <a href="https://pypi.python.org/pypi/nebula-aws/" alt="PyPI version">
        <img alt="PyPI" src="https://img.shields.io/pypi/v/nebula-aws?color=26272B&labelColor=090422"></a>
    <a href="https://github.com/kozmoai/nebula-aws/" alt="Stars">
        <img src="https://img.shields.io/github/stars/kozmoai/nebula-aws?color=26272B&labelColor=090422" /></a>
    <a href="https://pepy.tech/badge/nebula-aws/" alt="Downloads">
        <img src="https://img.shields.io/pypi/dm/nebula-aws?color=26272B&labelColor=090422" /></a>
    <a href="https://github.com/kozmoai/nebula-aws/pulse" alt="Activity">
        <img src="https://img.shields.io/github/commit-activity/m/kozmoai/nebula-aws?color=26272B&labelColor=090422" /></a>
    <br>
    <a href="https://nebula-community.slack.com" alt="Slack">
        <img src="https://img.shields.io/badge/slack-join_community-red.svg?color=26272B&labelColor=090422&logo=slack" /></a>
    <a href="https://discourse.nebula.io/" alt="Discourse">
        <img src="https://img.shields.io/badge/discourse-browse_forum-red.svg?color=26272B&labelColor=090422&logo=discourse" /></a>
</p>

## Welcome!

`nebula-aws` makes it easy to leverage the capabilities of AWS in your flows, featuring support for ECSTask, S3, Secrets Manager, Batch Job, and Client Waiter.


## Getting Started

### Saving credentials to a block

You will need an AWS account and credentials in order to use `nebula-aws`.

1. Refer to the [AWS Configuration documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds) on how to retrieve your access key ID and secret access key
2. Copy the access key ID and secret access key
3. Create a short script and replace the placeholders with your credential information and desired block name:

```python
from nebula_aws import AwsCredentials
AwsCredentials(
    aws_access_key_id="PLACEHOLDER",
    aws_secret_access_key="PLACEHOLDER",
    aws_session_token=None,  # replace this with token if necessary
    region_name="us-east-2"
).save("BLOCK-NAME-PLACEHOLDER")
```

Congrats! You can now load the saved block to use your credentials in your Python code:
 
```python
from nebula_aws import AwsCredentials
AwsCredentials.load("BLOCK-NAME-PLACEHOLDER")
```

!!! info "Registering blocks"

    Register blocks in this module to
    [view and edit them](https://docs.nebula.io/ui/blocks/)
    on Nebula Cloud:

    ```bash
    nebula block register -m nebula_aws
    ```

### Using Nebula with AWS ECS

`nebula_aws` allows you to use [AWS ECS](https://aws.amazon.com/ecs/) as infrastructure for your deployments. Using ECS for scheduled flow runs enables the dynamic provisioning of infrastructure for containers and unlocks greater scalability.

The snippets below show how you can use `nebula_aws` to run a task on ECS. The first example uses the `ECSTask` block as [infrastructure](https://docs.nebula.io/concepts/infrastructure/) and the second example shows using ECS within a flow.

#### As deployment Infrastructure


##### Set variables

To expedite copy/paste without the needing to update placeholders manually, update and execute the following.

```bash
export CREDENTIALS_BLOCK_NAME="aws-credentials"
export VPC_ID="vpc-id"
export ECS_TASK_BLOCK_NAME="ecs-task-example"
export S3_BUCKET_BLOCK_NAME="ecs-task-bucket-example"
```

##### Save an infrastructure and storage block

Save a custom infrastructure and storage block by executing the following snippet.

```python
import os
from nebula_aws import AwsCredentials, ECSTask, S3Bucket

aws_credentials = AwsCredentials.load(os.environ["CREDENTIALS_BLOCK_NAME"])

ecs_task = ECSTask(
    image="kozmoai/nebula:2-python3.10",
    aws_credentials=aws_credentials,
    vpc_id=os.environ["VPC_ID"],
)
ecs_task.save(os.environ["ECS_TASK_BLOCK_NAME"], overwrite=True)

bucket_name = "ecs-task-bucket-example"
s3_client = aws_credentials.get_s3_client()
s3_client.create_bucket(
    Bucket=bucket_name,
    CreateBucketConfiguration={"LocationConstraint": aws_credentials.region_name}
)
s3_bucket = S3Bucket(
    bucket_name=bucket_name,
    credentials=aws_credentials,
)
s3_bucket.save(os.environ["S3_BUCKET_BLOCK_NAME"], overwrite=True)
```

##### Write a flow

Then, use an existing flow to create a deployment with, or use the flow below if you don't have an existing flow handy.

```python
from nebula import flow

@flow(log_prints=True)
def ecs_task_flow():
    print("Hello, Nebula!")

if __name__ == "__main__":
    ecs_task_flow()
```

##### Create a deployment

If the script was named "ecs_task_script.py", build a deployment manifest with the following command.

```bash
nebula deployment build ecs_task_script.py:ecs_task_flow \
    -n ecs-task-deployment \
    -ib ecs-task/${ECS_TASK_BLOCK_NAME} \
    -sb s3-bucket/${S3_BUCKET_BLOCK_NAME} \
    --override env.EXTRA_PIP_PACKAGES=nebula-aws
```

Now apply the deployment!

```bash
nebula deployment apply ecs_task_flow-deployment.yaml
```

##### Test the deployment

Start an [agent](https://docs.nebula.io/latest/concepts/work-pools/) in a separate terminal. The agent will poll the Nebula API's work pool for scheduled flow runs.

```bash
nebula agent start -q 'default'
```

Run the deployment once to test it:

```bash
nebula deployment run ecs-task-flow/ecs-task-deployment
```

Once the flow run has completed, you will see `Hello, Nebula!` logged in the CLI and the Nebula UI.

!!! info "No class found for dispatch key"

    If you encounter an error message like `KeyError: "No class found for dispatch key 'ecs-task' in registry for type 'Block'."`,
    ensure `nebula-aws` is installed in the environment in which your agent is running!

Another tutorial on `ECSTask` can be found [here](https://towardsdatascience.com/nebula-aws-ecs-fargate-github-actions-make-serverless-dataflows-as-easy-as-py-f6025335effc).

#### Within Flow

You can also execute commands with an `ECSTask` block directly within a Nebula flow. Running containers via ECS in your flows is useful for executing non-Python code in a distributed manner while using Nebula.

```python
from nebula import flow
from nebula_aws import AwsCredentials
from nebula_aws.ecs import ECSTask

@flow
def ecs_task_flow():
    ecs_task = ECSTask(
        image="kozmoai/nebula:2-python3.10",
        credentials=AwsCredentials.load("BLOCK-NAME-PLACEHOLDER"),
        region="us-east-2",
        command=["echo", "Hello, Nebula!"],
    )
    return ecs_task.run()
```

This setup gives you all of the observation and orchestration benefits of Nebula, while also providing you the scalability of ECS.

### Using Nebula with AWS S3

`nebula_aws` allows you to read and write objects with AWS S3 within your Nebula flows.

The provided code snippet shows how you can use `nebula_aws` to upload a file to a AWS S3 bucket and download the same file under a different file name.

Note, the following code assumes that the bucket already exists.

```python
from pathlib import Path
from nebula import flow
from nebula_aws import AwsCredentials, S3Bucket

@flow
def s3_flow():
    # create a dummy file to upload
    file_path = Path("test-example.txt")
    file_path.write_text("Hello, Nebula!")

    aws_credentials = AwsCredentials.load("BLOCK-NAME-PLACEHOLDER")
    s3_bucket = S3Bucket(
        bucket_name="BUCKET-NAME-PLACEHOLDER",
        aws_credentials=aws_credentials
    )

    s3_bucket_path = s3_bucket.upload_from_path(file_path)
    downloaded_file_path = s3_bucket.download_object_to_path(
        s3_bucket_path, "downloaded-test-example.txt"
    )
    return downloaded_file_path.read_text()

s3_flow()
```

### Using Nebula with AWS Secrets Manager

`nebula_aws` allows you to read and write secrets with AWS Secrets Manager within your Nebula flows.

The provided code snippet shows how you can use `nebula_aws` to write a secret to the Secret Manager, read the secret data, delete the secret, and finally return the secret data.

```python
from nebula import flow
from nebula_aws import AwsCredentials, AwsSecret

@flow
def secrets_manager_flow():
    aws_credentials = AwsCredentials.load("BLOCK-NAME-PLACEHOLDER")
    aws_secret = AwsSecret(secret_name="test-example", aws_credentials=aws_credentials)
    aws_secret.write_secret(secret_data=b"Hello, Nebula!")
    secret_data = aws_secret.read_secret()
    aws_secret.delete_secret()
    return secret_data

secrets_manager_flow()
```

## Resources

Refer to the API documentation on the sidebar to explore all the capabilities of Nebula AWS!

For more tips on how to use tasks and flows in a Collection, check out [Using Collections](https://docs.nebula.io/collections/usage/)!

### Recipes

For additional recipes and examples, check out [`nebula-recipes`](https://github.com/kozmoai/nebula-recipes).

### Installation

Install `nebula-aws`

```bash
pip install nebula-aws
```

A list of available blocks in `nebula-aws` and their setup instructions can be found [here](https://kozmoai.github.io/nebula-aws/#blocks-catalog).

Requires an installation of Python 3.7+

We recommend using a Python virtual environment manager such as pipenv, conda or virtualenv.

These tasks are designed to work with Nebula 2.0. For more information about how to use Nebula, please refer to the [Nebula documentation](https://docs.nebula.io/).

### Feedback

If you encounter any bugs while using `nebula-aws`, feel free to open an issue in the [`nebula-aws`](https://github.com/kozmoai/nebula-aws) repository.

If you have any questions or issues while using `nebula-aws`, you can find help in either the [Nebula Discourse forum](https://discourse.nebula.io/) or the [Nebula Slack community](https://nebula.io/slack).
 
Feel free to star or watch [`nebula-aws`](https://github.com/kozmoai/nebula-aws) for updates too!
