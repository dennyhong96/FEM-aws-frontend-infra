# Frontend Masters: AWS for Frontend Engineers

You should have the following completed on your computer _before_ the workshop:

-   Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/installing.html).
-   Have [Node.js](https://nodejs.org/en/) installed on your system. (Recommended: Use [nvm](https://github.com/creationix/nvm).)
    -   Install `yarn` with `brew install yarn`.
-   Create an [AWS account](https://portal.aws.amazon.com/billing/signup#/start). (This will require a valid credit card.)
-   Create a [Travis CI](https://travis-ci.org/) account. (This should be as simple as logging in via GitHub).

**Follow-Along Guides**: I made a [set of visual follow-along guides](https://www.dropbox.com/sh/thuoclvoj3r9nut/AADAA5rUqF5awNVxjyFLoh55a?dl=0) that you can reference throughout the course.

## Repositories

-   [Noted (Base)](https://github.com/stevekinney/noted-base): This is the base that you can clone and work with.
-   [Noted (Live)](https://github.com/stevekinney/noted-live): This is the live version that I'm going to be working with.

## Useful Links

-   [Slides](https://speakerdeck.com/stevekinney/aws-for-frontend-engineers)
-   [Lambda@Edge Event Structure](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-event-structure.html)
-   [Mozilla Observatory](http://observatory.mozilla.org/)

## S3 Bucket Policy

This might be helpful at some point.

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::YOUR_BUCKET_NAME_HERE/*"
		}
	]
}
```

## Travis Configuration

```yml
language: node_js
node_js:
    - "8"
cache:
    yarn: true
    directories:
        - node_modules
script:
    - yarn test
before_deploy:
    - yarn global add travis-ci-cloudfront-invalidation
    - yarn run build
deploy:
    provider: s3
    access_key_id: $AWS_ACCESS_KEY_ID
    secret_access_key: $AWS_SECRET_ACCESS_KEY
    bucket: $S3_BUCKET
    skip_cleanup: true
    local-dir: dist
    on:
        branch: master
after_deploy:
    - travis-ci-cloudfront-invalidation -a $AWS_ACCESS_KEY_ID -s $AWS_SECRET_ACCESS_KEY -c $CLOUDFRONT_ID -i '/*' -b $TRAVIS_BRANCH -p $TRAVIS_PULL_REQUEST
```

### Redirecting to `index.html` on Valid Client-side Routes

Viewer Request

```js
exports.handler = (event, context, callback) => {
	const request = event.Records[0].cf.request;

	console.log("Before", request.uri);

	if (/notes\/\d(\/edit)?/.test(request.uri)) {
		request.uri = "/index.html";
	}

	console.log("After", request.uri);

	callback(null, request);
};
```

### Implementing Security Headers

Viewer Response

```js
"use strict";

exports.handler = (event, context, callback) => {
	const response = event.Records[0].cf.response;

	response.headers["strict-transport-security"] = [
		{
			key: "Strict-Transport-Security",
			value: "max-age=31536000; includeSubDomains",
		},
	];

	response.headers["content-security-policy"] = [
		{
			key: "Content-Security-Policy",
			value: "default-src 'self'",
		},
	];

	response.headers["x-xss-protection"] = [
		{
			key: "X-XSS-Protection",
			value: "1; mode=block",
		},
	];

	response.headers["x-content-type-options"] = [
		{
			key: "X-Content-Type-Options",
			value: "nosniff",
		},
	];

	response.headers["x-frame-options"] = [
		{
			key: "X-Frame-Options",
			value: "DENY",
		},
	];

	callback(null, response);
};
```

### Implementing a Redirect

Origin Request

```js
exports.handler = (event, context, callback) => {
	const request = event.Records[0].cf.request;

	const response = {
		status: "302",
		statusDescription: "Found",
		headers: {
			location: [
				{
					key: "Location",
					value: "https://bit.ly/very-secret",
				},
			],
		},
	};

	if (request.uri === "/secret") {
		console.log("Got it");
		return callback(null, response);
	}

	console.log("Nope");
	callback(null, request);
};
```
