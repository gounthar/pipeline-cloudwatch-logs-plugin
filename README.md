A Jenkins plugin to send Pipeline build logs to [Amazon CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
and display them directly from the same source.

This is the [reference implementation of JEP-210](https://github.com/jenkinsci/jep/blob/master/jep/210/README.adoc#prototype-implementation).

# Usage

After installing, go to **Manage Jenkins » AWS** and configure your **Log group**.
You may also need to configure your **Amazon Credentials**, depending on how Jenkins is running.
Be sure to click **Validate configuration** before saving.

Henceforth, so long as the plugin remains enabled, all Pipeline builds will stream log content directly to CloudWatch Logs.
Viewing the **Console** link in Jenkins (or the equivalent in Blue Ocean) will load log content directly from CloudWatch Logs.

The first line of a (classic) console view will be a link to the CloudWatch Logs section of the AWS Console
with the log stream name for all messages coming from the Jenkins master and a filter predefined to show that build.
(Currently you would need to manually browse to related log streams to see output produced by agents.)

## Security

With this plugin active, log content generated by processes running on agents, such as `sh` steps,
will be sent to CloudWatch Logs directly from that agent machine, without passing through the Jenkins master.
For that to work, the master will send AWS credentials to the agent sufficient to write logs.
That prevents a rogue build from writing to logs of another job (though not another build of the same job),
or otherwise causing problems.

It will attempt to use either the `AssumeRole` or `GetFederationToken` API calls to create a fresh session token
with a policy restriction allowing the agent only to write logs to the streams for this job and nothing else.
One of these is selected by inspecting the configured master credentials according to
[this table](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_request.html#stsapi_comparison).
If such a policy restriction fails, the master falls back to transmitting its own credentials.
**Validate configuration** will indicate whether these calls are likely to work before actually running a build.

Note that if agents are themselves running in EC2, you must use tools to block them from having any default AWS credentials.
For example, if the Jenkins system is running in EKS, you could try [kube2iam](https://github.com/jtblin/kube2iam).

### Permissions for the master

The Jenkins master will need permissions for at least these API calls, scoped to the log group name:

* `FilterLogEvents`
* `DescribeLogStreams`
* `CreateLogStream`
* `AssumeRole` (where applicable)
* `GetFederationToken` (where applicable)
* `PutLogEvents`
* `GetLogEvents`

### Permissions for the agent

Assuming either `AssumeRole` or `GetFederationToken` succeeds,
the agent will be given these permissions,
scoped to particular log group and stream names:

* `PutLogEvents`
* `GetLogEvents`

### `withCredentials`

The `withCredentials` step works to mask secrets accidentally printed to the console log, even from agents.
The secret will be present in the agent JVM (as always)
and the filtering happens in the agent JVM before transmission to CloudWatch Logs.

# Known issues

Temporary credentials are issued with a default expiration time (such as one hour)
and are not currently refreshed after that expiration,
so there may be a loss of content for individual steps running longer than that.

Only `AssumeRule` starting with temporary security credentials has been tested for purposes of policy restriction.
`GetFederationToken` has not been tested.

The retrieval of log content from CloudWatch Logs for display inside Jenkins is not well optimized.
API requests made from the master filter down to the level of log stream group, log stream name pattern,
build number, and (where applicable) step identifier.
However due to the use of bytestream-based cursors in Jenkins (core and Blue Ocean),
the entire build may be downloaded even if the console display is truncated;
and incremental display of a running build will perform multiple downloads.

Scalability to large numbers of builds, gigantic build logs, or extremely long lines of text has not been evaluated.

**Validate configuration** does not check availability of all master permissions.

Job names involving unusual characters may not work.

Log stream names are derived from job names,
so if you rename a job the logs for historical builds will not be displayed.

Pending [JEP-207](https://jenkins.io/jep/207), the plugin takes effect immediately upon installation,
and there is no record in `build.xml` that logs are in CloudWatch Logs.
Also historical builds logs using a `log` file will not be displayable.

# Automated tests

You can run tests in the usual way via

```sh
mvn test
```

but the meat of the testing is in `PipelineBridgeTest` which is a live test (contacting AWS)
and as such requires several environment variables to be set:

* `AWS_PROFILE` should select a profile in `~/.aws/credentials`.
  This may be a session token, but should _not_ have assumed a role.
* `AWS_ROLE` should refer to a role (`arn:aws:iam::…:role/…` syntax) which this profile may assume.
  The role should grant at least the master permissions detailed above.
* `AWS_REGION` should be set to, e.g., `us-east-1`.
* `CLOUDWATCH_LOG_GROUP_NAME` should be set to a test log group name, which you may need to precreate.

The actual test case are defined a TCK-like
[`LogStorageTestBase`](https://github.com/jenkinsci/workflow-api-plugin/blob/907cf64feb2f38e93aebbedf87946b0235c3dc93/src/test/java/org/jenkinsci/plugins/workflow/log/LogStorageTestBase.java#L81-L295)
and you may run an individual test case if desired:

```sh
mvn surefire:test -Dtest=PipelineBridgeTest\#smokes
```

Currently there is CI coverage only for the relatively simple unit & UI tests.
`PipelineBridgeTest` is skipped.

# Interactive tests

You can try the plugin interactively in the usual way via

```sh
export AWS_PROFILE=… # as above
mvn hpi:run
```

Then go [here](http://localhost:8080/jenkins/aws/) to set up AWS credentials.
You will need to select a **Region**, as above;
and then for **Amazon Credentials** click **Add** and create **AWS Credentials**.
You may leave **Access Key ID** and **Secret Access Key** blank if `AWS_PROFILE` was set,
but do click **Advanced…** and enter **IAM Role To Use** in the format of `AWS_ROLE` above.
With your newly created credentials selected,
enter your **Log group** as in `CLOUDWATCH_LOG_GROUP_NAME` above,
and click **Validate configuration**.
You should see **success**; a connection error;
or a warning about role assumption, as described in the security section above.

With AWS information configured, define an agent to use during builds.
This is most easily done with the [Mock Agent plugin](https://plugins.jenkins.io/mock-slave),
though you could use anything else.
Then create a Pipeline project with a script involving at least one remote `node` block
matching your agent’s label selector or name, e.g.

```groovy
node('remote') {
  sh '''
    set +x
    for x in 0 1 2 3 4 5 6 7 8 9
    do
      for y in 0 1 2 3 4 5 6 7 8 9
      do
        echo $x$y
        sleep 1
      done
    done
  '''
}
```

When you run a build, you should see an initial log line

```
[view in AWS Console, if authorized, plus related streams for agent output]
```

with a hyperlink to the Console.
Look at the two log streams created, one for messages originating from the Jenkins master,
one for messages produced and sent from the agent.
Each log event will have a JSON block with at least a `message` and `build` field,
and sometimes also `node` and/or `annotations` fields.
`node` will match the IDs used in the **Pipeline Steps** display.
`annotations` is used for hyperlinks, build structure metadata, and more.

You can also view the build log using the AWS CLI:

```sh
aws logs filter-log-events --log-group-name $CLOUDWATCH_LOG_GROUP_NAME --log-stream-name-prefix "${JOB_FULL_NAME}@" | jq -r '.events[].message | fromjson | .message'
```

Verify that the rest of the log looks like the normal Jenkins “classic” log UI for builds in progress.

You can also install the **Blue Ocean** plugin from the update center and check log display there.

# Alternatives

The [AWS CloudWatch Logs Publisher plugin](https://plugins.jenkins.io/aws-cloudwatch-logs-publisher)
assumes logs are still stored in `$JENKINS_HOME`,
and then just offers a Pipeline step (which your `Jenkinsfile` must call)
which uploads the log text up to that point to CloudWatch Logs
(any output after the step is called will not be uploaded).
Logs are still displayed from the traditional location,
and log text is still streamed from agents to the master via the Remoting channel.

# Other materials

* [Scaling Network Connections from the Jenkins Master](https://sched.co/F9NP) and [video](https://youtu.be/h7JK6Ak3cJc) (2018-09-18)
* [Cloud Native SIG subsection](https://jenkins.io/sigs/cloud-native/pluggable-storage/#build-log-storage) and [2018-08-16 status meeting](https://jenkins.io/sigs/cloud-native/#2018-08-16-status-meeting)

# Changelog

## 0.2 and later

See [GitHub Releases](https://github.com/jenkinsci/pipeline-cloudwatch-logs-plugin/releases).

## 0.1 (2018-11-20)

Initial release. Treat as alpha quality.
