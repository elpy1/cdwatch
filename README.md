
# cdwatch
A small python wrapper for AWS codedeploy to watch your EC2 deployment until completion and provide output similar to what you'd see in the AWS console.

## Requirements
- python3
- `boto3` pip module
- a `codedeploy` deployment to deploy :innocent:

## Info
It's sort of annoying when you're regularly testing deployments and you want information regarding the outcome of your deployment e.g. why it failed or took 30s longer than it did the previous time. Also, if you're running deployments from within some type of CI/CD pipeline I think it's much nicer to have the output all in the one place.

This tool was created to watch a `codedeploy` EC2 deployment until completion, exit appropriately and provide terminal output (including diagnostic information on failure).

`cdwatch` will only watch a single deployment target (the first) associated with the deployment-id its given. By default, it will watch the deployment target for a total of `120s` and will check the deployment every `2s`. `cdwatch` will stop either once the `timeout` elapses or the deployment finishes. You don't have to watch the deployment as it completes and can optionally provide a deployment-id with the `--id` argument.

Both the `timeout` and `freq` can be configured as arguments. These values should be in line with your expected deployment time.

**NOTE:** depending on your deployment strategy this tool might not be appropriate. In my case, I can be confident watching a single deployment target because if it fails I know the overall deployment fails.


## Usage
The script will read from `stdin` and expects a `json` payload containing the deployment-id from `codedeploy`. This means you can simply pipe`aws deploy create-deployment` in to `cdwatch`.


```
usage: cdwatch [-h] [--profile AWS_PROFILE] [--id ID] [--freq FREQ]
               [--timeout TIMEOUT] [--quiet | --json]

watch a codedeploy deployment

optional arguments:
  -h, --help            show this help message and exit
  --profile AWS_PROFILE
  --id ID               provide deployment-id as argument
  --freq FREQ           time in seconds between deployment checks
  --timeout TIMEOUT     total time in seconds to watch deployment
  --quiet               don't show diagnostic information on failure
  --json                display diagnostic output as raw json
```



## Examples
### Watch a deployment (successful)
```
[elpy@testbox ~]$ aws deploy create-deployment --application-name ${DEPLOY_APP} --deployment-group ${DEPLOY_GROUP} --s3-location bucket=${S3_BUCKET},key=${ARTIFACT},bundleType=tgz | cdwatch 
got deployment id: d-5F9XXX9B4
found deployment target(s): i-0e6ccxxxxxx2f2184

watching deployment for i-0e6ccxxxxxx2f2184..

ApplicationStop
 - duration: 0s
 - status: Succeeded ✔
DownloadBundle
 - duration: 3s
 - status: Succeeded ✔
BeforeInstall
 - duration: 0s
 - status: Succeeded ✔
Install
 - duration: 0s
 - status: Succeeded ✔
AfterInstall
 - duration: 19s
 - status: Succeeded ✔
ApplicationStart
 - duration: 0s
 - status: Succeeded ✔
ValidateService
 - duration: 0s
 - status: Succeeded ✔

watch ended..
```


### Watch a deployment (failed)
```
[elpy@testbox ~]$ aws deploy create-deployment --application-name ${DEPLOY_APP} --deployment-group ${DEPLOY_GROUP} --s3-location bucket=${S3_BUCKET},key=${ARTIFACT},bundleType=tgz | cdwatch 
got deployment id: d-38CXXXMC4
found deployment target(s): i-0e6ccxxxxxx2f2184

watching deployment for i-0e6ccxxxxxx2f2184..

ApplicationStop
 - duration: 0s
 - status: Succeeded ✔
DownloadBundle
 - duration: 2s
 - status: Succeeded ✔
BeforeInstall
 - duration: 0s
 - status: Succeeded ✔
Install
 - duration: 0s
 - status: Succeeded ✔
AfterInstall
 - duration: 8s
 - status: Failed ✘
ApplicationStart
 - status: Skipped
ValidateService
 - status: Skipped

watch ended..

lifecycle event failure detected! diagnostic information:
 - errorCode: ScriptFailed
 - scriptName: codedeploy/ansible.sh
 - message: Script at specified location: codedeploy/ansible.sh run as user root failed with exit code 2
 - logTail: [stdout]
[stdout]TASK [set facts] ***************************************************************
[stdout]ok: [localhost] => {"ansible_facts": {"ec2_hostname": "ip-10-xxx-xx-67.ap-southeast-2.compute.internal", "ec2_instance_id": "i-0e6ccxxxxxx2f2184", "ec2_local_ipv4": "10.xx.xx.79", "ec2_tag_app": "xxx-frontend", "ec2_tag_env": "prod"}, "changed": false}
[stdout]
[stdout]TASK [dat_role : dat task] *************************************
[stdout]ok: [localhost] => {"append": true, "changed": false, "comment": "", "group": 666, "groups": "wheel", "home": "/home/r00t", "move_home": false, "name": "r00t", "shell": "/bin/false", "state": "present", "uid": 666}
[stdout]
[stdout]TASK [systemd_timers : another task] **************************
[stdout]fatal: [localhost]: FAILED! => {"ansible_facts": {}, "ansible_included_var_files": [], "changed": false, "message": "We were unable to read either as JSON nor YAML, these are the errors we got from each:\nJSON: Expecting value: line 1 column 1 (char 0)\n\nSyntax Error while loading YAML.\n  expected alphabetic or numeric character, but found '*'\n\nThe error appears to be in '/opt/deploy/ansible/roles/dat_role/vars/prod/main.yml': line 269, column 19, but may\nbe elsewhere in the file depending on the exact syntax problem.\n\nThe offending line appears to be:\n\n    ExecStart: /usr/local/bin/drush @%i yeah\n    OnCalendar: *:0/15\n                  ^ here\n"}
[stdout]
[stdout]PLAY RECAP *********************************************************************
[stdout]localhost                  : ok=5    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0   
[stdout]
```
