---
layout: post
title:  "Really Bad Bugs - The auth no-op."
date:   2020-07-03 15:48:20 -0400
categories: really-bad-bugs debugging
---

# Four Types of Programmers #

If you work in software development for long enough you will likely encounter one 
or more of these types of programmers:
* programmers who don't have time for version control
* programmers who won't have time for IDEs
* programmers who don't have time for code written by anyone else
* programmers who don't have time for code reviews

It's a good week if you don't have to deal with any of these types or their code. 
It's probably a bad week if you have to deal with more than one. (This week I had 
to deal with three.)

I think more often than not, these programmers stay true to type, but you could argue 
that these aren't types of programmers, but bad habits that 
programmers at various times might exhibit. I think the key here is "don't have the time." 
Time pressures can make programmers 
do bad things, like not writing tests, leaving out doc and comments, and skipping 
code reviews that they might not be fully convinced of the value of. But
"don't have the time" can also mean they can't be bothered to make the effort, that the 
perceived costs of going through a code review would be greater than the benefits for them 
or the project or is less valuable than other things they could be doing. 

I don't like arguing with developers who are determined not 
to do follow best practices, but I think it's widely accepted that better tools and 
procedures will lead to better code, will save time later on. More eyes on a project 
means more developers who can potentially develop on 
that project. Etc. This is all an intro to my real point. Today I encountered a 
Really Bad Bug (TM) that I contend wouldn't have been a bug if code review had 
been done.

# Context #

We have a public facing API used for syncing metadata with partnering archives.  Calls to these
APIs look something like this:

    GET /changed-records?start-date=2020-01-01&end-date=2020-01-31
    GET /deleted-records?start-date=2020-01-01&end-date=2020-01-31
    
Each returns records of the form: `doc-type identifer date md5sum` for any records
that have been changed or added. This data isn't proprietary, but it's not public 
and generally we want to limit access to it. When we want to limit access to an API, 
typically we use API keys passed in the request header.

This particular API was written by a developer who has since moved on to another job. 
When that developer left another developer (not me) was assigned ownership of 
the project, but it has 
mostly remained unchanged. We recently switched host names for this service and 
had to update the users with the new endpoint URL. Because I was unlucky enough
to be at a meeting where this switch was discussed, I was asked to figure out if any 
changes to the API keys were necessary.

# The Investigation #

I wasn't familiar with this code so I made the following plan of attack:
1. Meet with the developer who owns this project and another developer who has 
overall expertise in the code base.
2. Use the service. Can I call the API with the already configured API keys? If 
so, then problem solved.
3. Look at the code.

**Step 1. Meet with the developers.** During this entire meeting the developer who 
owned the project didn't say a word. Bad sign. Expert developer was *pretty sure* 
that either API keys or IP whitelisting was protecting this interface and the 
settings wouldn't change, but not certain.

**Step 2. Try the service.** The existing documentation for this service was out 
of date. The doc had sample API calls (good) but they didn't work (bad). Luckily 
expert developer had just worked another ticket where he had documented some 
sample calls so was able to point me to that. I try the service and surprisingly 
I can access all endpoints with no auth key. I use the traditional developer 
method of double checking on my phone. This interface appears to be unrestricted.

**Step 3. Look at the code.** Lack of up-to-date API doc and lack of understanding 
means I am now reading the code. It's Thursday. It's the 4th day in a row that I'm 
not getting any of my own work done because of interrupts and meetings. I have to 
context switch from Python .NET/C#. This is not what I want to be doing with my 
time. I take notes to trace through the logic of the authorization code. I grep 
through the code and find the place where the access check is happening. 
It's using [netsqlazman](https://archive.codeplex.com/?p=netsqlazman) (.NET 
SQL Authorization Management)[^footnote] to authorize users and falling back on API 
keys when that fails. I do not understand netsqlazman but I know it uses a database, 
so we take a detour to...

**Step 3.5. Look at the authorization database.** This detour is awful. These are auto-generated 
tables with a bunch of views that aren't any more helpful. I can find a table that 
shows that some set of rules exist for authorizing users for this service, but not 
what the rules are. I give up.

At this point I still don't understand what is going on. We have an API that is 
apparently protected by an authorization tool and API keys, but in practice it's 
open to the world. The old server and the new server have the same configuration
and are using the same authorization database so I'm reasonably certain that if 
users can access the API on the old server they will be able to access it on the 
new server. But I'm pretty sure there's no entry for my cell phone's IP 
address in the netsqlazman DB, so something is obviously going wrong.

# The Solution #

I lied when I listed the above steps and said there were only three. There was a 4th 
step all along, and that was:

<ol start="4">
<li>Debug the code</li>
</ol>

Debugging this code is a pain. It involves using remote desktop to connect to a Windows 
server where I can run Visual Studio, creating a publishing profile to build and deploy 
my own instance of the code in Debug mode, restarting VS when I forget to run it in 
administrator mode, starting a new instance of remote debugger on the other server 
because it is running as a service but not in Admin mode, figuring out the process 
number of the application pool to connect to.

And then my bug points aren't hitting. It takes me a few times stepping through the code
to find out why. The reason is below.

{% highlight csharp %}
protected internal void AuthorizeClient(Authz authObj, string service)
{
    return;
    bool authz = authObj.checkForUserOrIPAuthorization (service);

    if (authz == false)
    {
        if (authByHeader(HEADER_TYPE))
        {
            authz = true;
        }
    }

    if (authz) {
        Logger.Info("Client Authorized");
    } else
    {
        Logger.Info("Client Unauthorized");
        string msg = "Not authorized to access this service.";
        throw new HttpException((int)HttpStatusCode.Unauthorized, msg);
    }
}
{% endhighlight %}

If you don't see it---and I didn't see it until after talking to the experts, reading through 
the code, then running the code in the debugger several times---let me present a different version:

{% highlight csharp %}
protected internal void AuthorizeClient(Authz authObj, string service)
{
    return;
    /*
    bool authz = authObj.checkForUserOrIPAuthorization (service);

    if (authz == false)
    {
        if (authByHeader(HEADER_TYPE))
        {
            authz = true;
        }
    }

    if (authz) {
        Logger.Info("Client Authorized");
    } else
    {
        Logger.Info("Client Unauthorized");
        string msg = "Not authorized to access this service.";
        throw new HttpException((int)HttpStatusCode.Unauthorized, msg);
    }
    */
}
{% endhighlight %}

The authorization code has been turned into a no-op by adding a `return` statement at the top of 
the function. There are over 1000 lines of code in just this one file and I missed the most 
important one. 

# Would code review have fixed this problem? #

There are several signs here to me that this code didn't go through proper process. First of all, 
a decent linter would have found this issue right away and if properly gated, the code wouldn't
have been deployed. The simplest improvement that I can suggest for this code is the one I made 
above. Comment out the dead code! (A linter may also be unhappy with an empty function.) The second
simplest improvement to this code is to add comments that explain why this change was made. Here
is how the no-op authorization function is being called elsewhere in the code:

{% highlight csharp %}
[System.Web.Http.HttpGet]
public HttpResponseMessage Availability()
{

    // Check that this was called from an authorized IP or user (ezid).
    try
    {
        AuthorizeClient(GetAuthorizationObject(), SERVICE_NAME);
    }
    catch (HttpException e)
    {
        Logger.Info("Caller was unauthorized for Service", e);
        return new HttpResponseMessage()
        {
            Content = new StringContent(
                "User not authorized",
                Encoding.UTF8,
                "text/plain"
            ),
            StatusCode = HttpStatusCode.Unauthorized
        };
    }
    ...
}
{% endhighlight %}

The comments haven't been updated. There are six API endpoints that call this function.
No comments on any of them. The only reason I knew this wasn't a mistake is because the
commit that caused this change had a comment and was linked to a ticket. (This is one 
great reason to use version control.)

Additionally, this code is weird. The Authorization library is not a web interface, but 
it's throwing an HTTPException to indicate when the user is not authorized. Lack of an 
exception means that a user is authorized. Using exceptions this way makes code 
harder to use and harder to test. Using some other data structure with a boolean value 
indicating if the user is authorized would be a better design. Also, arguably the default case
for any authorization function should be to deny access, and access is the "exception."

Finally, and unrelated to the authorization piece, this code is a custom implementation 
of something that is actually quite common for archives to do, syncing metadata. 
There's an established standard for doing this:  [OAI-PMH](https://www.openarchives.org/pmh/). 
One problem we encounter at work when using open-source software with our tech stack is that
almost all of our databases are in SQL Server and this not commonly supported. However 
in this case, we **already have a C# implementation of OAI-PMH that we use for another 
SQL Server backed metadata store**. If this project had gone through design review, 
would it have been implemented at all?

# Bad programmers or bad programming? #

I keep circling back to the issue of time, and the perceptions of engineers of what will
take too much time and what their time is worth. The person who wrote this code was a 
well-regarded, senior engineer, but the evidence suggests he did not follow process. Perhaps if I
had started by debugging, the process I feared because it was time-consuming, this would
have been less of a bad bug and not taken up half my day. Or if I, or someone else on my 
team, had taken the time to set up a better gates for merging deploying code for this project
maybe this bug would never have happened.


[^footnote]: This project appears to not have been developed since 2012. Also a bad sign.
