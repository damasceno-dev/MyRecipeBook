# MyRecipeBook Full-Stack Application Deployment Guide

This guide provides step-by-step instructions for deploying the complete full-stack application, including infrastructure, server, and web components.

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
* AWS RESOURCES CREATOR is the aws profile that is going to create the resources
> This segregation minimizes the permissions needed and improves security making the configuration needed by the root profile be minimal 

5. Export the secrets to base64 and create and add them to a variable in github secrets called ENCODED_SECRETS:

```
base64 -i .secrets
```

#### 1.2 Deploy Admin and Resources Infrastructure (Step 1 and 2)
On commit or using some tool to run the workflow locally (i. e: act), the '1-deploy.yml' will run the steps defined in the folder 1-admin and 2-resources.
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

2. Update the `.env.production` file with the AppRunner URL from step 3:
```env
# Backend API URL (from AppRunner deployment)
NEXT_PUBLIC_API_URL=https://your-apprunner-service-url

# Frontend Application URL (your Amplify URL)
NEXT_PUBLIC_AUTH_URL=https://your-amplify-web-url  # You are updatig this after Amplify deployment

# Google OAuth Configuration (obtained from backend project)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret

# NextAuth Configuration
NEXTAUTH_URL=https://your-amplify-web-url  # Must match NEXT_PUBLIC_AUTH_URL
NEXTAUTH_SECRET=your_nextauth_secret  # Required for secure authentication
```

3. Generate a secure NEXTAUTH_SECRET using one of these methods:
```bash
# Method 1: Using openssl
openssl rand -base64 32

# Method 2: Using node
node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
```

4. Push the repository to GitHub.

#### 4.2 Deploy in AWS Amplify

1. **Initial Deployment**
   - Log in to the AWS Management Console
   - Navigate to AWS Amplify
   - Click "Deploy an app"
   - Choose your source code provider (GitHub) and select your branch
   - Configure build settings:
     ```
     App name: myRecipeBook
     Framework: Next.js (auto-detected)
     Build command: npm run build
     Build output directory: .next
     ```

2. **Environment Variables Setup**
   - Before the deployment, at the step App Settings, in Advanced Settings
   - Add the following environment variables with your initial values:
     ```
     NEXT_PUBLIC_API_URL=https://your-apprunner-service-url
     GOOGLE_CLIENT_ID=your_google_client_id
     GOOGLE_CLIENT_SECRET=your_google_client_secret
     NEXTAUTH_SECRET=your_nextauth_secret
     ```

3. **Update URLs After Deployment**
   - Once the deployment is complete, note down your Amplify application URL
   - Add the following environment variables in AWS Amplify:
     ```
     NEXT_PUBLIC_AUTH_URL=https://your-amplify-web-url
     NEXTAUTH_URL=https://your-amplify-web-url
     ```
   - Click "Save" to trigger a new deployment with the updated URLs
   - Update your local `.env.production` with the same URLs for reference

> **Note**: The `.env.production` file is only used for local development reference and to generate the api routes with orval. 
> All production environment variables are managed through AWS Amplify's environment variables settings.

### 5. Finalize Server Configuration

1. Update the `appsettings.Production.json`. In the ExternalLogin:AllowedReturnUrls sections with the Web URL:
```json
{
     "ExternalLogin": {
      "AllowedReturnUrls": [
         "/",
         "/home",
         "/dashboard",
         "/logout",
         "test.org",
         "http://localhost:3000/redirect-after-login", 
         "https://your-amplify-web-url/redirect-after-login" # Add this line
      ]
}
```

2. Trigger a new server deployment to apply the changes.

### 6. That's it! Your application should be up and running
Verify that the application is working correctly by:
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
3. Manually delete the s3 bucket **_and_** the initial policy created in step 1.1.2


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
