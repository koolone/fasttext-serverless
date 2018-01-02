<!--
title: fastText prediction in Python for Serverless AWS
description: HTTP POST endpoint returns topic recommendations using a fastText model.
layout: Doc
-->
# fasttext-serverless
Serverless hashtag recommendations using fastText and Python with AWS Lambda.

A simple HTTP POST endpoint that returns hashtag recommendations. This function requires a pre-trained fastText model. When you send a properly formatted string in the body of a POST to this endpoint, it will reply with JSON containing up to 5 topic recommendations that it believes match that string. It will also identify and return a list of hashtags that are *already* included in the submitted text (so you can handle collisions if you want to). While the internal function is named `tagRecommendations` the HTTP endpoint is exposed as `recommendations`.


## Setup

**Step 1: Clone this repo**

```bash
$ git clone https://github.com/CareerVillage/fasttext-serverless/
```

**Step 2: Install and configure Serverless**
Serverless is a node.js tool that will allow you to easily deploy this project to a service such as AWS Lambda. [The docs for installing Serverless](https://serverless.com/framework/docs/providers/aws/guide/installation/) and [the docs for configuring it with credentials for your AWS account](https://serverless.com/framework/docs/providers/aws/guide/installation/) are excellent in case you run into trouble. For example, to install Serverless and configure it for AWS:

```bash
$ npm install -g serverless
$ serverless config credentials --provider aws --key AKIAIOSFODNN7EXAMPLE --secret wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**Step 3: Add your pre-trained classification model**
Train a supervised fastText model and add that model to the `trained_models/` folder in this project folder. For more information about how to train a fastText model check out [the fastText docs](https://fasttext.cc/docs/en/cheatsheet.html). Here's a hint:

```bash
$ ./fasttext supervised -input train.txt -output model_standard
$ mv model_standard.bin /path/to/fasttext-serverless/trained_models/model_standard.bin
```
_Note 1: If you don't use model_standard as your model name, you'll need to change `handler.py` to point to the filename you used_
_Note 2: There are upper limits to how large a serverless function can be on AWS Lambda. If your fasttext model is very big, you might get errors while trying to deploy. In that case, consider using fastText's `quantize` to reduce the memory footprint of your model._


**Step 4: Deploy to AWS**
Assuming you have properly configured Serverless to access AWS, to deploy the endpoint (with verbose logs) simply run

```bash
$ serverless deploy -v
Serverless: Packaging service...
Serverless: Excluding development dependencies...
Serverless: Uploading CloudFormation file to S3...
Serverless: Uploading artifacts...
Serverless: Uploading service .zip file to S3 (27.71 MB)...
Serverless: Validating template...
Serverless: Updating Stack...
Serverless: Checking Stack update progress...
..............
Serverless: Stack update finished...
Service Information
service: lambda
stage: dev
region: us-east-1
stack: lambda-dev
api keys:
  None
endpoints:
  POST - https://{your-subdomain-here}.execute-api.{your-region-code-here}.amazonaws.com/dev/recommendations
functions:
  tagRecommendations: lambda-dev-tagRecommendations
Serverless: Removing old service versions...
```


## Usage

You can now send an HTTP POST request directly to the endpoint. For example using curl you might do:

```bash
curl -X POST https://{your-subdomain-here}.execute-api.{your-region-code-here}.amazonaws.com/dev/recommendations --data '{ "text": "What should I do in the evenings and weekends during high school to become a pediatrician? I want to become a doctor after college so that I can help children recover from terrible diseases and illnesses. #doctor #medicine" }'
```

The expected result should be similar to:

```bash
{"hashtags_already_used": "#doctor #healthcare #medicine", "hashtags_recommended": "('__label__doctor 0.662109 __label__pediatrician 0.0585938 __label__medicine 0.015625 __label__pre-med 0.0136719 __label__surgeon 0.0136719\\n', '')"}
```


## Updating fastText assets

### Updating the fastText binary
The fastText binary used here was built using:

```bash
wget https://github.com/facebookresearch/fastText/archive/v0.1.0.zip
$ unzip v0.1.0.zip
$ cd fastText-0.1.0
$ make
```

To update the fastText binary ssh into a running EC2 instance (which is running an Amazon Linux AMI), follow the instructions at https://github.com/facebookresearch/fastText to update to the latest version of fastText so you can `make` the binary, and then copy (`scp`) the binary file into the folder for this repo.

### Updating the classification model file
To update the model_standard.bin file, you must have training data properly formated for fastText training (e.g., `training_set.txt` cleaned in the same way as on the machine doing prediction. For example, for CareerVillage we remove all punctuation, remove all HTML tags, and lowercase all characters) and for optimal results, you should also have a local copy of the wikipedia-based english language word vectors file provided by fastText (`wiki.en.vec`). Training is completed with the following parameters: `./fasttext supervised -input ./data/questions_set_for_training.txt -output model -pretrainedVectors ./data/wiki.en.vec -verbose 2 -lr 1.0 -epoch 20 -dim 300 -wordNgrams 2 -neg 10 -bucket 10000`. If you use the pretrained vectors, your model will almost certainly be too large for AWS Lambda, so you will need to use fastText's `quantize` to reduce the filesize. More information is available at https://github.com/facebookresearch/fastText#text-classification 


## Scaling

By default, AWS Lambda limits the total concurrent executions across all functions within a given region to 100. The default limit is a safety limit that protects you from costs due to potential runaway or recursive functions during initial development and testing. To increase this limit above the default, follow the steps in [To request a limit increase for concurrent executions](http://docs.aws.amazon.com/lambda/latest/dg/concurrent-executions.html#increase-concurrent-executions-limit).


## References

- Using fastText in the command line: https://github.com/facebookresearch/fastText (< This was used to train a tag recommendation model. That training is conducted offline. Not part of this Serverless AWS Lambda function.)
- Using Serverless to control AWS Lambda: https://serverless.com/framework/docs/providers/aws/guide/quick-start/ (< What we're using to control setting up, deploying, and developing on AWS Lambda)
- Boilerplate for setting up a Python GET endpoint in Serverless: https://github.com/serverless/examples/tree/master/aws-python-simple-http-endpoint (< I used this example as a starting point.)
- Using fastText with Lambda: https://blog.codefor.cash/2017/10/19/an-aws-lambda-serverless-implementation-of-facebook-fasttext-text-classification/ (< The part of this tutorial I used was the step of creating a fastText binary that will work in AWS Lambda by creating an EC2 instance with an Amazon Linux AMI, installing the dependencies, and then using `make` to build the binary. I am also using the same method to invoke the fasttext binary from inside the python function.)


## License
Please refer to the LICENSE file for license information applying to everything in this project except for the fastText binary. The license for the fastText binary is in the LICENSE_FASTTEXT file.
