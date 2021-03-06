# MUGicalNode

An attempt to implement my [MUGicalPHP calendar generation script](https://github.com/web-goddess/MUGicalPHP) to run as a collection of AWS Lambda serverless functions.

If you'd like to subscribe in your calendar application of choice, there are two options:

## Individual Events
These calendars include a separate event for every meetup. Depending on the city, that could involve a lot of events.
* https://krishoward.org/sydneymugs.ics
* https://krishoward.org/melbournemugs.ics
* https://krishoward.org/brisbanemugs.ics
* https://krishoward.org/perthmugs.ics
* https://krishoward.org/hobartmugs.ics

## Digests
These calendars have a single all-day event for each day that includes all of the meetups as a list within the description. For full details on a meetup, you'll have to click on the provided URL.
* https://krishoward.org/sydneymugs-digest.ics
* https://krishoward.org/melbournemugs-digest.ics
* https://krishoward.org/brisbanemugs-digest.ics
* https://krishoward.org/perthmugs-digest.ics
* https://krishoward.org/hobartmugs-digest.ics

## General
### Installation
The basis for this app came from Open Austin's [meetup-proxy-aws-lambda](https://github.com/open-austin/meetup-proxy-aws-lambda). I've broken it up into three separate lambda functions, each of which lives in its own directory within `src`:

* `src\getgroups`
* `src\getevents`
* `src\makecalendar`

After you clone the repo, you'll need to install some stuff:

```
npm install
npm install aws-sdk
npm install request-promise
npm install ical-toolkit
npm install striptags
```

You need to go to Meetup and [retrieve your API key](https://secure.meetup.com/meetup_api/key/). Then add a `secrets.json` file to each `src` directory that looks like this (being sure to paste in your key):

```
{
    "meetup_api_key": "{yourmeetupkey}"
}
```

### Testing

If you want to test out one of the lambdas, you'll need to first install the [AWS CLI](https://aws.amazon.com/cli/) to your machine.

Then update the `tests\index.js` file to point to the lambda you want to test.

Then just run:

```
npm run test
```

### Deployment

If you haven't created the lambdas yet, do that first. You can just use the "Author from scratch" option for each one. The runtime should be `Node.js 8.10`. The name of each lambda needs to match the name of a folder within `src`. I recommend setting the timeout to 3 minutes.

I recommend setting up a new execution role. You'll need to make sure this role has access to CloudWatch Logs, S3, SQS, and the relevant DynamoDB tables. (See below.)

From the directory on your machine, run this (being sure to substitute in the name of your lambda/folder:

```
./lambda.sh {nameoflambda}
```


## /src/getgroups

This lambda gets all the meetup groups associated with various topics in a given location and sends them to an SQS queue. This runs once per day at 1am.

### Setup
* You need to create an SQS queue called `meetupgroups`.
  * The retention period on the queue is set to 23 hours, which ensures that if something goes wrong, it gets purged before it runs again the next day.
  * The default visibility timeout on the queue is set to 5 minutes.
  * After 3 failed attempts to retrieve events, the group is sent to a dead letter queue.
* You need to set up a CloudWatch rule for each city to trigger the lambda to run on a cron schedule.
  * The target should be set to your lambda function.
  * You should configure the input to pass in a constant (JSON text): `{"targetlocation": "Brisbane"}`

## /src/getevents

This lambda retrieves a group from the queue and then calls the Meetup API to retrieve its upcoming events. These are saved to a DynamoDB database. This runs every minute, all the time.

### Setup
* You need to create a DynamoDB table called `meetups`.
  * You should enable the `Time to live attribute` and set the TTL attribute to `deletedate`.
* You need to set up a CloudWatch rule to trigger the lambda to run every minute.
  * The target should be set to your lambda function.

## /src/makecalendar

This lambda retrieves all the events for a given location from the database and turns them into two iCal calendars: one with lots of separate events and a digest that has one all-day event per day. These are uploaded to S3. This runs once per day at 11pm.

### Setup
* You need to create an S3 bucket to write to.
* You need to set up a CloudWatch rule for each city to trigger the lambda to run on a cron schedule.
  * The target should be set to your lambda function.
  * You should configure the input to pass in a constant (JSON text): `{"targetlocation": "Brisbane"}`


## /src/MUGicalNode - DEPRECATED

My first attempt at a direct implementation of the old version. It works, but the MeetupAPI really, _really_ doesn't like me hitting their API so much.

# License

The code for this repository has been released into the public domain by Kristine Howard via the Unlicense.

Originally based on Open Austin's [meetup-proxy-aws-lambda](https://github.com/open-austin/meetup-proxy-aws-lambda).

Created by [@web-goddess](https://github.com/web-goddess).
