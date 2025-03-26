# MyRecipeBook Full-Stack Application Deployment Guide

This guide provides step-by-step instructions for deploying the complete full-stack application, including infrastructure, server, and web.

## Project Structure

- **infra**: Terraform infrastructure code (AWS resources)
- **server**: Backend API application
- **web**: Frontend application

## Prerequisites

- AWS CLI installed and configured
- Node.js and npm installed
- Git
- GitHub account with access to this repository

## Deployment Steps

### 1. Infrastructure Setup (infra submodule)

#### 1.1 Initial AWS Configuration

1. Navigate to the infra directory

2. Review the `1-admin/main.tf` file to understand the required AWS console configuration:
   1. create the s3 bucket to manage terraform state
   2. create and attach to the AWS ADMIN profile the initial policy

3. Create a `.secrets` file based on the provided example:
```
cp .secrets.example .secrets
```

4. Edit the `.secrets` file with your AWS credentials and configuration:

* AWS ADMIN is the aws profile that is going to give the AWS RESOURCE CREATOR profile permission to create all resources needed by this app
* AWS RESOURCES CREATOR is the aws profile that is going to create this resources


5. Export the secrets to base64 and create and add them to a variable in github secrets called ENCODED_SECRETS:

```
base64 -i .secrets
```

#### 1.2 Deploy Admin and Resources Infrastructure (Step 1 and 2)
On commit or using some tool to run the workflow locally (i. e: act), the '1-deploy.yml' will run the steps defined in the folder 1-admin and 2-resources
If you using act on mac, this is the command to run the workflow locally. Run it in the root of the infra folder:

act -s ENCODED_SECRETS="$(base64 -i .secrets | tr -d '\n')" --container-architecture linux/amd64


After successful deployment, note down the following workflow outputs:
- *ECR Repository URL*
- *RDS Endpoint*
- *S3 Bucket Name*
- *SQS Delete User URL*

### 2. Server Configuration and Deployment

#### 2.1 Configure Server Settings

1. Navigate to the server directory
2. Fill the variables from step 1.2 in the corresponding file:

| Name              | Variable                                   | File                                            |
|-------------------|--------------------------------------------|-------------------------------------------------|
| RDS endpoint      | ConnectionString.DefaultConnection (Host)  | MyRecipeBook.API/appsettings.Production.json    |
| S3 bucket name    | AWS_S3_BUCKET_NAME                         | MyRecipeBook.Infrastructure/Infrastructure.env  |
| SQS url           | AWS_SQS_DELETE_USER_URL                    | MyRecipeBook.Infrastructure/Infrastructure.env  |
| ECR url           | AWS_ECR_URL                                | MyRecipeBook.Infrastructure/Infrastructure.env  |

3. Set up the remaining variables of the backend:
   Set the environment variables of the corresponding files using the examples:

| Example File                                           | Real File                                       |
|--------------------------------------------------------|-------------------------------------------------|
| MyRecipeBook.API/appsettings.Development.json          | MyRecipeBook.API/appsettings.Production.json    |
| MyRecipeBook.API/API.Example.env                       | MyRecipeBook.API/API.env                        |
| MyRecipeBook.Infrastructure/Infrastructure.Example.env | MyRecipeBook.Infrastructure/Infrastructure.env  |


#### 2.2 Configure GitHub Secrets for CI/CD

1. Create the following secrets in GitHub and paste the contents of the respective file:

   | Github repository secret | Content of the file                            |
   |--------------------------|------------------------------------------------|
   | API_ENV                  | MyRecipeBook.API/API.env                       |
   | INFRA_ENV                | MyRecipeBook.Infrastructure/Infrastructure.env |

#### 2.3 Deploy Server to ECR

1. Push to the repository to trigger the GitHub workflow, or manually run the workflow from the Actions tab.
2. The workflow will build the Docker image and push it to the ECR repository.

### 3. Deploy AppRunner (Step 3)

After running the deployment workflow and deploying the backed, go back to the infra repository and run the app runner workflow manually in GitHub:
1. Go to your GitHub repo → Click on "Actions". On the left sidebar, locate "app-runner.yml".
2. Click "Run workflow" (usually a dropdown button) and confirm.
3. After successful deployment, note down the AppRunner service URL from the outputs.

### 4. Web Configuration and Deployment

#### 4.1 Configure Web Environment

1. Navigate to the web directory

2. Update the `.env.production` file with the AppRunner URL:
```
NEXT_PUBLIC_API_URL=https://your-apprunner-service-url
```

3. Generate the api function from orval for the production environment, and push it to the repository

```
npm run generate-api:prod
```

#### 4.2 Deploy Web to AWS Amplify

1. Log in to the AWS Management Console.
2. Navigate to AWS Amplify.
3. Click "New app" → "Host web app".
4. Connect to your GitHub repository.
5. Configure the build settings as required.
6. Deploy the application.
7. Note down the Amplify application URL once deployment is complete.

### 5. Finalize Server Configuration

1. Update the `appsettings.Production.json`. In the ExternalLogin:AllowedReturnUrls sections with the Web URL:
```
{
     "ExternalLogin": {
      "AllowedReturnUrls": [
        "/",
        "/home",
        "/dashboard",
        "/logout",
        "test.org",
        "http://localhost:3000/redirect-after-login",
        "https://your-amplify-web-url" <---- ####  ADD THIS LINE  ####
      ]
}
```

2. Trigger a new server deployment to apply the changes.

### 6. Finalize Web Configuration

1. Update the `.env.production` file with all required values:
   ```env
   # Backend API URL (from AppRunner deployment)
   NEXT_PUBLIC_API_URL=https://your-apprunner-service-url

   # Google OAuth Configuration
   GOOGLE_CLIENT_ID=your_google_client_id
   GOOGLE_CLIENT_SECRET=your_google_client_secret

   # NextAuth Configuration
   NEXTAUTH_URL=https://your-amplify-web-url <---- #### ADD THIS LINE ####
   NEXTAUTH_SECRET=your_nextauth_secret
   ```

2. Generate the API types and functions for production:
   ```bash
   npm run generate:prod
   ```

3. Commit and push these changes to trigger a new deployment:
   ```bash
   git add .env.production src/api
   git commit -m "Update production configuration with final URLs"
   git push
   ```

4. Wait for AWS Amplify to complete the deployment.

5. Verify that the application is working correctly by:
   - Visiting the Amplify URL
   - Testing the authentication flow
   - Ensuring API communication is functioning properly


## CI/CD Information

The application has continuous integration and deployment set up:

- **Server**: Any push to the server directory will trigger the GitHub Actions workflow that builds and deploys the new image to ECR. AppRunner will automatically detect the new image and deploy it.

- **Web**: Any push to the web directory will trigger AWS Amplify to build and deploy the new version of the web application.

## Troubleshooting

- For server deployment issues, check the GitHub Actions workflow logs.
- For AppRunner issues, check the AWS AppRunner console and service logs.
- For web deployment issues, check the AWS Amplify build logs.


## Destroy workflow:
1. In the infra submodule, execute workflows/destroy.yml manually to destroy all infra and server resources.
2. In AWS Amplify, App settings, click on Delete app
3. Manually delete the s3 bucket and the initial policy created in step 1.1.2


## Development Workflow

### Handling Changes in Submodules

This project uses Git submodules to manage the three main components. When making changes, follow these guidelines:

#### Single Submodule Changes
If you have changes in only one submodule (e.g., only frontend changes):
1. Navigate to the submodule directory:
   ```bash
   cd web  # or server or infra
   ```
2. Commit and push your changes:
   ```bash
   git add .
   git commit -m "Your commit message"
   git push
   ```
3. Return to the main repository and update the submodule reference:
   ```bash
   cd ..
   git add web  # or server or infra
   git commit -m "Update web submodule reference"
   git push
   ```

#### Multiple Submodule Changes
If you have changes in multiple submodules (e.g., frontend and infrastructure):
1. Commit and push changes in each submodule separately
2. Return to the main repository and update all submodule references in a single commit:
   ```bash
   git add web infra  # or any combination of submodules
   git commit -m "Update submodule references"
   git push
   ```

> **Note**: Always commit changes in the submodules first, then update the references in the main repository. This ensures that the submodule references point to valid commits.
