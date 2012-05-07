# Real Time Foto Moderator

[RTFM](http://crowdflower.com/rtfm) is a [human-powered](http://crowdflower.com)
image moderation API.

## Overview

RTFM provides a [RESTful](http://en.wikipedia.org/wiki/Representational_state_transfer)
API that is designed to have resource-oriented URLs, use HTTP response codes to
indicate errors and use built-in HTTP features like HTTP authentication. All
responses will be returned in JSON.

The API endpoint is `https://rtfm.crowdflower.com/v1/`. All API requests must
be made over HTTPS. Requests made over plain HTTP will fail. Authentication is
required for all requests.

###  Authentication

Authentication to the API occurs via [HTTP Basic Auth](http://en.wikipedia.org/wiki/Basic_access_authentication).
Pass your API key as the basic auth username. You do not need to provide a
password.

API keys are managed in your account settings.

```shell
# curl uses the -u flag to pass basic auth credentials. Adding a colon after your
# API key will prevent curl from asking you for a password.

curl https://rtfm.crowdflower.com/v1/images \
  -u 7xgUnFVvHSsotiS6zaX2Nw1gcqwnBH0: \
  -d "url=http://example.com/test.jpg"
```

### Testing

RTFM provides a developer authentication key for testing. You can find this
key in your *Developer* subscription settings. All documented API calls work 
with your developer key. 

Images posted using this key are not sent to moderators in the crowd—they're
moderated based on substrings in the posted *url* parameter:

```ascii
| substring | url                             | moderation result                    |
|-----------+---------------------------------+--------------------------------------|
| approved  | http://example.com/approved.jpg | { "score": 1.0, "rating": approved } |
| rejected  | http://example.com/rejected.jpg | { "score": 0.1, "rating": rejected } |
| neither   | http://example.com/neither.jpg  | random result, either of the above   |
```

The substring can be anywhere in your domain, filename or query string.
A webhook will be posted to your webhook URL within several minutes.

### Errors

RTFM uses HTTP response codes to indicate success or failure of a request.
Codes in the 2xx range indicate success, codes in the 4xx range indicate an
error and codes in the 5xx range indicate an error with RTFM's servers.

* **200** OK - everything worked
* **401** Unauthorized - invalid API key provided
* **402** Payment required - insufficient funds on your account
* **404** Not found - the requested item doesn't exist
* **422** Unprocessable entity - invalid submitted data

### Rate limit

RTFM allows up to 10 requests per second from the same IP address. If you
exceed this limit, you'll get a *503 Service Unavailable* response for subsequent
requests.

## Images

Moderators screen images based on an extensive ruleset. See
[Moderation](#moderation) for the list of criteria.

##### **Score**

A confidence score, between 0 and 1, calculated from moderator responses.
0 is rejected, 1 is approved.

##### **Rating**

* **rejected**: score less than 0.5
* **approved**: score of 0.5 or greater

##### **States**

* **processing**: image is being moderated
* **on hold**: account has insufficient funds
* **completed**: image has completed moderation

### Posting

Image URLs must be publically accessible. A [webhook](#images/webhook) is sent
when moderation for an image is completed.

An optional metadata hash parameter can be posted with your image. You can use
this parameter for arbitrary data, e.g. an internal primary key. We will return
this metadata with your image data when we send a webhook or if you make API
call, requesting that image.

`POST /images` uploads a image:

```shell
# paramters:
#
# "url": string
# "metadata" (optional): hash

curl https://rtfm.crowdflower.com/v1/images \
  -u 7xgUnFVvHSsotiS6zaX2Nw1gcqwnBH0: \
  -d "url=http://example.com/test.jpg" \
  -d "metadata[internal_id]=Aj39x" \
  -d "metadata[internal_status]=spam"
```

Response:
```json
# attributes:
#
# image: hash
#   "id": string
#   "url": string
#   "metadata" (optional): hash

{
  "image": {
    "id": {IMAGE_ID},
    "url": {IMAGE_URL},
    "metadata": {IMAGE_METADATA}
  }
}
```

### Retrieving

`GET /images/{IMAGE_ID}` returns data about a posted image.

```shell
curl https://rtfm.crowdflower.com/v1/images/{IMAGE_ID} \
  -u 7xgUnFVvHSsotiS6zaX2Nw1gcqwnBH0:
```

Response:
```json
# attributes:
#
# image: hash
#   "id": string
#   "url": string
#   "score": integer
#   "rating": string
#   "state": string
#   "metadata" (optional): hash

{
  "image": {
    "id": {IMAGE_ID},
    "url": {IMAGE_URL},
    "score": {IMAGE_SCORE},
    "rating": {IMAGE_RATING},
    "state": {IMAGE_STATE},
    "metadata": {IMAGE_METADATA}
  }
}
```

### Webhook

We will POST moderation results to your webhook URL.
This URL is managed in your account settings.

```json
# attributes:
#
# image: hash
#   "id": string
#   "url": string
#   "score": integer
#   "rating": string
#   "state": string
#   "metadata" (optional): hash

{
  "image": {
    "id": {IMAGE_ID},
    "url": {IMAGE_URL},
    "score": {IMAGE_SCORE},
    "rating": {IMAGE_RATING},
    "state": {IMAGE_STATE},
    "metadata": {IMAGE_METADATA}
  }
}
```

#### Failure

RTFM expects a 2xx HTTP response code when posting webhooks to your server. Your
webhook will immediately be disabled if we receive a 3xx or 4xx response code.
RTFM will retry up to 10 times if your server returns a 5xx error. If there are
10 consecutive 5xx errors for a single image, we will disable the webhook and
notify you. All pending webhooks will be sent when you re-enable your webhook.

#### Signature

All webhook POSTs include a "X-CrowdFlower-Signature" HTTP header. You can use
this signature to ensure that you only accept requests from our servers.

The signature is composed of a Base64 encoded SHA1 digest of the JSON body and is
computed by signing the unescaped JSON body of the request with an RSA key.

Use [this public key](http://rtfm.crowdflower.com/webhook_public.pem) to verify
the signature.

Shell:
```shell
echo -n "unescaped JSON body of webhook request" > body.json
echo "base64 encoded contents of X-CrowdFlower-Signature" | openssl enc -base64 -d > sig.txt
openssl dgst -sha1 -verify webhook_public.pem -signature sig.txt body.json
```

Ruby:
```ruby
public_key = OpenSSL::PKey::RSA.new(File.read("webhook_public.pem"))
digest = OpenSSL::Digest::SHA1.new
public_key.verify(digest, Base64.decode64(signature), unencoded_json_body)
```

PHP:
```php
$fp = fopen("webhook_public.pem", "r");
$cert = fread($fp, 8192);
fclose($fp);
$pubkeyid = openssl_get_publickey($cert);
openssl_verify($data, $signature, $pubkeyid);
openssl_free_key($pubkeyid);
```

## API Libraries 

### Ruby

CrowdFlower currently maintains a Ruby gem for interacting with the API.  The gem
can be found here: https://github.com/dolores/rtfm-ruby

### Other

If you've created a simple wrapper for the RTFM API in another language, we would
be happy to host it on our Github account.  Let us know by emailing rtfm@crowdflower.com

## Moderation

Moderators follow extensive rules while screening images. These rules have been
effective in avoiding user complaints and problems with Apple's App Store
review process, specifically:

**prohibited sexual material or nudity**:

* sexual acts, either real or illustrated
* sexually explicit or overly suggestive photos
* nudity (frontal, back or side)
* nudity (particularly of the genitals) covered by a towel, hat or other means
* grabbing, holding or touching genitals or genital area
* transparent/sheer or wet material below the waist or covering women's nipples/breasts
* erections or outline of genitals through clothing
* bare skin one inch directly above the pubic area
* shirtless body shots indoors. shirtless body shots are only allowed in natural settings (e.g. beach or swimming pool)
* cleavage shots without a face
* pubic hair
* underwear, including underwear waistband showing above pants
* body/torso shots without a head/face
* crotch/butt/abs without a head/face
* display of semen or any fluid made to look like semen or ejaculate
* images edited to disguise sexual acts (e.g. black box or blurred filters)
* sex props or toys, including the use of fruits/vegetables
* any obscene gestures or lewd behavior (e.g. middle finger or banana used to simulate male genitalia)
* hands or fingers placed in pants or pulling underwear outward


**appropriate dress**:

* public swimwear is allowed if the following is observed: no pubic hair, no women's nipples, no outline of genitals and no portion of the butt can be present
* swimwear photos are only allowed if they are in natural settings (e.g. beach or swimming pool)
* pants and shorts must be worn normally, buttoned and not pulled or hanging down


**prohibited ads or copyrighted material**:

* advertised services, goods, events, websites or apps
* images or illustrations


**prohibited violent or illegal material**:

* illegal drugs, drug use or paraphernalia
* guns, firearms or weapons
* violent acts to the user, someone else or animals (including blood)
* profanity or curse words

## Contact
Report issues or suggest improvements on our
[GitHub repository](https://github.com/dolores/rtfm-api/issues) or email us at rtfm@crowdflower.com.
