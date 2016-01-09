# Nestorbot Programming Manual

If you want a quick (~4 min) overview of the Nestor programming environment, watch the video here:

[![Nestor Video](http://img.youtube.com/vi/ZuJAFkgWMPw/0.jpg)](http://www.youtube.com/watch?v=ZuJAFkgWMPw)


Nestor can be programmed using Javascript to create a powerful bot for
your team that can automate away the mundane.

Nestor's API is based on [Hubot's API](https://hubot.github.com/docs/scripting/) and it tries to be as compatible as
possible with Hubot's API -- The Nestor Programming Manual has also been adapted from the Hubot documentation.

Nestor is powered by "apps" that are analogous to Hubot scripts.

## Structure of an App

The entrypoint to an app is either a `respond` or a `hear` method that
belongs to a `robot` instance. A `hear` method is invoked for every
message that is received in a channel whereas `respond` is invoked when
a message is addressed to nestorbot itself (for e.g. "@nestorbot what's
the weather now?")

Both methods take a regular expression and a callback function as
parameters.

For example:

```javascript

robot.hear(/gif( me)? (.*?)$/i, function(msg) {
  // your code here
});

robot.respond(/hello/i, function(msg) {
  // your code here
});
```

The `robot.hear(/gif( me)? (.*?)$/i` callback is called anytime a message's text matches. For example:

* gif me honey badger
* gif fail whale
* give me a gif star wars


The `robot.respond /hello/i` callback is only called for messages that are immediately preceded by nestorbot's nickname.

* @nestorbot: hello
* hello @nestorbot

It wouldn't be called for:

* nestorbot hello
  * because it doesn't contain the Slack User ID for nestorbot
* hello
  * because it is not addressed to nestorbot

## Send and Reply

The `msg` parameter is an instance of `Response`. With it, you can `send` a message back to the channel the `msg` came from, or `reply` to the person that sent the message. For example:

```javascript
robot.hear(/gif me honeybadger/i, function(msg) {
  msg.send("https://media.giphy.com/media/aui0RuYoqtRa8/giphy-facebook_s.jpg")
});

robot.respond(/weather in san francisco/i, function(msg) {
  msg.reply("sunny and awesome");
});
```
The `robot.hear /gif me honeybadgers/` callback sends a message exactly as specified regardless of who said it, "https://media.giphy.com/media/aui0RuYoqtRa8/giphy-facebook_s.jpg"

If a user Bob says "@nestorbot: weather in san francisco", `robot.respond /weather in san francisco/i` callback sends a message "@bob: sunny and awesome"

## Capturing data

So far, our apps have had static responses, which while amusing, are boring functionality-wise. `msg.match` has the result of `match`ing the incoming message against the regular expression. This is just a [JavaScript thing](http://www.w3schools.com/jsref/jsref_match.asp), which ends up being an array with index 0 being the full text matching the expression. If you include capture groups, those will be populated `res.match`. For example, if we update an app like:

```javascript
robot.respond(/weather in (.*)/i, function(msg) {
  // your code here
});
```

If Bob says "@nestorbot: weather in san francisco", then `res.match[0]` is "weather in san francisco", and `res.match[1]` is just "san francisco". Now we can start doing more dynamic things:

```javascript
robot.respond(/weather in (.*)/i, function(msg) {
  location = msg.match[1];

  if(location == "san francisco") {
    msg.reply("sunny and awesome");
  } else {
    msg.reply("i don't know");
  }
});
```

## Random

A common pattern is to hear or respond to commands, and send with a random funny image or line of text from an array of possibilities.

```javascript
greetings = ['hi there!', 'hello there', 'aloha'];

msg.send(msg.random(greetings));
```

## Making HTTP Calls

Nestorbot can make HTTP calls on your behalf to integrate & consume third party APIs. This can be through an instance of `HttpClient` available at `robot.http`. The simplest case looks like:


```javascript
robot.http("https://my-favorite-website").get()(function(err, res, body) {
  // Your code here
});
```

A post looks like:

```javascript
var data;

data = JSON.stringify({
  foo: 'bar'
});

robot.http("https://my-favorite-website").
      header('Content-Type', 'application/json').
      post(data)(function(err, res, body) {
        // Your Code Here
      });
```


`err` is an error encountered on the way, if one was encountered. You'll generally want to check for this and handle accordingly:

```javascript
robot.http("https://my-favorite-website").get()(function(err, res, body) {
  if (err) {
    msg.send("Encountered an error :( " + err);
    return;
  }

  // Your code here -- knowing the HTTP call was successful
});
```

`res` is an instance of node's [http.ServerResponse](http://nodejs.org/api/http.html#http_class_http_serverresponse). Most of the methods don't matter as much when using `HttpClient`, but of interest are `statusCode` and `getHeader`. Use `statusCode` to check for the HTTP status code, where usually non-200 means something bad happened. Use `getHeader` for peeking at the header, for example to check for rate limiting:

```javascript
robot.http("https://my-favorite-website").get()(function(err, res, body) {
  var rateLimitRemaining;
  if (res.statusCode !== 200) {
    msg.send("Request didn't come back HTTP 200 :(");
    return;
  }
  if (res.getHeader('X-RateLimit-Limit')) {
    rateLimitRemaining = parseInt(res.getHeader('X-RateLimit-Limit'));
  }
  if (rateLimitRemaining && rateLimitRemaining < 1) {
    return msg.send("Rate Limit hit, stop believing for awhile");
  }

  // Rest of your code
});

```

`body` is the response's body as a string, the thing you probably care about the most:

```javascript
robot.http("https://my-favorite-website").get()(function(err, res, body) {
  return msg.send("Got back " + body);
});
```

### JSON

If you are talking to APIs, the easiest way is going to be JSON because it doesn't require any extra dependencies. When making the `robot.http` call, you should usually set the  `Accept` header to give the API a clue that's what you are expecting back. Once you get the `body` back, you can parse it with `JSON.parse`:

```javascript
robot.http("https://my-favorite-api").
      header('Accept', 'application/json').
      get()(function(err, res, body) {
  var data;
  data = JSON.parse(body);
  msg.send(data.key + " value is " + data.value);
});
```

It's possible to get non-JSON back, like if the API hit an error and it tries to render a normal HTML error instead of JSON. To be on the safe side, you should check the         `Content-Type`, and catch any errors while parsing.

```javascript
robot.http("https://my-favorite-api").
      header('Accept', 'application/json').
      get()(function(err, res, body) {
  var data, error;
  if (response.getHeader('Content-Type') !== 'application/json') {
    msg.send("Didn't get back JSON :(");
    return;
  }

  data = null;
  try {
    return data = JSON.parse(body);
  } catch (_error) {
    error = _error;
    msg.send("Ran into an error parsing JSON :(");
  }
});
```

### XML

XML is not supported at the moment

## Environment Variables

If you use API keys and other secrets in your app, it's not a good idea to hard code them in your app code. Nestorbot allows you to have environment variables in your app so that code & data can be kept separate. You can set environment variables in the app by going to the section titled "Environment" for each app.

```javascript
robot.respond(/what is the answer to the ultimate question of life/, function(msg) {
  answer = env.HUBOT_ANSWER_TO_THE_ULTIMATE_QUESTION_OF_LIFE_THE_UNIVERSE_AND_EVERYTHING;
  msg.send(answer + ", but what is the question?");
});
```
We can also check if the environment variable is set and respond accordingly.

```javascript
robot.respond(/what is the answer to the ultimate question of life/, function(msg) {
  answer = env.HUBOT_ANSWER_TO_THE_ULTIMATE_QUESTION_OF_LIFE_THE_UNIVERSE_AND_EVERYTHING;
  if (answer == null) {
    msg.send("Missing HUBOT_ANSWER_TO_THE_ULTIMATE_QUESTION_OF_LIFE_THE_UNIVERSE_AND_EVERYTHING in environment: please set and try again");
    return;
  }

  msg.send(answer + ", but what is the question?");
});
```

## Persistence

Nestorbot has a persistent key-value store exposed as `robot.brain` that can be
used to store and retrieve data by apps.

```javascript
robot.respond(/have a soda/i, function(msg) {
  var sodasHad;
  sodasHad = robot.brain.get('totalSodas') * 1 || 0;
  if (sodasHad > 4) {
    return msg.reply("I'm too fizzy..");
  } else {
    msg.reply('Sure!');
    return robot.brain.set('totalSodas', sodasHad + 1);
  }
});

robot.respond(/sleep it off/i, function(msg) {
  robot.brain.set('totalSodas', 0);
  return msg.reply('zzzzz');
});
```

If your app to lookup user data, there are methods on `robot.brain` for looking up one or many users by id, name, or 'fuzzy' matching of name: `userForName`,             `userForId`, and `usersForFuzzyName`.

```javascript
robot.respond(/who is @?([\w .\-]+)\?*$/i, function(msg) {
  var name, user, users;
  name = msg.match[1].trim();
  users = robot.brain.usersForFuzzyName(name);
  if (users.length === 1) {
    user = users[0];
    return msg.send(name + " is user - " + user);
  }
});
```

## Feature Requests / Bug Reports / Corrections

If you have feature requests, bug reports in both the app as well as the manual, please [raise an issue](https://github.com/zerobotlabs/asknestor-docs/issues). If you have suggestions for improving the manual, please [open a pull request](https://github.com/zerobotlabs/asknestor-docs/pulls).
