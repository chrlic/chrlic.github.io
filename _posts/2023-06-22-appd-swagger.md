---
title: Setting up business transactions based on API definition
---

# Setting up business transactions based on API definition #

If you have been working with AppDynamics for some time, you would probably agree, that the concept of business transactions helps a lot in both getting business relevant data about application performance and stability and in aiding in problem isolation process.  

Sometimes, default settings for business transaction detection work well, but more often, fine tuning is required to get relevant results. One of the painful tasks is to cover REST API’s well. Why? The default detection rule in this case identifies business transactions based on the first two segments of the URI path. Nine times out of ten, this is not an optimal setting. API’s often follow URI path pattern like `/v1/rest/customer/<cid>/order/<oid>`. Problems immediately appear: 

* By default, we get then one business transaction for all REST API calls - `/v1/rest`.  

* Expanding to the first three segments does not solve the problem if we want to differentiate between customer orders and invoices, for example. 

* Expanding further out to the first five segments does not help either – here we have a problem with API cardinality, since `<cid>` takes different values and pollutes the business transaction namespace. And it is scarce – 50 business transactions per agent, 200 per application by default. 

* Next idea is to take the third and the fifth segment of the URI. But there are likely other API patterns in the same API, where this does not work either. 

In the end, in projects I was involved, the solution was to create transaction detection rules regex-based with patterns like this: 

Business transaction name - `/v1/rest/customer/{cid}/order/{oid}`

Matching Regex - `/v1/rest/customer/.*/order/.*` or `/v1/rest/customer/[^/]+/order/[^/]+` 

Setting this up for an API with couple dozen endpoints is not fun. In one project I was involved in, with many applications providing REST APIs of different patterns, I finally decided to create a tool which automates this to a substantial extent. Since many applications providing REST API are built on frameworks which automatically provide API documentation in Swagger format, the tool takes the file with the API definitions and converts them into transaction detection rules, saving hours and hours of manual work. 

If you happen to be in an analogous situation, you might find the tool handy, and it is now released as an open-source project at [github.com/cisco-open/swagger-appd-tool](<https://github.com/cisco-open/swagger-appd-tool>) with explanation how to use it as a GUI-based tool and how to potentially automate updates as part of your CI/CD pipeline.  

One closing note – keep in mind the limits on the number of business transactions still apply. Either pass only the important API endpoints in the definition file to the tool or clean-up business transactions afterwards in the controller GUI.  

Len me know, what you think! 