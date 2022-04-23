# hello-cloudrun

This is a complete walkthrough for how to get a (minimal) Dockerised website (using Flask) up and running on Google Cloud Run. The instructions include setting up a (ci)cd pipeline in Github Actions that securely deploys your project to Google whenever you push new code to the main branch on Github.

It does _not_ cover how to terraform resources, e.g., a database, or anything else that you'd normally include in a real production application. Nor how to set up unit testing in the pipeline.

This is a more secure reimplementation of my previous hello-cloudrun walkthrough which used downloaded key credentials. This walkthrough instead uses [Workload Identity Federation](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions) with the [google-github-actions/auth](https://github.com/google-github-actions/auth) Github Action to authenticate. See [setup](https://github.com/google-github-actions/auth#setup) for alternative instructions.

When you are done, you will have:

- A running web service deployed to Google Cloud Run
- Github Actions pipeline that automatically deploys code changes to Google

## Complete walkthrough

This walkthrough will mostly use of the Cloud Shell (icon looks like `>_` in the Google Cloud console menu). The same steps can be
performed in the browser or the [gcloud](https://cloud.google.com/sdk/docs/install) command-line tool. There are quite a few steps, but you only have to perform them once.

### Create cloud account and project

In browser:

1. Create an account (or use an existing one) and login to  Google Cloud account: https://cloud.google.com/
1. Create a new project with name 'hello-cloudrun' (or whatever you prefer); select it in the console.
1. **Important**: Note down the auto-assigned project ID, e.g. 'hello-cloudrun-1234'. Don't worry, you can always find it later, but you will need it soon :-)

### Configure project

In the Cloud Shell (or terminal):

Login:
```bash
gcloud auth login  #terminal only
```

Set environment variables that will be used later:
```bash
export PROJECT_ID='hello-cloudrun-1234'  # use the actual project ID
export ACCOUNT_NAME='hello-cloudrun-sa'
```

Select the project:
```bash
gcloud config set project $PROJECT_ID
```

Enable required APIs in Google cloud:
```bash
gcloud services enable \
cloudbuild.googleapis.com \
run.googleapis.com \
containerregistry.googleapis.com
```

Create a service account that will own the running process:
```bash
gcloud iam service-accounts create $ACCOUNT_NAME \
  --description="Account that manages services in $PROJECT_ID" \
  --display-name="service-account-$PROJECT_ID"
```

Assign role `run.admin` to service account:
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/run.admin
```

Assign role `storage.admin` to service account:
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/storage.admin
```

Assign role `iam.serviceAccountUser` to service account:
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountUser
```

### Configure connection to Github

To enable Github to deploy to Google Cloud, we will now configure [Workload Identity Federation](https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions). The steps are replicated from [google-github-actions setup instructions](https://github.com/google-github-actions/auth#setup).

In Google Cloud (or terminal):

Create some more environment variables, this time for the Workload Identity Federation:
```bash
export POOL_NAME='hello-cloudrun-pool'
export PROVIDER_NAME='hello-cloudrun-provider'
export REPO='skipperkongen/hello-cloudrun' # should match you Github repo
```

Enable the IAM Credentials API:
```bash
gcloud services enable iamcredentials.googleapis.com \
 --project $PROJECT_ID
```

Create a Workload Identity _pool_:
```bash
gcloud iam workload-identity-pools create $POOL_NAME \
 --project=$PROJECT_ID \
 --location="global" \
 --display-name="Identity pool"
```

Get the full ID of the Workload Identity pool:
```bash
gcloud iam workload-identity-pools describe $POOL_NAME \
 --project=$PROJECT_ID \
 --location="global" \
 --format="value(name)"
```
Save this value as an environment variable:
```bash
export WORKLOAD_IDENTITY_POOL_ID='...'  # value from above
```

Create a Workload Identity _Provider_ in that pool
```bash
gcloud iam workload-identity-pools providers create-oidc $PROVIDER_NAME \
  --project=$PROJECT_ID \
  --location="global" \
  --workload-identity-pool=$POOL_NAME \
  --display-name="Provider for pool" \
  --attribute-mapping="google.subject=assertion.sub,attribute.actor=assertion.actor,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

Allow authentications from the Workload Identity Provider originating from your repository to impersonate the Service Account created above:
```bash
gcloud iam service-accounts add-iam-policy-binding "${ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project=$PROJECT_ID \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WORKLOAD_IDENTITY_POOL_ID}/attribute.repository/${REPO}"
```

Extract the Workload Identity Provider resource name:
```bash
gcloud iam workload-identity-pools providers describe $PROVIDER_NAME \
  --project=$PROJECT_ID \
  --location="global" \
  --workload-identity-pool=$POOL_NAME \
  --format="value(name)"
```

Place this value in the Github secret GCP_WORKLOAD_IDENTITY_PROVIDER (see below).

### Configure Github secrets

Create the following secrets in your Github repository. They are not actually sensitive secrets, but I find it neat to reference the values from Github actions by variable names.
  - **GCP_APP_NAME**: The name of your app, e.g., 'hello-cloudrun', used in the Docker image tag.
  - **GCP_PROJECT_ID**: The ID of your project. It's the value you stored in `$PROJECT_ID`. You can also find it in the Google Cloud console.
  - **GCP_WORKLOAD_IDENTITY_PROVIDER**: It's the value you found in the last step of the previous section
  - **GCP_SERVICE_ACCOUNT_EMAIL**: The email of the service account you have created. You can find this value in the Google Cloud console under IAM -> Service Account.
