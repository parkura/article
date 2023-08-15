AWS Step Functions Activities
Integrating decoupled workers into state machine workflows



![image](https://github.com/parkura/article/assets/64126618/484de88c-68a2-47ef-9123-409936baf956)


source: diagram by author
Launched in 2016, AWS Step Functions offers state machines to orchestrate requests across AWS services. Supporting integrations with Lambda functions, SQS queues, DynamoDB tables, and many more services, Step Functions can automate complex processes and manage thousands of execution starts per second — accommodating for serverless architectures and taking advantage of their scaling capabilities.

There may be cases where we want to rate limit steps in our workflows: quality checking data manually, or capping Lambda function concurrencies to accommodate for external API limits. This adds an asynchronous element to an otherwise synchronous flow. Previously we explored Step Functions Callbacks to handle such scenarios: generating task tokens, pushing them to asynchronous workers for processing, and pausing our executions until the workers returned an outcome.

Often this suffices, but suppose we have a worker unsupported by Callbacks, or another within a network inaccessible to Step Functions, leaving us unable to push task tokens. How do we integrate workers in these cases?

Activities — one of the first features to launch for Step Functions —is one option to address these problems. Broadly similar to Callbacks, Activities rely on workers and tokens to coordinate asynchronous tasks. The fundamental difference lies in the relationship between Step Functions and the worker: where Callbacks are push-based, Activity workers poll for tasks.


How does an Activity Work?
There’s two key parts to an Activity. First we have the worker. This can be a Lambda function, an EC2 instance, or effectively anything that runs code. The worker polls Step Functions for tasks using the Activity’s unique Amazon Resource Number (ARN).

When a pending task is retrieved, the worker acquires a unique task token along with input data to process. The worker applies its magic, determines whether the request succeeded or failed, and reports this outcome back to the Step Functions. If the request takes a while to process, the worker may also send a heartbeat to acknowledge the request remains in progress.

Within the Step Functions workflow, we take the Activity’s ARN and embed it within a new Activity invocation state. This state initiates new Activity tasks which the worker polls for. When a Step Functions execution reaches the invocation state, the workflow is paused until the Activity acknowledges a response from the worker, or the Activity times out.

![image](https://github.com/parkura/article/assets/64126618/9ae4ff11-bc02-4906-b474-e688aae6bf67)

source: diagram by author
How do we Build an Activity?
Let’s provision a Step Functions Activity using the latest version of the AWS Cloud Development Kit (CDK) for TypeScript — version 2.10.0 at the time of writing. In this example, we’ll provision a Lambda function as our Activity worker and we’ll integrate this into a Step Functions workflow with only three states: Start, Activity Task, and End.

First we provision an Activity using CDK’s Activity construct. The name for the Activity is optional. We incorporate this into a StepFunctionsInvokeActivity state, which will create activity tasks for our worker. Last, but not least, we define a StateMachine to complete our Step Functions configuration, passing the invocation state into the definition.

import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { Activity, StateMachine } from 'aws-cdk-lib/aws-stepfunctions'
import { StepFunctionsInvokeActivity } from 'aws-cdk-lib/aws-stepfunctions-tasks';

export class AwsStepFunctionsActivityStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);
    
    const activity = new Activity(this, 'Activity', {
      activityName: 'example-activity'
    });

    const activityTask = new StepFunctionsInvokeActivity(this, 'ActivityTask', {
      activity,
    });

    const stateMachineName = 'example-state-machine';

    new StateMachine(this, 'StateMachine', {
      definition: activityTask,
      stateMachineName,
    });
  }
}

Now we can turn our attention to the worker. So far we’ve created an Activity, but no worker is yet processing its tasks. Choosing a Node Lambda function for this example, let’s provision a NodejsFunction, passing the activity ARN as an environment variable for reference within the function code.

import { Duration } from 'aws-cdk-lib';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { RetentionDays } from 'aws-cdk-lib/aws-logs';

const activityWorker = new NodejsFunction(this, 'ActivityWorker', {
  description: `Lambda worker for activity in Step Functions '${stateMachineName}'`,
  entry: 'src/activityWorker.ts',
  environment: {
    ACTIVITY_ARN: activity.activityArn,
  },
  functionName: `example-activity-worker`,
  logRetention: RetentionDays.FIVE_DAYS,
  timeout: Duration.minutes(2),
});

Defining the Lambda TypeScript code within src/activityWorker.ts, we read the ACTIVITY_ARN environment variable and long poll for pending tasks using the AWS SDK getActivityTask method. Note that polling may run for up to one minute, whilst Lambda functions default to a three-second timeout, so make sure to increase the Lambda timeout above one minute.


import { ScheduledHandler } from 'aws-lambda';
import { AWSError, StepFunctions } from 'aws-sdk';

const { ACTIVITY_ARN } = process.env;
const LOG = require('simple-node-logger').createSimpleLogger();
const STEP_FUNCTIONS_CLIENT = new StepFunctions();

export const handler = async (_event: ScheduledHandler) => {
  LOG.info('Activity worker invoked');
  if (!ACTIVITY_ARN) {
    throw Error('Activity ARN from environment variables is undefined')
  }

  const activityTask = await STEP_FUNCTIONS_CLIENT.getActivityTask({
    activityArn: ACTIVITY_ARN,
    workerName: 'example-activity-worker',
  }).promise();

  if (!activityTask.taskToken) {
    LOG.info('No tasks to process');
    return;
  }
}

If a pending task is found, we have input data and a task token to process. Using an elementary random generator to decide whether this task should succeed or fail, we can provide output along with the task token to the SDK’s sendTaskSuccess or sendTaskFailure methods depending on the outcome.

// randomly decide whether to be successful or not
const isTaskSuccessful = Math.random() < 0.5;

let response: PromiseResult<unknown, AWSError>;
let taskOutput;

if (isTaskSuccessful) {
  taskOutput = {
    output: activityTask.input as string,
    taskToken: activityTask.taskToken as string,
  };
  response = await STEP_FUNCTIONS_CLIENT.sendTaskSuccess(taskOutput).promise();
} else {
  taskOutput = {
    cause: 'Flipping a coin. Unlucky this time.',
    error: 'Bad luck! Try again.',
    taskToken: activityTask.taskToken as string,
  };
  response = await STEP_FUNCTIONS_CLIENT.sendTaskFailure(taskOutput).promise();
}

That wraps up the worker logic! Now for the finishing touches. Let’s return to the CDK and grant our Lambda function Identity and Access Management (IAM) permissions to poll and report task outcomes to the Step Functions Activity, as well as scheduling a five minute invocation rate on the Lambda function to keep it warm.


import { Rule, Schedule } from 'aws-cdk-lib/aws-events';
import { LambdaFunction as LambdaFunctionTarget } from 'aws-cdk-lib/aws-events-targets'; 
import { PolicyStatement } from 'aws-cdk-lib/aws-iam';

activityWorker.addToRolePolicy(
  new PolicyStatement({
    actions: ['states:GetActivityTask', 'states:SendTask*'],
    resources: [activity.activityArn],
  })
);

new Rule(this, 'ActivitySchedule', {
  schedule: Schedule.cron({ minute: '0/5' }),
  targets: [new LambdaFunctionTarget(activityWorker)],
});

Invoking the Step Functions manually from the AWS Console, we observe the pending activity task, which shows blue to indicate it is in progress until our Cron schedule invokes the Lambda worker. This will remain in progress for up to seven minutes depending on the invocation and processing times. When the worker fetches the task and reports success or failure, our Step Functions share this outcome on the Console. In failure cases, we see an Exception tab for the invocation state surfacing error details from the worker.

![image](https://github.com/parkura/article/assets/64126618/e0bfc403-0cb8-4405-b046-3d3557ef80d1)


What Limitations do Activities Present?
Activity workers struggle to poll for pending tasks under high throughput. If large volumes of traffic is anticipated, consider using Callbacks with SQS queues and configure workers to poll queues instead of Activities for tasks.

Conclusion
Bearing a large resemblance to Callbacks, Activities provide an alternative approach to orchestrate asynchronous tasks within Step Functions. Requiring a little extra work to accommodate for long polling of task tokens, Activities are unlikely to be the first choice for most asynchronous use cases.

However, where decoupled workers are necessary, or Callbacks don’t support integration with the workers, Activities remain a valuable option. Keep in mind the limitations for high throughput, where an alternative approach with Callbacks and SQS queues may be more robust.

The full codebase for this Activity walkthrough is available on GitHub. If you have prior experience working with Step Functions Activities, or Step Functions more generally, do let me know.

Blog posted amended on February 5th 2022 to share limitations regarding Activity worker polling and high throughout traffic with a recommended workaround using Callbacks and SQS queues. Credit to 
Adam Wong
 for sharing this.
