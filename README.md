<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  

- [Explainer - Partition the HTTP Cache](#explainer---partition-the-http-cache)
  - [Introduction](#introduction)
  - [Goals](#goals)
  - [Choosing the Partitioning Key](#choosing-the-partitioning-key)
    - [Partition using top frame site: Double keying](#partition-using-top-frame-site-double-keying)
    - [Partition using top frame and frame site: Triple keying](#partition-using-top-frame-and-frame-site-triple-keying)
  - [Proposed solution](#proposed-solution)
    - [Use top-frame and subframe as keys (Triple keying)](#use-top-frame-and-subframe-as-keys-triple-keying)
    - [Define same-site as scheme://eTLD+1 for the initial launch](#define-same-site-as-schemeetld1-for-the-initial-launch)
  - [Impact on metrics](#impact-on-metrics)
    - [Network traffic](#network-traffic)
    - [Page performance](#page-performance)
    - [Cache](#cache)
  - [Impact on APIs](#impact-on-apis)
    - [Fetch API](#fetch-api)
  - [Alternative solutions considered](#alternative-solutions-considered)
    - [Partition using frame only](#partition-using-frame-only)
    - [Partition using the chain of frames](#partition-using-the-chain-of-frames)
  - [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
  - [Acknowledgements](#acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Explainer - Partition the HTTP Cache


## Introduction

Chrome’s HTTP cache is currently shared across all sites, with a single namespace for all resources regardless of which site the resource is fetched from. This opens the browser to a side-channel attack where one site can detect if another site has loaded a resource by checking if it’s in the cache. Such exploits have been demonstrated in the wild. 

Here, we propose to partition the HTTP cache to prevent documents from one site from knowing if a resource from a cross-origin document load was cached or not. The exact key used to partition on is described later in the explainer. Firefox has also published an intent to implement to partition their cache, and Safari has partitioned their cache for several years now.

Such partitioning limits the reusability of third-party resources. Each site will need to load those third party resources (such as fonts or scripts) for themselves at least once. This increases network usage and may ultimately degrade page load performance. Chrome’s preliminary experiments with partitioning show that the overall cache miss rate increases by around 2% (more details below) but changes to first/largest contentful paint aren’t statistically significant. This may change as we flesh out the implementation and progress to larger populations but it’s an encouraging start.


## Goals

The goal is to isolate sites such that one site can't communicate with another via the cache.

Doing so prevents cache attacks such as the following:



*   **Cross-site search attack:** There exist cross site search attack proofs-of-concept which exploit the fact that some popular sites load a specific image when a search result is empty. By opening a tab and performing a search and then checking for that image in the cache, an adversary can detect if an arbitrary string is in the user’s search results. 

*   **Detect if a user has visited a specific site:** If the cached resource is specific to a particular site or to a particular cohort of sites, an adversary can detect user’s browsing history by checking if the cache has that resource.

*   **Tracking:** In addition to the above cache attacks, the cache can also be used to store cross-site super-cookies as a tracking mechanism. To clear them the user has to delete their entire cache (not just a particular site). Since such this is neither transparent nor under the user’s control, it results in tracking that doesn’t respect user choice.

## Choosing the Partitioning Key

This section details the pros and cons of the various possible partitioning keys.



### Partition using top frame site: Double keying

Each frame’s cache is shared with all other frames on the page and with other pages with the same top-frame site.

**Benefits**



*   **Isolation between cross-site pages:** Two top-frame documents whish are not same-site will not share the cache, and therefore will not be able to determine if the other loaded a given resource.
*   **Precedence in other browsers:** Partitioning using top frame eTLD+1 has been implemented in Safari for over five years now.

**Challenges/Limitations**

This solution leads to the cache being shared across all of the subframes in a page. Thus it assumes that the top-level publisher trusts the frames that it embeds and other embedded frames trust the top level frame to only embed frames from other trustworthy publishers.

**Performance impact**

As mentioned in the metrics details below, preliminary experiments with partitioning show that the overall cache miss rate increases by about 2 percentage points but changes to first contentful paint aren’t statistically significant and the overall fraction of bytes loaded from the network only increase by around 1.5 percentage points.


### Partition using top frame and frame site: Triple keying

Each frame’s cache is only shared with same-site frames on documents from the same top-level domain.

This solution uses both top-frame and frame domains as the cache partitioning key. 

**Benefits**



*   **Isolation between cross-site pages**
*   **Isolation between cross-site frames on a page**
Double keying will solve the cross-site search and similar security attacks between top-level pages but not between frames, which can happen if:
    *  A popular site embeds a malicious cross-site iframe.
    *  A malicious top-level site embeds a popular site as an iframe. This will require that the popular site does not have a framebusting defense. The fact that defense against framing is an opt-in security feature, suggests there might be some sites with user sensitive data that could be protected against such attacks using triple keying.
In general, using double keying does not prevent leaking information across cross-site frames.


**Challenges/Limitations**

Performance. 

**Performance impact**

Results for core metrics like first contentful paint, percentage of bytes served from the network and cache misses are the same as with double keying (mentioned in the above section). 

## Proposed solution
### Use top-frame and subframe as keys (Triple keying)

As detailed in the metrics below there isn't a big performance difference between double and triple keying. Since the latter provides the added security benefit between cross-site frames, Chrome plans to use both in their partitioning key.

### Define same-site as scheme://eTLD+1 for the initial launch
It is likely for frames on a page to belong to the same site if not the same origin and we would like to continue giving those frames the performance benefits of caching. For this reason, we plan to go with scheme://eTLD+1 instead of origin for the initial launch. In the long term, since dependency on Publix Suffix List is not ideal, the plan is to migrate to other more sustainable mechanisms like First Party Sets or use origin with an opt-out mechanism so that frames can opt-out from triple keying to double keying, if there is a need.

## Impact on metrics

This section goes into the details of metrics for both of the partitioning approaches mentioned above:


### Network traffic

*   Fraction of bytes read from the network:
    *   Control: 65.9%
    *   Double keying: 66.4%
    *   Triple keying: 66.4%


### Page performance



*   Navigation start to first or largest contentful paint:
    *   No statistically significant change for both partitioning approaches.
*   Browser jankiness:
    *   No statistically significant change for both partitioning approaches.
*   Interactive timing delay for input processing:
    *   No statistically significant change for both partitioning approaches.


### Cache

This section gives the increase in cache miss rates overall for all types of resources.

It also gives the metric for specific types of resources like 3rd party fonts, css and js files. The 3rd party metrics is a good measure to see the impact on CDNs. Please note that at this time the 3rd party resources metrics are fairly new and we will watch how these numbers change over the next few weeks and update them here.



*   Total cache miss rates
    *   Control: 52.7%
    *   Double keying: 54.4%
    *   Triple keying: 54.7%
*   Cache miss rates for 3rd party fonts
    *   Control: 30.9%
    *   Double keying: 40.3%
    *   Triple keying: 42%
*   Cache miss rates for 3rd party javascript files:
    *   Control: 29.5%
    *   Double keying: 35.2%
    *   Triple keying: 37.15%
*   Cache miss rates for 3rd party css files:
    *   Control: 21%
    *   Double keying: 24.8%
    *   Triple keying 25.6%

## Impact on APIs


### Fetch API

The fetch API has an 'only-if-cached' mechanism that allows sites to observe if same-origin resources are in the http cache. The partitioned HTTP cache will have some cases where different caching behavior will be observable via the Fetch API.  For example, consider:



*   Top level frame A with origin foo.com
*   3P iframe B with origin foo.com in a different cross-origin tab 
*   Iframe B does a normal fetch(url) to populate the http cache
*   Iframe B uses BroadcastChannel to signal to window A
*   Window A uses fetch(url, { cache: 'only-if-cached' }), but does not see the cached resource as expected since the cache is partitioned by the top-frame-origin.


## Alternative solutions considered


### Partition using frame only

Each frame’s cache is shared with all other same-site frames, regardless of the top-level page they’re on.

This solution only uses the request’s frame as the partitioning key. 

**Benefits**



*   **Isolation between cross-site frames:** This solution isolates two different cross-site frames from their cache accesses.

**Challenges/Limitations**

Tracking the user across different top-level sites will be possible by storing persistent user identifiers via third party frames. 


### Partition using the chain of frames

Each frame’s cache is shared with other same-site frames, only if the chain of frames that nest those frames is the same.

This solution uses the request’s chain of frames as the partitioning key. 

**Benefits**



*   Clear isolation between two different chains of frames

**Challenges/Limitations**

Performance. We do not have any data for this approach as of now.


## Stakeholder Feedback / Opposition



*   Firefox: Public support, have sent an [intent to implement](https://groups.google.com/forum/#!msg/mozilla.dev.platform/eFx-93iBPpU/4HQ2UFN9DgAJ).
*   Safari: Public support, have an existing [implementation](https://bugs.webkit.org/show_bug.cgi?id=110269).
*   Web developers: This is not a breaking change, but it will have performance considerations for some organizations. For instance those that serve large volumes of highly cacheable resources across many sites (e.g., fonts and popular scripts). This needs to be balanced with the privacy and security concerns and the fact that sites often use different versions of popular libraries, reducing the benefits of such caching. It’s also worth reiterating that Safari has already partitioned its cache and that Firefox has signaled that it wants to as well.


## Acknowledgements

Many thanks for the valuable feedback from:

Josh Karlin




