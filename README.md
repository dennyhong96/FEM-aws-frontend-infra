## Useful Links

-   [Slides](https://speakerdeck.com/stevekinney/aws-for-frontend-engineers)
-   [Lambda@Edge Event Structure](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/lambda-event-structure.html)
-   [Mozilla Observatory](http://observatory.mozilla.org/)

## S3 Bucket Policy

This might be helpful at some point.

```json
{
	"Version": "2012-10-17",
	"Id": "Policy1618156787602",
	"Statement": [
		{
			"Sid": "Stmt1618156749891",
			"Effect": "Allow",
			"Principal": "*",
			"Action": "s3:GetObject",
			"Resource": "arn:aws:s3:::feawstest.tk/*"
		},
		{
			"Sid": "Stmt1618156785821",
			"Effect": "Allow",
			"Principal": {
				"AWS": "arn:aws:iam::598463965071:user/travis"
			},
			"Action": [
				"s3:PutObject",
				"s3:GetObject",
				"s3:DeleteObject",
				"s3:AbortMultipartUpload",
				"s3:GetObjectAcl",
				"s3:PutObjectAcl"
			],
			"Resource": "arn:aws:s3:::feawstest.tk/*"
		}
	]
}
```

## Travis Configuration

```yml
language: node_js # this is a node app
node_js:
    - "8" # use node 8 to build the app
cache:
    npm: true # cache npm dependencies
    directories:
        - node_modules # if deps haven't change, use cached node_modules
script:
    - npm run test # run the test suit, if test suit fails, the build will fail
before_deploy:
    - npm install -g travis-ci-cloudfront-invalidation # Install pkg for creating cloudfront invalidations
    - npm run build # build the app
deploy:
    provider: s3
    access_key_id: $AWS_ACCESS_KEY_ID # env vars
    secret_access_key: $AWS_SECRET_ACCESS_KEY
    bucket: $S3_BUCKET
    skip_cleanup: true # no need to clean up old files in s3
    region: us-west-2
    local-dir: dist # build dir
    "on":
        branch: master # only deploy on changes to master branch
after_deploy:
    - travis-ci-cloudfront-invalidation -a $AWS_ACCESS_KEY_ID -s $AWS_SECRET_ACCESS_KEY -c $CLOUDFRONT_ID -i '/*' -b $TRAVIS_BRANCH -p $TRAVIS_PULL_REQUEST # '*' invalidates everything
```

### Redirecting to `index.html` on Valid Client-side Routes

Viewer Request

```js
// Redirect Client Routes
exports.handler = async (event) => {
	// Viewer request - runs before request hits cloud front
	// One Cloudfront distribution can only have one viewer request trigger

	const request = event.Records[0].cf.request;

	console.log("Before", request.uri);

	// if url looks like a valid url "/notes/1"
	// change it to just index.html
	// which should be a cloudfront cache hit, legit 200
	if (/notes\/\d/.test(request.uri)) {
		request.uri = "/";
	}

	// Swap out a specific image
	if (request.uri === "/prince-1.jpg") {
		request.uri = "/prince-2.jpg";
	}

	console.log("After", request.uri);

	// Return the request object
	return request;
};
```

### Implementing Security Headers

Viewer Response

```js
// Modify Response Headers
exports.handler = async (event) => {
	const response = event.Records[0].cf.response;

	// When response leaves Cloudfront to viewers
	// Modify the response headers
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

	response.headers["server"] = [
		{
			key: "server",
			value: "quantum web server",
		},
	];

	return response;
};
```

### Implementing a Redirect

Origin Request

```js
// Server Side Redirect
exports.handler = async (event) => {
	// Get request obj from event
	const request = event.Records[0].cf.request;

	// Create a response obj for temporary redirect
	const response = {
		status: "302", // 'status' is the only required prop for response
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

	// if requset uri is '/secret', return the response to redirect
	if (request.uri === "/secret") {
		return response;
	}

	// else pass along the request to continue
	return request;
};
```

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
