---
layout: post
title: "On verbosity of express endpoints."
description: "Hacking with express middlewares for fun and profit"
tags: [node, express]
---
{% include image.html path="posts/bolongese.png" path-detail="posts/bolongese.png" alt="Javascript Bolongese" %}
***image borrowed from [flickr](https://www.flickr.com/photos/50158183@N00/388654424/in/photolist-AkXqc-AkXqm-AkXqf-4J9Q6r-jwvZYX-99rnRc-Axex2-9UQ4oT-bQq7CB-9jAyhU-52knk-ji45gE-8dX1z2-AMWU68-7MPr9T-bB2sEd-cnP9C-ji1KqL-7fYr6L-4YK6M-4JdYQU-4J9Anp-4JdJCQ-jhYWUT-eJa6B-ji45XE-4J9DcF-4W5mmn-4Je2gG-jwwUbo-b2pe4Z-hYjFox-6aJBsD-4J9FN4-be1AdV-yKC8qt-ojGLgz-nf3an6-jwuQ4k-jww2fK)***

There is a very high risk of writing JavaScript bolognese 
in express endpoints, and sometimes we can't notice how flexible express 
can be even with this minimalistic toolset that it gives out of the box. 
In this post I would like to share some thoughts about proper definition of express 
endpoint and what can be delegated to middlewares to make codebase as much reusable as possible. 
Before starting to read this small ***disclaimer***: these ideas is applicable 
in case when you know that that your express app will grow in terms of codebase 
if not this post will be more harmful rather than useful. 
After this quick warning moment let’s get started.

Sometimes you have some custom ecosystem of logging, error responses, success responses. 
Below you can see the most common express endpoint flow.

{% include image.html path="posts/express-request-flow.png" path-detail="posts/express-request-flow.png" alt="Endpoint Request Flow" %}

## Welcome to the spaghetti world

In any express project we always have endpoints, where we have the following flow: 
client initiates simple http request and expects some json data.
Under the hood we have bunch of operations such as asynchronous database calls, 
fetching some data from 3rd party api, logging success state, 
tracking data to some 3rd party apis and responding with success payload, 
in our example let’s assume that client expects son only when everything is ok.
 
 {% highlight javascript %}
 app.get('/data/:id/filtered', (req, res, next) => {
     const { id } = req.params;

     database.getById(id)
         .then((data) => {
             logger('[INFO] data is received');
             tracking('data.received', {id: id});
             return data;
         })
         .then((data) => dataFilterService({id: data}))
         .then((apiResult) => {
             logger('[INFO] data is filtered');
             tracking('api.result.received');
             res.json(apiResult);
         })
         .catch((error) => {
             logger('[ERROR] ', error.message);
             exceptionCatcher(error);
             res.status(500).json({
                 error: 'Something went wrong'
             });
         });
 });
 {% endhighlight %}
 
 Looking at this small example you may notice that some 
 parts of this endpoints leaks more details that may seems that 
 it is out this endpoint responsibility?
 
 After working with dozens of different express apps 
 I have noticed that middleware layer is pretty underused, almost everywhere 
 it is used only as some tiny request processing tool. 
 Yes obviously it is intended to do such thing and it does it pretty well but sometimes 
 I can see that there is a bit chance to produce code duplications and start testing 
 more than needed. What express routes intended to do? 
 Does express route need to know more than a single thing, 
 how can we leverage single responsive principle in our example?
 In our first example the responsibility of the endpoint 
 is to send processed data entry by certain identifier provided by client. 
 If we have only this endpoint this code looks more or less ok. 
 Let’s assume that we also need to fetch raw data entry 
 without initiating any processing service.
 
{% highlight javascript %}
app.get('/data/:entry_id', (req, res, next) => {
    const { entry_id } = req.params;
    
    database.getById(id)
        .then((data) => {
            return res.json(data);
        })
        .catch((error) => {
            logger('[ERROR] ', error.message);
            exceptionCatcher(error);
            res.status(500).json({
                error: 'Something went wrong'
            });
        });
});
{% endhighlight %}

As you may notice we are introducing more complexity in our codebase by adding duplication 
which sounds like a great target for refactoring. How can we DRY this endpoints.
First of all let’s pay attention to error handling in both examples? 
You probably may notice that first we can do is provide utility 
function for error handling and require it in both modules. 
But as we are using express why no to leverage built-in middleware layer in this case.

{% highlight javascript %}
app.use(function(err, req, res, next) {
    logger('[ERROR] ', error.message);
    exceptionCatcher(error);
    res.status(500).json({
        error: 'Something went wrong'
    });
});
{% endhighlight %}

## What did we achieve after that?
 
First of all we got rid out of one external dependency 
that we may stub in our tests, and obviously 
this dependency tell us nothing about real intention of this endpoint. 
When you are ordering a cup of coffee your outcome should be mug of coffee and you don’t 
want to hear from they guy who helps barista to repair coffee machine because of N reasons, 
in our case repair guy is external dependency that we are encapsulating in middleware 
that does one thing and does it properly.
Let’s look at the outcome.

{% highlight javascript %}
app.get('/data/:entry_id', (req, res, next) => {
    const { entry_id } = req.params;
    
    database.getById(id)
        .then((data) => {
            logger('[INFO] data is received');
            tracking('data.received', {id: id});
            return res.json(data);
        })
        .catch(next);
});

app.get('/data/:entry_id/filtered', (req, res, next) => {
    const { entry_id } = req.params;
    
    database.getById(id)
        .then((data) => dataFilterService({id: data}))
        .then((apiResult) => {
            logger('[INFO] data is filtered');
            tracking('api.result.received');
            res.json(apiResult);
        })
        .catch(next);
});

{% endhighlight %}

This refactoring looks very tiny but it doesn't mean that benefit is tiny as well,
in this case you don’t need to test things that are out of ones responsibility.

### What can we do next?
You may noticed that fetching from params and next 
from database is also looks like copy paste driven development, 
which will also may cause possible issues.
Let’s assume that we plan to add more endpoints that will 
lookup by entry_id parameter, will we constantly copy and paste this data fetching behaviour? 
Is there any express way to solve this problem? 
If we pay attention to express documentation express gives us nifty feature 
of declaring [param](http://expressjs.com/en/api.html#app.param) based middleware 
why not to use this? But before let’s think about conceptual idea of request lifecycle in express app.

{% include image.html path="posts/express-request-object-flow.png" path-detail="posts/express-request-object-flow.png" alt="Request Oject Flow" %}

In express we can have as much middlewares as it is is possible which means that we can pre-process our 
request object adding additional keys and values before it goes to final destination e.g. client.
So what does it mean for us? It means that we can make loose 
coupled middleware functions that can be reused all over our application logic 
which will lead us to less code when our system will be extended.

So let’s define our param handler:

{% highlight javascript %}
app.param('entry_id', (req, res, next, entry_id) => {    
    getDataById(entry_id)
        .then((data) => {
            req.metadata = {
                info: 'data is received',
                event: 'data.received'
            };
            
            req.result = data;

            next();
        })
        .catch(next);
});
{% endhighlight %}

Looking at this small code you may notice that we provided additional 
metadata to our request object plus mixed in result of database query. 
So after we extracted common data fetching from route param we can extract common 
code to DRY our express endpoints and the result will look like this:

{% highlight javascript %}
app.get('/data/:entry_id', (req, res, next) => {
    next();
});

app.get('/data/:entry_id/filtered', (req, res, next) => {    
    dataFilterService(req.result)
        .then((apiResult) => {
            req.metadata = {
                info: 'data is filtered',
                event: 'api.result.received'
            }
            
            req.result = apiResult;
            next();
        })
        .catch(next);
});
{% endhighlight %}

After this changes applied we removed bunch of duplications 
from both our endpoints plus we have introduced single responsibility principle, 
our endpoints does only one thing.
But where is response and logging and tracking you may ask? 
And here is the most controversial part of this post. 
According to our diagram we can pass request using by calling 
next function to other middlewares and work with our 
request until we hit req.end or finalise its lifecycle in one of our middlewares.

{% highlight javascript %}
// tracker middleware
app.use((req, res, next) => {
    if (req.metadata.event) {
        tracking(req.metadata.event);
        next();
    }
});

// logger middleware
app.use('/data', (req, res, next) => {
    if (req.metadata.info) {
        logger(req.metadata.info);
        next();
    }
});

// responder middleware
app.use((req, res, next) => {
    if (req.result) {
        res.json(req.result);
    } else {
        next(new Error('Result not found'));
    }
});
{% endhighlight %}
So after this changes we will have decoupled tiny middlewares and we can and make 
our subsystems as much reusable as possible. 
I’m not sure that it is the best idea ever 
but in terms of testability of each subcomponent 
it looks more than reasonable to do this for bigger express applications.

If you are interested and wan’t to play with code examples you can use this [gist](https://gist.github.com/strangeworks/fe9b279ca6c00f0923cad94135af2de6), 
thank you for your time spent of this long read, I hope it was useful 