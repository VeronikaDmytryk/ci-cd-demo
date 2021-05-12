# Continuous Integration and deployment of a Node.js lambda function
This repository is for demonstration of CI/CD of a Node.js lambda function.

### Why CI/CD is used

### Prerequisites
installed Node.js on your machine
To check if you have it, run
``
node --version
``
If you don't have it, download it from  [here]("https://nodejs.org/en/download/")

### Structure of the project you are about to create
Now the structure of the project should look like this
```
git/
github/
    workflows/
        lambda-ci-cd.yaml
lambda/
    node_modules/
    tests/
        unit.test.js
    lambda.test.js
    index.js
    package.json
    .gitignore
```

## Step 1
Create a new repository on GitHub.
Clone it to your local machine
``
git clone <link-of-your-repo.git>
``
Create and switch to a new branch for your lambda
``
git branch lambda
``
``
git checkout lambda
``

## Step 2
Create a folder with the name of your lambda
> In this example, the name of the folder is 'lambda'

## Step 3
If you are writing a lambda function using Node.js, next step would be to 
initiate a node project in the newly created folder.
``
cd ./lambda
``
``
npm init
``
Fill in some information about the lambda.
> For test command type 'jest'
> Don't forget to create .gitignore file which would exclude node_modules folder from uploading to the remote repository 

## Step 4
Create your lambda function.
In lambda folder create an index.js file, and define exports.handler function in it in s similar format
```
module.exports.handler =  async function(event, context) {
    console.log("Hello world")
    return 8;
}
```

## Step 5
In the lambda folder, create your tests folder with unit tests for your function
All unit tests files has to end with **'.test.js'**. Example - functionName.test.js

## Step 6 
To run tests you need to install jest first.
``
npm install jest
``
Now you are ready to run your tests by running
``
npm run test
``
### Make sure all your test cases pass, before going to the next step.

## Step 7
Before creating the workflow, which would put our code from the repository to the AWS Cloud,
You need to prepare 2 things in an AWS account:
* an s3 bucket for your lambda's code
* a Node.js lambda function
> Save the name of the **bucket** and the **function**, you'll use it later

## Step 8
Now let's create a workflow using [Github Actions syntax]("https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions") 
Add **.github/workflows** folders to the root of the project and create a **lambda-ci-cd.yaml** file

## Step 9
Here is the content you should add to the file, with comments
```
  name: CI/CD Pipeline for Node.js lambda     # Name of the pipeline
  on:                                         
    pull_request:                             # the workflow will be triggered in the case of a
      paths:                                  # pull-request or push to the main branch
      - 'lambda/**'                           # only if something in lambda folder got changed
      branches:
          - main
    push:
      paths:
      - 'lambda/**'
      branches:
          - main
  jobs:
    # job1
    # job2
```
There are 3 sections of the workflow, which answers these questions:
* **name** - What is the name of the workflow?
* **on** - When to trigger the workflow (conditions)?
* **jobs** - What to do during the workflow?

## Step 10
Now, you will specify 2 jobs in the jobs section. 
One is for continuous integration (CI) and one is for continuous deployment (CD)
```
  jobs:
    # job1
    continuous-integration:               # Name of the job
      runs-on: ubuntu-latest              # Runner with the OS where to execute the below code
      steps:
        # step1
        # step2
    # job2
    continuous-deployment:                # Name of the job
      runs-on: ubuntu-latest              # Runner with the OS where to execute the below code
      needs: [continuous-integration]     # This job requires a SUCCESSFUL completion of the continuous-integration job
      steps:
        # step1
        # step2
```
Using Github Actions, you don't nessecarily need to have a runner (server) for testing and/or buildig our lambda (or app).
You can just specify the Operating System that you would like to use, and Github will provide a runner for us for free* (for public repositories)
But there is another option - you can specify a self-hosted runner. To find more about self-hosted runners, click [here](https://docs.github.com/en/actions/hosting-your-own-runners/about-self-hosted-runners).

## Step 11
Create tasks for the CI job.
```
    steps:
      - uses: actions/checkout@v1                           # Github  lambda.zip to be uploaded to S3
      - name: Install Dependencies                          # Install dependencies step
        working-directory: ./lambda
        run: npm install
      - name: Test                                          # Test the lambda
        working-directory: ./lambda
        run: npm run test
      - name: Create Zipfile archive of Dependencies        # Zip the lambda. Note - we create the zip file in the root of the project
        run: cd ./lambda && zip -g -r9 ../lambda.zip -r .   
      - name: Upload zip file artifact                      # Github prepares your lambda zip to be uploaded to S3
        uses: actions/upload-artifact@v2                    # Github utility we need to prepare it
        with:
          name: lambda 
          path: lambda.zip                                  # The lambda.zip in the root
```
If you are not familiar with the zip command, here are some details
* **zip** - the command to create a zip archive
* **-g**
* **-r9** - the archiving algorithm to use
* **../lambda.zip** - create a zip archive one folder above. In our case it is the root of the project
* **-r** - recursively adds files (add all the files from the subdirectories as well)
* **.** - from the current directory

## Step 12
Before implementing the Continuous Deployment job, you need to add your AWS credentials to the Github repository as secrets.
This will allow your code to be deployed straight to the AWS Cloud from the Github repository.
For that you need to go back to your Github repsitory you created at the beginning of this tutorial
Go to **Settings -> Secrets -> New repository secret**

### Create the following keys:
* **AWS_ACCESS_KEY_ID**
* **AWS_SECRET_ACCESS_KEY**
* **AWS_REGION**
* **AWS_SESSION_TOKEN** (needed only if you have expirable credentials. Ex. Using AWS Educate Account)
> You can find these credentials for your user in your AWS management console - IAM. 
> If you are using AWS Educate account you should find your account details on Vocareum.com

## Step 13
Create tasks for the CD job.
```
      # Step 1
      - name: Install AWS CLI                                           # Install AWS CLI on Github's server
        uses: unfor19/install-aws-cli-action@v1
        with:
          version: 1
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}           # Provide your AWS credentials
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_SESSION_TOKEN: ${{ secrets.AWS_SESSION_TOKEN }}           # See notes* below
          AWS_DEFAULT_REGION: ${{ secrets.AWS_REGION }}
      # Step 2
      - name: Download Lambda lambda.zip                                # Download the zip folder that you prepared in the first job
        uses: actions/download-artifact@v2
        with:
          name: lambda                                                  # The name of the zip folder here
      # Step 3 - would need to update
      - name: Upload to S3                                              # Upload the zip folder to S3 bucket. It copies the zip folder to the S3 bucket
        run: aws s3 cp lambda.zip s3://<your-s3-bucket-name-here>/lambda.zip
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_SESSION_TOKEN: ${{ secrets.AWS_SESSION_TOKEN }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_REGION }}
      # Step 4 - would need to update
      - name: Deploy new Lambda                                         # Deploy the code from the S3 to the lambda
        run: aws <your-lambda-name-here> update-function-code --function-name <your-aws-lambda-name-here> --s3-bucket <your-s3-bucket-name-here> --s3-key lambda.zip
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_SESSION_TOKEN: ${{ secrets.AWS_SESSION_TOKEN }}
          AWS_DEFAULT_REGION: ${{ secrets.AWS_REGION }}
```
### Replace 
***<your-s3-bucket-name-here>*** with the s3 bucket that you created in the AWS earlier
***<your-aws-lambda-name-here>*** with the lambda name that your created in the AWS earlier
> \* You only need to provide aws_session_token if you are using expirable credentials

## Step 14
Now you can commit our changes and push it to our remote repository
``
git add *
``
``
git commit -m "Created a lambda function and added a CI/CD workflow"
``
``
git push origin lambda
``

## Step 15
Create a pull request for **lambda** branch to merge **main**.
> If you are using expirable credentials, please update those before creating a pull request.

This will trigger our workflow.
You can check how it goes in the Checks tab
If everything went successfully, you will see green checkmark next to both of our jobs.

If something failed (ex. expirable credentials), you can re-run jobs by clicking on the **'Re-run jobs'**

##### Congratulations with your first Github Actions CI/CD workflow