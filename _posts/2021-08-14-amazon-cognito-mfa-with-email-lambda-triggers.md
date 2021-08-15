---
title: Amazon Cognito MFA with Email Using Lambda Triggers
description: How to set up MFA with an email address using Amazon Cognito
author: James Ingold
published: true
---

Amazon Cognito is a great service for easily getting started with authentication. It also has multi-factor authentication (MFA) right out of the box using a cell phone for SMS or a TOTP (Time-based One Time Password) device such as Authy or Google Authenticator. The requirement that I had was to only use MFA via email. I would have preferred to use the built-in AWS Cognito MFA workflows but this was a hard requirement for this project. This was in the healthcare industry and their email was usually restricted to being on the hospital network. Therefore, I had to allow MFA by email only.

This post will describe how to implement MFA with email on Amazon Cognito by using a Custom Authentication Flow and Amazon SES (Simple Email Service).

First, go to the AWS console and set up SES. Make sure that you have an email verified in Amazon SES. If your account is in Sandbox mode for Amazon SES, you will want to make sure to verify _both the sender and the receiver email address_. When the account is in production mode, you will not need to verify email addresses that you are sending to. I point this out because I spent 2 hours trying to figure out why sending an email was failing due to this issue.

To get started, we have to use a Custom Authentication Flow via Lambdas to handle this deviation from the built-in AWS Cognito MFA workflows. Amazon Cognito works via Lambda Functions and they allow different hooks to customize the authentication flow:

![Amazon Cognito Lifecycle](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3yl4hpkdlsieeh3e3tfc.png)
[Amazon Cognito Lifecyle Triggers](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools-working-with-aws-lambda-triggers.html):

- Define Auth Challenge: Determines the next challenge in a custom authentication flow (tell Cognito about our custom challenge)
- Create Auth Challenge: Creates a challenge in a custom authentication flow (send the MFA code)
- Verify Auth Challenge: Determines if a response is correct in a custom authentication flow (verify that the code entered matches the code we created)

There are a couple built in challenges that we can utilize, including FORCE_NEW_PASSWORD and PASSWORD_VERIFIER.

In order to create a custom auth flow that allows us to use MFA code via email we're going to implement it using these Lambda lifecycle hooks. The authentication flow will be as follows:

1. The user enters their email and password on the app.
2. On our secure server side (NodeJS/Express), we'll use the UpdateUserAttributes Cognito method to generate a new MFA code for the user and save it into their attributes as a custom property.
3. Our define auth challenge lambda function will be hit. If the user has a temporary password, we'll return the FORCE_NEW_PASSWORD challenge to the client.
4. Due to a bug in the custom authentication workflow, after the user changes their password, we'll force them to log in again. The bug makes it so we won't get to our custom challenges if the user is changing their password
5. Alright, now they're back to the login screen where they'll enter their email and new password
6. The PASSWORD_VERIFIER challenge will run verifying the users password. After that, we'll return a CUSTOM_CHALLENGE - our MFA by email challenge!
7. Our create auth challenge lambda function will run. In this function, we will send the MFA code. We will have the users attributes and we'll use SES to email them the code generated on the secure server side.
8. Once the challenge is created and the email sent, we'll return the current authentication flow state (CUSTOM_CHALLENGE) which should update the UI to ask for the MFA code.
9. The user inputs the MFA code that they received from their email. Now we send that answer to the verify auth challenge. This will verify the answer to what we have in the user's custom attributes for their current MFA code. We'll also make sure that the time frame for a valid code has not passed.

### Part 1: Create Serverless Project for Cognito Lifecycle Hooks

I'm going to use the Serverless framework because it's easy to manage and I know it well. However, you could just use Lambda functions if you wanted to keep it simple.

First, create a new pool through Amazon Cognito console. You can also use a cloud formation template, such as this [example](https://github.com/james-ingold/cognito-mfa-email-example/blob/main/pool_cloudformation.yaml).

Next, add a custom attribute called "authChallenge". This will hold our MFA code and a timestamp to make sure it's not expired.

![Custom Attribute](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nkwbum1dbly8e4gbucbv.png)

Keep Multi-Factor Authentication (MFA) set to "OFF" since we're going to be implementing this ourselves via a Custom Auth Flow and the Cognito Lifecycle Triggers.

Once the pool is created, create a new serverless project

```
npm install -g serverless
serverless create -n cognito-mfa-email-example
```

#### Define Lifecycle Function

The Define Cognito Lifecyle Event will determine the next challenge in a custom auth flow. This is a flow, we're going from one event to another starting with the PASSWORD_VERIFIER. There are a few important properties to return: issueTokens, failAuthentication, and the next challengeName to run. Once the password is confirmed (by Cognito), then we run our custom challenge. The PASSWORD_VERIFIER is a built in Cognito challenge which checks the user's password.

```javascript
module.exports.handler = async event => {
  // Kicks off with Secure Remote Password
  if (
    event.request.session.length === 1 &&
    event.request.session[0].challengeName === "SRP_A" &&
    event.request.session[0].challengeResult === true
  ) {
    event.response.issueTokens = false;
    event.response.failAuthentication = false;
    event.response.challengeName = "PASSWORD_VERIFIER";
    return event;
  }
  if (event.request.userNotFound) {
    event.response.failAuthentication = true;
    event.response.issueTokens = false;
    return event;
  }

  // Check result of last challenge
  if (
    event.request.session &&
    event.request.session.length > 2 &&
    event.request.session.slice(-1)[0].challengeResult === true
  ) {
    // The user provided the right answer - issue their tokens
    event.response.failAuthentication = false;
    event.response.issueTokens = true;
    return event;
  }

  event.response.issueTokens = false;
  event.response.failAuthentication = false;
  event.response.challengeName = "CUSTOM_CHALLENGE";
  return event;
};
```

#### Create Lifecyle Function

The Create function will run, creating our CUSTOM_CHALLENGE. We're going to use AWS SES to send an email with the MFA code in this function. The request will contain the user attributes from Cognito, we'll use the authChallenge custom attribute to get the code created on our server side and email that code to the user which they will use to complete MFA. [We also set the publicChallengeParameters and privateChallengeParameters which we'll pass through the flow](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-create-auth-challenge.html#cognito-user-pools-lambda-trigger-syntax-create-auth-challenge).

```javascript
const mailer = require("./mailer");
module.exports.handler = async event => {
  const challenge = event.request.userAttributes["custom:authChallenge"];
  const [authChallenge, timestamp] = (
    event.request.userAttributes["custom:authChallenge"] || ""
  ).split(",");
  // This is sent back to the client app
  event.response.publicChallengeParameters = {
    email: event.request.userAttributes.email
  };

  // add acceptable answer
  // so it can be verified by the "Verify Auth Challenge Response" trigger
  event.response.privateChallengeParameters = {
    challenge: challenge
  };

  // we want to check and make sure we haven't sent the code before in this login session before sending the code
  if (
    event.request.session.length < 3 &&
    !event.request.session.find(s => s.challengeName === "CUSTOM_CHALLENGE")
  )
    await mailer.send(
      "Your Access Code",
      event.request.userAttributes.email,
      "Please use the code below to login: <br /><br /> <b>" +
        authChallenge +
        "</b>"
    );

  return event;
};
```

#### Verify Lifecycle Function

In the verify lifecycle function, we take the code that the user inputted and the one we have in privateChallengeParameters to compare if they have the right code. Amazon Cognito invokes this trigger to verify if the response from the end user for a custom Auth Challenge is valid or not. We also check the timestamp to make sure it's not expired. We set the challenge response property of answerCorrect based on if they answered correctly. You could have even more challenges and the custom challenge flow loop would repeat until all challenges are answered.

```javascript
const LINK_TIMEOUT = 30 * 60;

module.exports.handler = async event => {
  // Get challenge and timestamp
  const [authChallenge, timestamp] = (
    event.request.privateChallengeParameters.challenge || ""
  ).split(",");

  // Check if code is equal to what we expect...
  if (event.request.challengeAnswer === authChallenge) {
    // Check if the link hasn't timed out
    if (Number(timestamp) > new Date().valueOf() / 1000 - LINK_TIMEOUT) {
      event.response.answerCorrect = true;
      return event;
    }
  }

  event.response.answerCorrect = false;
  return event;
};
```

##### Deploy the Project

I've included a full [sample serverless.yaml here](https://github.com/james-ingold/cognito-mfa-email-example/blob/main/serverless.yml). You'll want to have the following IAM role capabilities and the following function declarations. I'm using the Amazon Systems Parameter Store for setting the EMAIL_ADDRESS variable which was used in the Create lifecycle function

serverless.yaml snippet

```
iamRoleStatements:
    - Effect: "Allow"
      Action:
        - cognito-idp:AdminGetUser
        - cognito-idp:AdminUpdateUserAttributes
      Resource:
        - {"Fn::GetAtt": [UserPool, Arn]}
    - Effect: "Allow"
      Action:
        - ses:SendEmail
      Resource:
        - "*"

functions:
  define:
    handler: defineAuth.handler
    events:
      - cognitoUserPool:
          pool: ${self:custom.userPoolName}
          trigger: DefineAuthChallenge
          existing: true
  create:
    handler: createAuth.handler
    environment:
      EMAIL_ADDRESS: ${ssm:/${self:provider.stage}_EMAIL_ADDRESS}
    events:
      - cognitoUserPool:
          pool: ${self:custom.userPoolName}
          trigger: CreateAuthChallenge
          existing: true
  verify:
    handler: verify.handler
    events:
      - cognitoUserPool:
          pool: ${self:custom.userPoolName}
          trigger: VerifyAuthChallengeResponse
          existing: true
```

### Part 2: Pseudo Backend Code

Now that we've got our Cognito handlers set up and a user pool, we can run through some example server side code that handles the auth sequence. We start with the SRP_A (Secure Remote Password) challenge which kicks off the authentication flow.

##### Login Route: Kick off auth flow and prepare custom challenge

```javascript
router.post("/login", async (req, res, next) => {
  try {
    const SRP_A = CognitoHelper.calculateA();
    const userHash = generateHash(
      req.body.username,
      this.secretHash,
      this.clientId
    );
    const params = {
      UserPoolId: this.poolId,
      AuthFlow: "CUSTOM_AUTH",
      ClientId: this.clientId,
      AuthParameters: {
        USERNAME: req.body.username,
        PASSWORD: req.body.password,
        SECRET_HASH: userHash,
        SRP_A: SRP_A,
        CHALLENGE_NAME: "SRP_A"
      }
    };

    try {
      const data = await this.cognitoIdentity
        .adminInitiateAuth(params)
        .promise();
      // SRP_A response
      const { ChallengeName, ChallengeParameters, Session } = data;
      // Set up User for MFA
      const authChallenge = _.map([...Array(8).keys()], n =>
        Math.floor(Math.random() * 10)
      ).join("");
      await this.cognitoIdentity
        .adminUpdateUserAttributes({
          UserAttributes: [
            {
              Name: "custom:authChallenge",
              Value: `${authChallenge},${Math.round(
                new Date().valueOf() / 1000
              )}`
            }
          ],
          UserPoolId: this.poolId,
          Username: req.body.username
        })
        .promise();
      const hkdf = CognitoHelper.getPasswordAuthenticationKey(
        ChallengeParameters.USER_ID_FOR_SRP,
        req.body.password,
        ChallengeParameters.SRP_B,
        ChallengeParameters.SALT,
        this.poolId
      );
      const dateNow = CognitoHelper.getNowString();
      const signatureString = CognitoHelper.calculateSignature(
        hkdf,
        this.poolIdOnly,
        ChallengeParameters.USER_ID_FOR_SRP,
        ChallengeParameters.SECRET_BLOCK,
        dateNow
      );
      const responseParams = {
        ChallengeName: ChallengeName,
        ClientId: this.clientId,
        ChallengeResponses: {
          PASSWORD_CLAIM_SIGNATURE: signatureString,
          PASSWORD_CLAIM_SECRET_BLOCK: ChallengeParameters.SECRET_BLOCK,
          TIMESTAMP: dateNow,
          USERNAME: ChallengeParameters.USER_ID_FOR_SRP,
          SECRET_HASH: generateHash(
            ChallengeParameters.USERNAME,
            this.secretHash,
            this.clientId
          )
        },
        Session: Session
      };
      // Password verifier response, should be set up for custom mfa challenge now
      const respData = await this.cognitoIdentity
        .respondToAuthChallenge(responseParams)
        .promise();
      res.status(200).json(respData).end();
    }
  } catch (err) {
    console.log(err);
    if (err.code === "UserNotFoundException")
      return res.status(400).send("Invalid Credentials");
    res.sendStatus(500);
  }
});
```

##### Verify Route: This handles responding to our auth challenge with Cognito which will call our verify lifecycle handler.

```
router.post('/verify', async (req, res, next) => {
  try {
    let data = {}
    const { email, password, confirmPassword, user, code } = req.body
    const params = {
      ChallengeName: user.ChallengeName,
      ClientId: this.clientId,
      ChallengeResponses: {
        USERNAME: email,
        ANSWER: code,
        SECRET_HASH: generateHash(email, this.secretHash, this.clientId)
      },
      Session: user.Session
    }
    const data = await this.cognitoIdentity.respondToAuthChallenge(params).promise()

    // if we failed the custom challenge
    if (!data.AuthenticationResult && user.ChallengeName === 'CUSTOM_CHALLENGE' && code) {
      data.error = 'Invalid Code'
      return res.json({ success: false, data: data })
    } else {
      // add any user claims here
      data.AuthenticationResult.UserClaims = {}
      return res.json({ success: true, data: data })
    }
  } catch (err) {
    console.log(err)
    res.sendStatus(500)
  }
})
```

### Conclusion

That's it! We've now built a custom authentication workflow for handling MFA by email using Amazon Cognito. Our server code initiates the auth flow and then provides challenge responses until we have verified that the user has a valid MFA code. Happy Codings!

##### References:

[Source Code Repository](https://github.com/james-ingold/cognito-mfa-email-example)

[Cognito Custom Auth Lambdas](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-challenge.html){:target="\_blank"}
