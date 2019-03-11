---
title: TCOG Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell

toc_footers:

includes:

search: true
---

# Introduction

[TCOG](http://tcog.news.com.au/) is a transformation tier for consuming various content sources and delivering templates. In a nutshell, TCOG is responsible for fetching templates and content, combining them together and sending the end-result back to the client.


# URL Organisation

URLs are split into two types:

- 1 to 1 Mappings with an existing API
- TCOG components 

## 1 to 1 Mappings URL Structure

"1 to 1" Mappings mimick the URLs of the API they are modelled after, except they are preceded by a URI part indicating who owns the URL. Currently there are 1 to 1 Mappings to the Content and Config API.

For example `http://tcog.news.com.au/news/content/v1/?t_product=tcog` mirrors the call to `http://cdn.newsapi.com.au/content/v1/?api_key=XYZ123`

This is a foresight for when we need TCOG to sit before other APIs.

TCOG passes on every URI parameter, except those prefixed with `t_` (eg: t_product), to the downstream endpoint.

### Product Identifiers

TCOG URLs use a `t_product` URI parameter. TCOG internally keeps a configuration key-value store that is used to look up required data for the service TCOG is calling. In the context of using the Content API, `t_product=tcog` will mean that TCOG can look up the Content API and Image API keys configured for the tcog product that are required by the downstream API.

The product also specifies who should be contacted if an error occurs. Notifications are conveyed via [Opsgenie](http://www.opsgenie.com/).


### TCOG Components

Components live under the /component/ namespace in the URI. They also require a `t_product` identifier.

Components are customised endpoints that solve specific problems not met by downstream APIs.

### URI Parameters

TCOG uses any URI parameter that begins with the `t_`, `tc_` and `td_` prefixes. These parameters are removed from the downstream request to the API TCOG fronts. Any other URI parameter will be passed along.

-	`t_` - top level tcog options
-	`tc_` - signifies anything to do with a config
-	`td_` signifies anything to do with display overrides

TCOG has a variety of URI parameters that operate to produce various effects suchs as:

-	toggling on specific parts of templates (whether to show a title, a standfirst, etc.)
-	indicating which template to use (the `t_template` parameter)
-	which css classes or pieces of text to use (all text is sanitized before appearing in a template)

Unless overridden by `t_template` the default template is usually 'extended'.

| Parameter      | Example          | Required 
| -------------- | ---------------- | -------- 
| `t_product`    | `DailyTelegraph` | Yes
| `t_template`   | `s3/something`   | Yes  
| `t_output`     | `json`           | No
| `tc_something` | TBA              | TBA
| `td_timestamp` | TBA              | TBA

<aside class="notice">NOTE: <code>t_product</code> parameters are case sensitive. <code>DailyTelegraph</code> and <code>dailytelgraph</code> will be treated differently.</aside>

# How to create a new TCOG stack

## Create the new configuration

Two configurations are needed: 

- Bamboo cake config
- TCOG config

### Bamboo bake config

Specify this value when baking the environment - in tcogsubdomain.

NetworkSecurityEnv - prod.


### Tcog conf in the codebase.

The environment config file - nca (news.com.au).

Create conf/products/nca.json

Tcog discovers the new conf file to read from via the $env environment variable from shell (see end of cfn/files/core/app_update.sh).

## Create the new SQS queue.

Go to http://bamboo.news.com.au/browse/NEWSTECH-TCOGSQSDEPLOY .

Run a customized build. 

NetworkSecurityEnv - prod

tcogsubdomain - nca

The new build number & tcogsubdomain will indicate the SQS queue to use in the next step.

This will create an SQS queue like 'https://ap-southeast-2.queue.amazonaws.com/877800914193/tcog-nca-22-sqs'.

Once you have created the SQS queue ensure the CAPI team have updated Percolate
on their side to publish into the queue.

## Create the new env file for the new stack

Create conf/env/nca.json

Look at an existing env file for another product. Copy the contents into the new file.

Usually, the only values to change are the redis host and the sqs queue.

## Create the Cloud Formation Redis config for the new stack

In `cnf/redis-redis.json` add a an entry for the new stack to `Mappings/ResourceSize/CacheInstance`:

e.g. 

```
        "apps" : "cache.m3.xlarge",
        "nca" : "cache.r3.xlarge"

```

## Bake a new Redis

This process is basically the same as what is described in `How to Release and Deploy Tcog Infrastructure'. Provided the configuration has been 
created correctly, as described above, you will be create a new Redis instance on Bamboo - http://bamboo.news.com.au/browse/NEWSTECH-REDI

## Bake a new tcog stack 

As above, this is described in 'How to Release and Deploy Tcog Applications.'

## Ensure the Route 53 mapping exists for the new product, in Akamai.

Route 53 has already been configured for product related CNAMEs ahead of time. If you are creating a new one, make it point to tcog core prod.

Next, you will need to cooridinate updating Akamai to map the new stack (origin host), in this case 'nca', to the correct CNAME - nca.tcog.com.au.
 
These will have to be performed by ops.

See PerthNow or The Australian Akamai config as an example (wp.mastheads.prod is the Akamai config for tcog sites).

Change origins from tcog.new.com.au to <env>.tcog.news.com.au. This way the only change you'll have to perform to switchover is at the Route 53 level.

## Update the tcog Akamai config

The Akamai config is called 'tcog'. 

The Akamai tcog config is responsible for routing from an Akamai URL (e.g. a.tcog.new.com.au/etc?t_product=foo) to an internal URL that maps directly to a tcog call (e.g. tcog.news.com.au/component/etc?t_product=foo)
otherwise known as the Origin URL.

The 'a.tcog.news.com.au' CNAME layer is there to allow for indirection. 

## Nginx conf check

A further, underlying detail, is to ensure that nginx also knows about any new CNAMEs. As above, many CNAMEs for products have already been pre-configured but
be aware that new CNAMEs will also need to be mapped in an nginx.conf.

## Now map Route53 for the new Redis

Go to Route 53 in AWS.

You need to create a new CNAME for redis in the tcog proper domain section. A cname has already been created, automatically, by baking the new Redis instance - e.g. nca-1.redis-redis.builds.tcog.cp1.news.com.au .

The new CNAME that you create will look like:

nca.redis-redis.tcog.cp1.news.com.au and it will have a TTL of 60 seconds and a Value of the corresponding CNAME for the new redis generated via bamboo.

## Validate the new environment
Look for the newly created ASG that has been produced via baking the new tcog image.

If it's a brand new CNAME that has not been in production before you can switch Route53 to point to the new stack. 




# Git practices for TCOG

The below is an overview of how we use git (Stash) to take on new work and share it with the team.

Make a Pull Request for everything - features, docs, etc. - except incrementing a TCOG version in package.json. That's done on master.

At times we have broken this rule to accomodate an emergency. We generally regret doing so.

Although at times we may deploy feature branchs to non-prod TCOG stacks to support the business, the master branch is what we use for production.

## Naming your Branch

Use a branch prefix that maps to the purpose of your branch. 

`feature/add-the-livefyre-API`
`bugfix/replace-brightcove-with-video-integrator`
`tweak/uat-uses-c3s`
`new-stack/tcog-prd3`

We don't have strict rules about what makes sense as a prefix, sometimes new ones emerge, but the four above should get you a fair way.

## Feature Branches and Rebasing

We use a 'feature branch' strategy for git. It means you can work on a particular branch in isolation however you like, using rebases to keep in sync with changes on the master branch. 

Finally, when you are happy with your work, you should run a final interactive rebase (see below), which allows all of your changes to appear as one git commit change with a meaningful message for the git log. Only bringing one commit into master keeps it simple for us to track changes.

An example git log:

```
commit e0d5ec37f1170b787bbbcd7b4e4c77435da6f816
Author: Nicholas Faiz <nicholasf@mngl1002224.news.newslimited.local>
Date:   Tue Feb 28 09:50:24 2017 +1100

    v4.18.0
    
    * Remove counting stream from logs and health check
    * Worker lib no longer requires analytics.
    * Remove logstash from the sqs listener
    * Remove analytics from, invalidator
    * Replace logging in agent from logstash analytics to bunyan logger. Only log errors
    * Refactor analytics out of deprecate-params
    * Remove analytics/bench from template_loader
    * Removed analytics
    * Remove opsgenie
    * Remove analytics redis from config
    * Redis connection timeout set to 20 seconds, tcp-keepalive 300 (as the default value for 3.2.4)

```

The above was a large codebase refactor, so it touched on many points. Shorter descriptions of work are okay too.

## Example flow of a Feature Branch

`git checkout -b feature/vidora-integration`

You make any number of commits on your branch. A few "WIP" commit messages, etc..

At some stage you want to bring in changes from master.

Run `git log` and count `n` the number of git commits you've made.

### Squashing up via rebase

Squashing up means you will combine a number of commits into one commit. You use this to bring together all your commit messages.

Let's say `n`=5.

`git rebase -i HEAD~5` 

You will see a list of git commits. For each one except the very top specify `s`.

For example:

```
pick c483b286 tweak: update video documentation for capi v2
s 40a1b7d0 The Vidora prototype.
```

Where I am 'squashing up" from 40a1b7d0 into c483b286. Another way of saying it is ensure that the top line is left with pick but the other lines begin with an s.

Save this and then decide how to craft your log messages into a single meaningful one.

### Rebasing against master

Then run `git rebase master`.

If you open the git log and look, you will see your commit is at the top of the tree, with the latest changes from master beneath it.

If you have run into conflicts you will need to resolve them then `git rebase --continue`. At any stage you can `git rebase --abort` if you are unhappy with how a conflict was resolved.

By following this process you can continually work on your branch and hone it, rebasing against master to get its lastest work.

### Ready for a PR

When you are ready to submit a PR run a `git push` and follow the instructions Stash will give you.

```
♪  tcog git:(docs/how-to-stash) ✗ git push
fatal: The current branch docs/how-to-stash has no upstream branch.
To push the current branch and set the remote as upstream, use

    git push --set-upstream origin docs/how-to-stash
```

After running `git push --set-upstream origin docs/how-to-stash` Stash responds with a URL that will let me create a PR.

```
remote: Create pull request for docs/how-to-stash:
remote:   http://stash.news.com.au/projects/TCOG/repos/tcog/compare/commits?sourceBranch=refs/heads/docs/how-to-stash 
```

## Pull Requests

Pull Requests are a well documented convention in the programming world these days. Some internal conventions we use are:

* be time effective, if the reviewer suggests several small tweaks but is otherwise happy with the code, then it's good to merge as soon as the small tweaks are done (no need for another round of formal review).
* the owner fo the PR is responsible for keeping the branch up to date (rebased) against master (so it's also good for the reviewer to be efficient about reacting to new PRs).

## General Dos and Donts

* Do use PRs as a place to discuss best practices and ideas. PRs mean at least two devs have agreed with new code and open up a space of conversation.
* Don't form new feature branchs of other feature branches. Sometimes this creates a chain of dependencies that haven't been agreed to by another team member. It's always best to branch off master.


# Akamai

All users should be user the Akamai endpoint of tcog. An Akamai endpoint is denoted by `a.` in the hostname eg `a.tcog.news.com.au`. The only time the origin host is used is between Akamai and tcog.

ESI processing of fragments is turned off by defualt. It has to be explicitly opted into by appending the `esi=true` param.

eg:

```
origin : http://tcog.news.com.au/component/article/dailytelegraph/desktop/newslocal/inner-west/westconnex-richard-moras-fed-up-with-workers-wrongly-identifying-his-st-peters-house-to-be-demolished/news-story/3e92c0fd276f5108fd353694e42f0585?t_product=DailyTelegraph&t_template=s3/chronicle-templaterouter/index
akamai : http://a.tcog.news.com.au/component/article/dailytelegraph/desktop/newslocal/inner-west/westconnex-richard-moras-fed-up-with-workers-wrongly-identifying-his-st-peters-house-to-be-demolished/news-story/3e92c0fd276f5108fd353694e42f0585?t_product=DailyTelegraph&t_template=s3/chronicle-templaterouter/index
esi    : http://a.tcog.news.com.au/component/article/dailytelegraph/desktop/newslocal/inner-west/westconnex-richard-moras-fed-up-with-workers-wrongly-identifying-his-st-peters-house-to-be-demolished/news-story/3e92c0fd276f5108fd353694e42f0585?t_product=DailyTelegraph&t_template=s3/chronicle-templaterouter/index&esi=true
```

### Integrations ( routing )

Sites can integrate with tcog in one of two ways, as above with tcog fragments or via an origin mapping. An origin mapping is where a site offloads the responsibility for processing a request to an external service.

Within our network we have origin mappings for metros, regionals and nationals which direct traffic for articles and galleries to tcog.

For a site to integrate in this way the following information is required by tcog in-order for tcog to permit access.

- **x-tcog-template** : the remote template to user
- **x-tcog-product** : the tcog product name

<aside class="notice">These request headers are the equivalent of the <code>t_product</code> & <code>t_template</code> query parameters.</aside>


In addition to this, the tcog host is required. This may be different depending upon the environment you are integrating with and an origin base path. The origin base path relates to the default path to the origin host eg: "/component/article".

Rules are the applied to determine when this routing should occur. For example

- /news-story/* : if the requests if of type article
- /image-gallery/* : if the requests if of type gallery
- /story-*-* : legacy article urls
- /(photos|gallery)-*-* : legacy gallery urls

**Examples**

```
- http://www.news.com.au/world/north-america/trump-to-oreilly-i-dont-know-if-obama-will-admit-this/news-story/a5d640220809d9866309d3054c5cfacf
- http://www.dailytelegraph.com.au/entertainment/inside-starstudded-super-bowl-2017/image-gallery/8ffdcd0e34986a70939649bc13d7eea5
- https://www.couriermail.com.au/business/companies/fortescue-metals-restructures-rosters-for-pilbara-mine-workers/news-story/fb59d53e19b0abc3152716cc86acfa32
```
Akamai will forward request in all cases to tcog as follows.

`tcog.news.com.au/component/article/<request-path>?<request-querystring>`

<aside class="notice">It's worth noting that image gallery uses the same endpoint as article. For example <code>t_product</code> & <code>t_template</code> query parameters.</aside>


### Debugging

Akamai provides a debug console for testing urls which are Akamai enabled and may also contain nested fragments.

Often you may need to help teams debug why a tcog url is not working when it is ESI'd into a page.

1. Visit https://fed3.news.net.au/adfs/ls/idpinitiatedsignon.aspx
2. Select "Akamai Luna Portal" from drop down
3. Visit https://newscomau.luna-sp.com/portal/esid_2.jsp

The debug portal has the following options

- **URL to Debug** : the url you need to test ( www.news.com.au/... or a.tcog.news.com.au )

Most environments are configured to work with Akamai.

- **Client Request Headers** : additional headers needed to access the page

The following additional headers are required in-order to correctly debug pages on our network.

```
ndmesidebug:654321
Cookie: nk=anything; n_regis=anything; n_rme=anything; open_token=anonymous; sr=true
```

ndmesidebug is need to bypass IP restrictions that some urls may have in place and the Cookie header is needed to force a paywall bypass.

ESI Developer Guide : https://www.akamai.com/us/en/multimedia/documents/technical-publication/akamai-esi-developers-guide-technical-publication.pdf

#### Debug Headers Extension

This extension makes it possible to inpect information Akamai knows about a request it received such as how it has been cached, for how long and which ghost server processed it.

See video/akamai-headers.mp4 for more detail.

https://chrome.google.com/webstore/detail/akamai-debug-headers/lcfphdldglgaodelggpckakfficpeefj?hl=en

#### Debugging config changes

An Akamai config can be deployed to two networks. A staging network where you can conduct testing prior to a production deployment and the production network.

Depending upon the change made you may be able to verify using the Debug Headers Extension alone. Should this not be possible you may have to resort to using curl.

In either scenario you will need to ensure that the host you are attempting to connect to is correctly routed via the staging network.

```
curl -c cookiejar -L --resolve <host>:<staging-network-ip> <url-to-test>
```
> testing via curl

```
curl -c cookiejar -L --resolve a.tcog.news.com.au:80:23.50.63.24 http://a.tcog.news.com.au/healthcheck\?t_product\=newscomau
curl -c cookiejar -L --resolve www.news.com.au:80:23.50.63.24 http://www.news.com.au/world/north-america/donald-trump-to-order-temporary-ban-refugees/news-story/9423d195c14da764324f16929281f9a1
```
> curl examples

#### Advanced ( experimental )

An advanced debug capability is available using a local dockerised version of Akamais testing server ETS.

See : http://stash.news.com.au/projects/TCOG/repos/akamai-ets/browse


