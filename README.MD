This project provides a ready to deploy REST API allowing to predict if an image is offensive or has adult content.

It relies on [open_nsfw](https://github.com/yahoo/open_nsfw), which provides a pre-trained open source neural network model for [Caffe](https://github.com/BVLC/caffe).

The current project doesn't aim at improving the quality of the predictions. The main goal is to provide a ready to deploy
solution for people that might need this kind of service for free.

### Deploying

The python REST API, with Caffe and open_nsfw are packaged all together in a Docker image. So in theory you could just 
use an [existing build from docker hub] or build your own.

#### Using an existing build

TODO

#### Running locally

First you will need to install Docker for your OS. Then it should be as simple as running `docker-compose build` to build 
the image and `docker-compose up` to start it. You should be to test it at `http://localhost:5000/?url=https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png`.

#### Running on Heroku

A simple and quick option to get it up and ready for you to use from your website is via Heroku. You will need an Heroku 
account for that. Then it's as simple as:

 - Login to the Heroku container registry `heroku container:login`
 - Create an app if one doesn't already exist `heroku create YOUR_APP_NAME`
 - Upload the image `heroku container:push web`
 - Release it to your app `heroku container:release web` 

The service should be up and running. It might take a bit of time for the first request to be processed, though subsequent ones
should be faster.

#### Running in offline mode

For now running in batch/offline mode is outside of the scope of the project but any contribution to do so is welcome.

### Usage

You can use the classification API via two endpoints.


#### Single prediction

To get a prediction score for one single image just use the GET endpoint with a parameter named `url`.

```
curl -X GET -H 'Content-Type: application/json' http://localhost:5000\?url\=https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png

{
  "score": 0.00016061133646871895,
  "url": "https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png"
}
```

The response will be a json object with a `score` property having a value between 0 and 1, 1 meaning it is for sure
adult content while 0 it isn't. If there is an error while fetching the URL there will be two properties `error_code` 
and `error_reason` instead of the `score`:

```
{
  "error_code": 500,
  "error_reason": "[Errno -2] Name or service not known",
  "url": "https://foobar"
}
```


##### Batch predictions

Remark that doing the classification isn't a fast operation so you shouldn't call this API in places where you want the 
response in real time/low latency (for ex. to display it to the user), but instead call the API periodically. For that 
purpose use the batch classification API as follows.

```
curl -X POST -H 'Content-Type: application/json' \
 -d '{"images": [{"url": "http://foo.bar", "id": 1}, {"url": "https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png", "id": 2, "extra_props": {"foo": "bar"}}]}' \
 http://localhost:5000/batch-classify
 
{"predictions": [
{"url": "http://foo.bar", "error_reason": "[Errno -2] Name or service not known", "error_code": 500, "id": 1},
{"url": "https://www.google.com/images/branding/googlelogo/2x/googlelogo_color_272x92dp.png", "score": 0.00016061133646871895, "id": 2, "extra_props": {"foo": "bar"}}
]}
```

Each entry should have an `url` property pointing to the image accessible over HTTP/HTTPS. Any extra attribute will be 
preserved and sent back in the response. This allows easily to identify each entry and eventually pass along some context.

Note that the response is being streamed  back, so you could read it in a streaming fashion and process the results as 
they come in (as opposed to wait for the entire response before processing it). This is mostly done to work around
Heroku's 30s Request timeout.

Note that failing to process one entry in the batch will not fail the entire operation, instead the result for this single
entry will report the error. However if there is an error in handling the input JSON or some other global error the format
of the response is not guaranteed so you should check the response status code.


### License
Code licensed under the [BSD 2 clause license] (https://github.com/BVLC/caffe/blob/master/LICENSE). See LICENSE file for terms.
