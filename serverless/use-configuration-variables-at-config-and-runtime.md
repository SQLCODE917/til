# Use Configuration Variables At Config And Runtime

In Serverless 0.4.2, variables defined in `_meta/variables/s-variables-common.json` are accessible in all JSON config files,
like `s-function.json` and `s-resources-cf.json`,
and all domains and stages.

Given the contents of `s-varialbes-common.json` like

```
{
    "KinesisStreamArn": "arn:aws:kinesis:us-east-1:1234567890:stream/my-stream"
}
```

You can use it in an interpolation within your `s-function.json`,
for example when declaring an event source:

```
    ...
    "events": [
        {
            "name": "eventStream",
            "type": "kinesisstream",
            "config": {
                "streamArn": "${KinesisStreamArn}",
                "enabled": true
            }
        }
    ]
    ...
```

However, this does not allow you to reference these variables within the _handler_ of your Lambda function.
One way to get around that would be by taking advantage of the [serverless-helpers-js](https://github.com/serverless/serverless-helpers-js) package that can asynchronously introspect the output of `s-resources-cf.json` CloudFormation template.
Because you can interpolate `s-variables-common.json` variables inside the template, all you have to do is connect the dots:

In your `s-resources-cf.json`

```
    ...
    "Outputs": {
        "KinesisStreamArn": {
            "Description": "ARN of the Kinesis stream",
            "Value": "${KinesisStreamArn}"
        }
    }
    ...
```

Install the `serverless-helpers-js` in the component that contains the function that needs to know the _KinesisStreamArn_ at runtime:

```
npm install --save serverless-helpers-js
```

At the top of your function's handler, import the _serverless-helpers_:

```
var ServerlessHelpers = require('serverless-helpers-js');
ServerlessHelpers.loadEnv();
```

_loadEnv()_ will load your project's _.env_ file for cases when you decide to test manually.
For more on the project structure, consult the [docs](http://docs.serverless.com/docs/project-structure)

Define your function's handler:

```
module.exports.handler = function(event, context) {

    if (!process.env.SERVERLESS_REGION) {
        process.env.SERVERLESS_REGION = (process.env.AWS_REGION) ?
            process.env.AWS_REGION : process.env.AWS_DEFAULT_REGION;
    }

    ServerlessHelpers.CF.loadVars()
    .then(function() {
        ...
        console.log("KinesisStreamArn:", process.env.SERVERLESS_CF_KinesisStreamArn);
        ...
    })
    .catch(function(err) {
        return context.done(err, null);
    });
};
```

The `SERVERLESS_REGION` variable dance is an exception to the textbook implementation of _CF.loadVars()_.
You could provide it by the means of `[serverless env set](http://docs.serverless.com/docs/env-set)`.
However, more often than not, you will have that information defined already, in your `AWS_REGION` or `AWS_DEFAULT_REGION` environmental variables.
If you want to avoid having that unsightly block of code every time you use _CF.loadVars()_, add this command to your build process:

```
serverless env set -k "SERVERLESS_REGION" -v "us-east-1" -s "my-stage" -r "us-east-1"
```

This will enrich your S3 bucket with a variable you can access with `process.env.SERVERLESS_REGION` in your handler.
