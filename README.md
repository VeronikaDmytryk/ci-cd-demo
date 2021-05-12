# ci-cd-demo
This repository is for demonstration of CI/CD of a Node.js lambda function.

## Step 1
Create repository
## Step 2
Add .github/workflows folders to the root of the project
## Step 3
Create a folder with the name of your lambda

## Step 4
If you are writing a lambda function using Node.js, next step would be to 
initiate a node project
```
cd ./lambda
npm init
```
Fill in some information about the lambda

## Step 5
Create your lambda function.
In lambda folder create an index.js file, and define exports.handler function in it in a  similar a format
```
exports.handler =  async function(event, context) {
  console.log("EVENT: \n" + JSON.stringify(event, null, 2))
  return context.logStreamName
}
```


### Structure of the project
Now the structure of the project should look like this
git/
github/
    workflows/
lambda

