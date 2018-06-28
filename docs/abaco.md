Automating Analysis with Abaco
================================================

We are still in the early stages of setting up Abaco on the Cyverse tenant. In the future many of these processes will be templated and baked into the
cyverse-cli. In this example, we will show you how to setup automated analysis that is triggered when a file is uploaded to agave.

To get setup using abaco, you will need the following requirements:

1. Install the [Cyverse SDK CLI](getting-started.md)
2. Install the [Abaco CLI](https://github.com/johnfonner/abaco-cli)
3. [Docker CE](https://www.docker.com/community-edition)
4. [Docker Hub Account](https://hub.docker.com/)
5. [jq](https://stedolan.github.io/jq/)

Next you will need to subscribe to the Abaco API, this is done with a cURL command:
```
curl -d "apiName=Abaco&apiProvider=admin&apiVersion=v2" -u <cyverse_uname>:<cyverse_password> https://agave.iplantc.org/clients/v2/my_client/subscriptions
```

Then a Cyverse admin will need to give you an authorized role on the Abaco API. You can contact me (Joshua Urrutia) to facilitate the authorization. In the future these first two steps will happen automatically.

Create an Agave App or use an Existing App
-----------------

Next, you'll need an agave app you'd like to use to run your analysis. There is a fastqc example app you can clone and deploy here:
```
git clone https://github.com/JoshuaUrrutia/fastqc_app.git
```
In the future you will be able to build and deploy this app in one step by editing the fields in the `app.ini` file and running:
```
apps-deploy fastqc-0.11.7 fastqc-0.11.7/app.json.j2
```
But until the new Cyverse-cli is released, you'll have to build and deploy manually:
```
CONTAINER_IMAGE="$DOCKER_ORG/$CONTAINER_NAME:$version"
docker build -t ${CONTAINER_IMAGE}
docker push ${CONTAINER_IMAGE}
```
Then edit the relevant fields in the `fastqc-0.11.7/app.json.j2`, and run:
```
apps-addupdate -F fastqc-0.11.7/app.json.j2
```

Setting up the Reactor
-----------------

Now you are ready to setup a reactor. You can copy the following reactor to use as a template:
```
git clone https://github.com/JoshuaUrrutia/fastqc_router_reactor.git
```
Edit the `DOCKER_HUB_ORG=` field in the `reactor.rc` file to point to your dockerhub username, and run:
```
abaco deploy
```
From the `fastqc_router_reactor` directory. After `abaco deploy` has run it will return an actor ID at the end, ex:
```
Successfully deployed actor with ID: GPrgrggl5ler4
```
Keep track of this ID because you will need it for the next step. If you misplace it you can run `abaco list` to see the currently deployed reactors:
```
$ abaco list
example-actor  60apmNJPZGAJk  READY
fastqc_router  GPrgrggl5ler4  READY
```
`fastqc_router` is the actor alias, and `GPrgrggl5ler4` is the actor ID.

Creating File System Notifications
-----------------

Now you're ready to create a file system notification. This notification will pass a message to the `fastqc_router_reactor` when a file is uploaded to a particular directory in agave. The `fastqc_router_reactor` takes this notification, crafts a job.json, and submits a job to the `fastqc_app`. We've created a python wrapper to help setup the file system notifications, you can download the python scripts here:
```
git clone https://github.com/JoshuaUrrutia/abaco_notifications.git
```
From the `abaco_notifications` directory, you can run `add_notify_reactor.py` to setup a notification. For example:
```
python add_notify_reactor.py data.iplantcollaborative.org urrutia/fastqc GPrgrggl5ler4
```
This will send a notification to the `fastqc_router_reactor` whenever a file is uploaded the `fastqc` directory in my personal storage space on `data.iplantcollaborative.org`. More generally, you can use the following pattern:
```
python add_notify_reactor.py $AGAVE_SYSTEM_NAME $PATH_TO_DIRECTORY $ACTOR_ID
```

Now the only thing left to do is to test and see if your `notification -> reactor -> app` chain is functioning.

```
files-upload -F read1.fastq.gz -S data.iplantcollaborative.org urrutia/fastqc
```

Within a few minutes you should see a message get passed to your reactor, you can check by running:
```
$ abaco executions -v $ACTOR_ID
```
If a message was passed you can find the execution id in the json message that is returned:
```
"executions": [
  {
    "id": "5P6MjVp8K1y55",
    "message_received_time": "2018-06-21 02:43:16.627228",
    "start_time": null,
    "status": "SUBMITTED"
  }
```
To see the logs from that execution, you can run:
```
abaco logs $ACTOR_ID $EXECUTION_ID
abaco logs gO0JeWaBM4p3J 5P6MjVp8K1y55
```

If executed successfully and agave job will be submitted, you can check the status of your 5 most recent jobs by running:
```
jobs-list -l 5
```

Once the job is finished running, you can see the output in a directory called `analyzed` which will be in the same directory where you uploaded the original file.
```
files-list -S $AGAVE_SYSTEM_NAME $PATH_TO_DIRECTORY/analyzed
files-list -S data.iplantcollaborative.org urrutia/fastqc/analyzed
```

There's not reason to stop at a single app, you can craft an entire automated pipeline that deploys multiple apps to run series and/or parallel. The sky is the limit!

Troubleshooting
-----------------

If the reactor never executed, you can check the notifications are working by posting notifications to `requestbin` using the `add_notify_requestbin.py` script in the `abaco_notifications` directory:
```
$ python add_notify_requestbin.py $AGAVE_SYSTEM_NAME $PATH_TO_DIRECTORY
assocationIds = 344770698063965720-242ac112-0001-002
notification id: 213876312678002200-242ac11a-0001-011
notification url: https://requestbin.agaveapi.co/10ymcld1
```
You can reupload the file and check the requestbin url to see if it recieves the notification:
```
files-upload -F $LOCAL_FILE -S $AGAVE_SYSTEM_NAME $PATH_TO_DIRECTORY
```
Add `?inspect` onto the end of the notification url that was returned by the `add_notify_requestbin.py` script, ex: `https://requestbin.agaveapi.co/10ymcld1?inspect`.

If the reactor executed, but did not launch your app, you can check the reactor logs:
```
abaco logs $ACTOR_ID $EXECUTION_ID
```

If the app launched, but you are not getting the output you expect, you can check the app logging. Run `jobs-list` to find the relevant job_ID, then you can run:
```
jobs-history $JOB_ID
jobs-output-list $JOB_ID
jobs-output-get $JOB_ID filename.err
cat filename.err
jobs-output-get $JOB_ID filename.out
cat filename.out
```
To get logs from the app execution. Good luck, and please reach out to us when you need help troubleshooting!

<iframe width="560" height="315" src="https://www.youtube.com/embed/zVa26lS4oIU" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
