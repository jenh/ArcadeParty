# Arcade Party for Amazon Alexa - NodeJS

This is a basic port of Arcade Party, an Alexa skill that streams [Andy Hofle's Arcade Ambience project](http://www.arcade.hofle.com/) files from [archive.org](https://archive.org/details/ArcadeAmbience). 

I originally wrote [Arcade Party](https://amzn.to/33OONuQ) in Python with Flask-Ask, using Redis to save user/session state, and it's hosted on an EC2 server running Apache and Python WSGI. It was a quick turnaround project to get familiar with Alexa skill development before I wrote the more complicated  [Radio Fun Time](https://amzn.to/3kBM8uy). It turned out the be much more popular than Radio Fun Time.

This project is a quick, initial prototype for an Arcade Party clone using NodeJS, hosted on AWS Lambda with DynamoDB to save user/session state and as such, it's missing some features (like choosing a track!), while it inherits other features from the  [Amazon Alexa NodeJS Audio Player Sample](https://github.com/alexa/skill-sample-nodejs-audio-player/tree/mainline/multiple-streams) I adapted it from, like looping and shuffle. These are initial steps towards porting something more complicated (like Radio Time Warp) and my intention is to share the code and tutorial so that anyone can get a simple audioplayer skill for Alexa up and running quickly.

This README walks you through a complete Alexa skill deployment using the ASK and AWS command line interface. 

- [Prerequisites](#prerequisites)
- [Create ASK and AWS User Profiles](#createaskandawsuserprofiles)
- [Prepare the skill infrastructure](#infrastructure)
- [Configure DynamoDB access](#configuredynamodbaccess)
- [Customize your skill](#customizeyourskill)
- [Clean up unused resources](#cleanup)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Known Issues](#knownissues)
 
If you find this demo and tutorial helpful, please consider contributing to [Archive.org](https://archive.org/donate). Don't forget to try out the official Arcade Party skill on Alexa and, if you're a big fan of classic video games, be sure to check out [8bitworkshop.com](https://8bitworkshop.com), where you can write and prototype games for classic video game systems in your browser.



## Prerequisites<a name="prerequisites"></a>

Before you begin, you'll need the following:

 - [An Amazon AWS account](http://aws.amazon.com)
- [Node.js and npm](https://www.npmjs.com/get-npm) installed
 - The latest ASK CLI [installed and configured](https://developer.amazon.com/en-US/docs/alexa/smapi/quick-start-alexa-skills-kit-command-line-interface.html) 
 - The latest AWS CLI [installed](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html) and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)
 - [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)


## Create ASK and AWS User Profiles<a name="createaskandawsuserprofiles"></a>

If this is your first time deploying a skill using the ASK SDK v2 with Node.js command line interface, you should re-run `ask configure` to generate both a new IAM user and profile with the permissions you'll need to deploy. Even if it's not your first skill deployment, creating a new profile helps ensure that you don't run into permissions problems later.

**To create and use a new IAM account for ASK:**

1. At the command line, run:

    ```
    ask configure
    ```
2. When prompted to select an ASK profile, select **Create New Profile**.

3. Provide a name you'll remember, like `alexa-deploy`.

4. When prompted, a browser window will appear and prompt you to log in. 

5. After successfully logging in, return to your command line, type **Y** and press Enter to link the new account to your AWS account.

6. When prompted to select an AWS profile, select **Create New Profile**.

7. Provide an AWS profile name, for example, `AWS_Profile_for_Alexa_Deploy` and press Enter.

    Another browser window will open to the IAM Dashboard, listing the permissions summary. This should include IAMFullAccess, AWSCloudFormationFullAccess, AmazonS3FullAccess, and AWSLambdaFullAccess. 

8. Click **Create User**.

9. Copy and save the Access key ID and Secret access key, then return to the terminal to supply them to the ASK configuration wizard. 

## Prepare the skill infrastructure<a name="infrastructure"></a>

Now that you have prerequisites installed and configured, you can clone the project, and prepare the skill infrastructure for your first deployment.

1. Clone the project onto your system:

    ```
    $ git clone https://github.com/jenh/ArcadeParty
    ```

2. Change into the `Arcade_Party` directory (`cd Arcade_Party`) and customize the following files:

    - `lambda/index.js`: Customize your prompts. If you're new to Node.js, you should skip this step and return to it after you deploy and test the skill for the first time and have verified that it's running successfully.
   
    - `lambda/constants.js`: Optionally, specify a new `dynamoDBTableName`, and add links and titles for the mp3s you want to use. Leave the Alexa skill ID empty for now as we won't have one until we deploy.

    - `lambda/package.json`: You can change the name and author here (the name controls the autogenerated Lambda function name, so if you're picky about that, change it here), but otherwise skip editing this file on first pass and update after you start hacking `install.js`.  
    
    - `policies/DynamoDBSecureTable`: If you changed the database name in `constants.js`, make sure it matches here. For example, if you changed the database to `Arcade_Party_DB` in `constants.js`, the `Resource` should be set to `"arn:aws:dynamodb:*:*:table/Arcade_Party_DB`.
    
    - `skill-package/skill.json`: Change metadata as needed for your project. Update the icons to your own icons. The arcade photo used here is courtesy Rob Boudon via Wikimedia Commons ([CC BY 2.0](http://creativecommons.org/licenses/by/2.0)).
    
    - `skill-package/interactionModels/custom/en-US.json`: Change invocationName (this must be entered in lower case) and, optionally, add or remove sample utterances as needed. As with `index.js`, it's a good idea to wait until the skill is deployed and working to customize utterances.
    

3. Initiate the Alexa skill project:

    ```
    $ ask init
    ```

4. When prompted, you can accept the default values. If you don't expect a lot of usage, you can choose **No** for CloudFormation ([CloudFormation is free for Alexa skills](https://aws.amazon.com/cloudformation/pricing/)), and you may also want to change **Lambda runtime** to `nodejs12.x`. 

5. Review the policy that results, then type **Y** and press Enter to finish.

6. Install npm dependencies in the `lambda` folder, then return to the main directory:

    ```
    $ cd lambda && npm install && cd ..
    ```

7. You're now ready to deploy the first iteration of your Alexa skill. The following command will create a new Alexa skill, package and upload your NodeJS to Lambda as a new Lambda function, and create an execution role for your function:

    ``` 
    $ ask deploy

    ==================== Deploy Skill Metadata ====================
    Skill package deployed successfully.
    Skill ID: amzn1.ask.skill.[your_skill_uuid]

    ==================== Build Skill Code ====================
    npm WARN arcade-party-node-lambda@2.0.0 No repository field.

    audited 256 packages in 1.758s
    found 0 vulnerabilities

    Skill code built successfully.
    Code for region default built to /home/jfhh/Arcade_Party/.ask/lambda/build.zip successfully with build flow nodejs-npm.

    ==================== Deploy Skill Infrastructure ====================
      ✔ Deploy Alexa skill infrastructure for region "default"
      The api endpoints of skill.json have been updated from the skill infrastructure deploy results.
    Skill infrastructures deployed successfully through @ask-cli/cfn-deployer.

    ==================== Enable Skill ====================
    Skill is enabled successfully.

    ```   

7. Copy the skill ID that results (this should look like `amzn1.ask.skill.fae40d8e-5a83-49d6-8864-71bdb8fc2ead`) into the `appID` attribute in `lambda/constants.js`.

    Note: If you closed the terminal without copying the skill ID, you can list your skills at any time by running:

    ```
    ask smapi list-skills-for-vendor
    ```
8. Re-deploy the skill:

    ```
    ask deploy
    ```


## Configure DynamoDB access<a name="configuredynamodbaccess"></a>

The skill and Lambda function are now deployed to the AWS Cloud, but it won't work just yet. The Lambda function needs permission to access DynamoDB in order to create and update the table that holds user metadata (user and session IDs and state, including current track, audio offset, and playback preferences). Otherwise, the skill will fail on launch.

To do this, we create a security policy that allows DynamoDB access and attach it to the IAM role that the `aws deploy` command auto-generated for our Lambda function. 

This project contains a JSON file that allows DynamoDB access, but restricts permissions to the table specified inside the policy file. While it's common for developers to give Lambda functions DynamoDBFullAccess permissions, it's just as easy (and good practice) to configure more restrictive permissions.
    
**To generate and attach the table-specific DynamoDB policy**:

1. Double-check that the Resource in `policies/DynamoDBSecureTable.json` matches what you set in `lambda/constants.js`. If it doesn't match, edit it within `policies/DynamoDBSecureTable.json`. If you decide to change `constants.js` instead, be sure to run `aws redeploy` again. If you run the following `grep` command, the output should look like the following, where `ap_demo` is the database name you are using:

    ```
    $ grep Resource.*arn policies/DynamoDBSecureTable.json && grep dynamo lambda/constants.js 
    
    "Resource": "arn:aws:dynamodb:*:*:table/ap_demo"
    dynamoDBTableName: 'ap_demo',
    ```

2. Create the policy. Note that the ``file://`` uri inside the `policy-document` argument is very important and the command will fail if you omit it. 

    ```
    $ aws iam create-policy --policy-name ap_demo_dynamodb --policy-document file://policies/DynamoDBSecureTable.json


    {
        "Policy": {
            "PolicyName": "ap_demo_dynamodb",
            "PolicyId": "123456890ABCCDEFGX",
            "Arn": "arn:aws:iam::12345689012:policy/ap_demo_dynamodb",
            "Path": "/",
            "DefaultVersionId": "v1",
            "AttachmentCount": 0,
            "PermissionsBoundaryUsageCount": 0,
            "IsAttachable": true,
            "CreateDate": "2020-08-11T03:26:01+00:00",
            "UpdateDate": "2020-08-11T03:26:01+00:00"
        }
    }

    ``` 
    
    If you encounter any errors, add `--debug` to obtain troubleshooting information.

3. Make a note of the `Arn` attached to the newly-created policy, we'll need it to attach the policy.


4. Next, we'll locate the role that requires the policy. This role was created by the initial `ask deploy` and is typically the most recently-created role (and you should see your NodeJS package name from `lambda/`), so instead of listing all roles with `aws iam list-roles`, we'll add the ``--query`` option to locate the newest role quickly:

    ```
    $ aws iam list-roles --query 'reverse(sort_by(Roles[].[CreateDate,RoleName], &[0]))[0]' --output text

    > 2020-08-11T00:08:26+00:00       ask-DemoArcadePartyLambda-default-skillStack-15-AlexaSkillIAMRole-1234567890ABC
    ``` 
     Tip: If your organization has a large number of roles that are updated frequently, you can print a list of the newest roles, where YYYY-MM-DD is today's date:

    ```
    $ aws iam list-roles --query 'Roles[?CreateDate>=`YYYY-MM-DD`].[RoleName,CreateDate]' --output text
    ```

5. We can now attach the policy to our role using the policy Arn and role:

    ```
   $ aws iam attach-role-policy --role-name ask-DemoArcadePartyLambda-default-skillStack-15-AlexaSkillIAMRole-123456890AC --policy-arn arn:aws:iam::012345689012:policy/ap_demo_dynamodb
    ```

6. You're now ready to test the skill! You can test on the command line or [Developer Console](https://developer.amazon.com/alexa/console), but it's more fun for the first try to use an Alexa device or Alexa mobile app: try it out using the invocation you set in the `en_US.json` model file by saying "Alexa, open *invocationName*." 

    
## Customize your skill<a name="customizeyourskill"></a>

You can now open `lambda/index.js` and customize your skill even further. Remember to run `ask deploy` to update your skill after any change.

## Clean up unused resources<a name="cleanup"></a>

Because deploying skills this way is pretty speedy, you may find yourself deploying a number of them for testing purposes. Because DynamoDB incurs billing charges, you'll want to delete DynamoDB tables and it's also good practice to delete resources you don't need, like the Lambda function and role. You can do this from the AWS console or from the command line. 

**To delete unused DynamoDB tables using the AWS CLI:**

1. Find the table(s) you want to delete by showing all available tables:

    ```
    $ aws dynamodb list-tables
    ```
2. Delete the table. Note that you don't need the full resource URI, you should be able delete with the short table name:

    ```
    $ aws dynamodb delete-table --table-name YOUR_TABLE_NAME
    ```

**To delete unused Lambda functions using the AWS CLI:**

1. Find the Lambda function you want to delete by showing all available functions:

    ```
    $ aws lambda list-functions
    ```
2. Delete the function, you can use the Function name or Arn:

    ```
    aws lambda delete-function --function-name ask-ap_demo-default-default-12345678901
    ```

Note: Be careful here! Make sure you are deleting the correct function!

**To delete unused IAM roles using the AWS CLI:**

Deleting roles is a little more involved, you need to detach all policies from the role before deleting.

1. Find the role you want to delete by showing all available functions:

    ```
    $ aws iam list-roles
    ```
2. Locate the attached policies for the role:

    ```
    $ aws iam list-attached-role-policies --role-name ap_demo

    {
      "AttachedPolicies": [
        {
          "PolicyName": "ap_demo_dynamodb",
          "PolicyArn": "arn:aws:iam::123456789012:policy/ap_demo_dynamodb"
        }
      ]
    }
    ```

3. Detach the policy (or policies, if applicable):

    ```
    $ aws iam detach-role-policy --role-name "ask-apdemo-default-skillStack-15-AlexaSkillIAMRole-1DCKB83JKAELG" --policy-arn "arn:aws:iam::123456789012:policy/ap_demo_dynamodb"
    ```
4. Delete the role, you can use the Function name or Arn:

    ```
    aws iam delete-role --role-name ask-apdemo-default-skillStack-15-AlexaSkillIAMRole-1DCKB83JKAELG
    ```

Note: Be careful here! Make sure you are deleting the correct function!


## Testing<a name="testing"></a>

There are a number of ways to test your skill, and you'll probably find yourself using every one of them.

You can test from an IDE (you can install the AWS extension for Visual Studio and test directly from the IDE), from the [Lambda console](https://console.aws.amazon.com/lambda), from a device (an Alexa device or the Alexa app on a mobile device), and from the [Alexa Development Console](https://developer.amazon.com/alexa/console/ask).

For specific errors in NodeJS, you may need to test directly from the Lambda UI, an IDE, or the command line. To generate test cases, the Alexa Development Console is really handy, as you can type your queries, then copy the Javascript out into test cases. If you share your test cases or check them into Git, be sure to change any identifiers first (Tip: the `uuidgen` tool on Linux is handy for generating fake skill IDs).

**To run a test from the command line:**

1. To test your Lambda function at the command line, you can use the following command:

    ```
    aws lambda invoke --function ask-mylambdafunction-AlexaSkillFunction-ABCDEFGHI11 --payload file://tests/mytestfile.json output.log && cat output.log
    ```
## Troubleshooting<a name="troubleshooting"></a>
 
I've included a list of helpful commands that I used when porting this skill and issues I ran into.

### Viewing CloudWatch logs on the command line

Sometimes you don't have the patience to wait for the CloudWatch console to reload, and it's faster and more fun to tail logs at the command line as they come in. You can use `aws logs` to do this.

**To tail logs**:

1. List the log groups available:
    ```
    $ aws logs describe-log-groups
    ```
2. Run tail for the log group you want to track, and you can use `follow` to watch log events as they arrive (like `tail -f`):

    ```
    aws logs tail /aws/lambda/ask-ArcadePartyNodeJSonLambdaEd-AlexaSkillFunction-ABCD789ASD12 --follow
    ```

### Common errors 

I tried to configure this project so that you shouldn't run into these issues, but you may well encounter them here or in other projects.

***"The trigger setting for the Lambda function is invalid."***

If you changed the Lambda function in the manifest and redeployed, even adding the Alexa Skills Kit trigger and skill ID after-the-fact might not 'take.' I was able to work around this by going into the Alexa Console and manually updating the endpoint.

Then pull the manifest down from Amazon & copy it over (make a backup first!) your old skills.json:

```
ask smapi get-skill-manifest -s amzn1.ask.skill.YOUR_SKILL_UUID -g development >> new-skills.json
```
Or just start from scratch! 

***"String instance at property path "$.manifest.apis.custom.endpoint.uri\" does not match the regular expression: "^(arn|https://)"***

This and the *Unable to deploy target skill-infrastructure when the skillId has not been created yet* error are a chicken and egg problem I struggled with for awhile: many thanks to [Jose Quinto, who documented the solution](https://blog.josequinto.com/2018/04/25/deploy-alexa-skill-using-already-created-lambda-function-and-role/#Deploy-the-skill). You just need to clear the entire `endpoint` option from your manifest and start a fresh deployment. ASK will add it back after it creates everything. 

Be careful if you do this a lot; you may find yourself with multiple Lambda functions and might discover that you're testing a stale function! (No wonder your changes didn't take!) Check the manifest to make sure that it matches what you're testing or what you think you're testing, especially if you test Lambda functions directly.

I removed the `endpoint` option from the `skills.json` file I include in this project, but if you copy skill manifests around like most of us do, you will probably run into the error again!


### Cloning Your Skill

`ask` 2.0 no longer has a cloning option, but you can access the same functionality creating and changing into a new directory, running `list-skills-for-vendor` to obtain a list of skills and then running the `ask smapi export-package` command:
    
```
$ ask smapi list-skills-for-vendor

   {
      "_links": {
        "self": {
          "href": "/v1/skills/amzn1.ask.skill.59d694ce-dd0e-4d52-9853-493c0f131b0f/stages/development/manifest"
        }
      },
      "apis": [
        "custom"
      ],
      "asin": "A123456789",
      "lastUpdated": "2020-01-13T01:51:03.091Z",
      "nameByLocale": {
        "en-AU": "Arcade Party",
        "en-GB": "Arcade Party",
        "en-IN": "Arcade Party",
        "en-US": "Arcade Party"
      },
      "publicationStatus": "DEVELOPMENT",
      "skillId": "amzn1.ask.skill.59d694ce-dd0e-4d52-9853-493c0f131b0f",
      "stage": "development"
    },

$ ask smapi export-package -s amzn1.ask.skill.59d694ce-dd0e-4d52-9853-493c0f131b0f -g development
```

## Known Issues and "to do"s<a name="knownissues"></a>

- The current behavior is not identical to the released version of Arcade Party (Python/WSGI on Apache). This is currently a hastily put-together proof-of-concept port and a personal project to get more comfortable with Lambda and NodeJS.

- Add test cases.

- Add Python WSGI code and instructions (I started working on the Python tutorial last year and got pulled away, so the code is complete, the Apache configuration instructions are about 75% complete, so it shouldn't take much to add them).

- Port Radio Time Warp/Radio Fun Time.
