---
title: A Stateless Auth System (High Scalability)  
published: true
tags: HLD System-Design Authentication Authorization High-Scalability
---

<p align="center">
  <img height="400px" src="{{site.baseurl}}/assets/images/access-denied.gif">
</p>

Any system is incomplete without an Auth subsystem. It is one of the most core and critical components of all since it acts as an entrypoint for all requests made to your system.  
But before we dive into the design that I am going to discuss in a while, let's first set some context around so that we are all on the same page.  

## Authentication and Authorization

These 2 funny sounding terms are highly confusing and usually used interchangeably when not understood correctly.  

**Authentication**  
It deals with identifying the user who is making a request to your system. If the system is not able to identify the user from the request, then it usually returns an error code implying there is something wrong with the request. We have a status code in HTTP specifically for this case - **401**.  
A status code 401 means there is something wrong with the token sent because of which the system isn't able to correctly identify the user from it.  


**Authorisation**  
It's usually the second step after the **authentication** succeeds. After we've identified the user, we try to figure out whether that user is allowed to access the particular resource in the request. For example, the user could be trying to read/access a specific post he/she wrote a couple of months ago. So the job of the **authorisation** subsystem is to identify if the user has the permission to access a particular resource/operation/action requested in the call. If not HTTP sends out a **403** status code implying that the user was identified but he/she does not have the right permission to access that resource or call the operation.  


## A Session Token Approach

<p align="center">
  <img height="500px" src="{{site.baseurl}}/assets/images/session-store.png">
</p>

This probably is the easiest to implement and hard to maintain at scale.
After a user submits their credentials, the system issues a unique random string which we call a `session_token`. The system remembers this session token and knows who created it, so whenever a user makes a request with this specific session token, the system can immediately tell if this session token is valid or not and look up the user against it to check if they have the right permissions.

<table>
<colgroup>
<col width="50%" />
<col width="50%" />
</colgroup>
<thead>
<tr class="header">
<th>session_token</th>
<th>user</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">54*hgjhgjh^G</td>
<td markdown="span">goku</td>
</tr>
<tr>
<td markdown="span">45y6^v!%rv#$</td>
<td markdown="span">vegeta
</td>
</tr>
</tbody>
</table>

<table>
<colgroup>
<col width="50%" />
<col width="50%" />
</colgroup>
<thead>
<tr class="header">
<th>user</th>
<th>permissions</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">goku</td>
<td markdown="span">canSSJ1, canSSJ2, canSSJ3</td>
</tr>
<tr>
<td markdown="span">vegeta</td>
<td markdown="span">canSSJ1, canSSJ2</td>
</tr>
</tbody>
</table>

The only problem with this approach at scale is maintaining the state on the servers. Assuming we have a ton of traffic, we'll have to store all this information in a common storage system and this storage system would have to be scaled accordingly (based on the reads and writes this system expects). Also, this storage now becomes a single point of failure if not replicated/backed up frequently.  

<p align="center">
  <img height="500px" src="{{site.baseurl}}/assets/images/session-store-load.png">
</p>

You could argue that we can scale our storage by using the right amount of replication and sharding to cater to our reads and writes, and TBH a lot of the tech giants do do that, but maybe there's a better way.  

## A Stateless Approach

Another way to solve the problem is for the session token to be self-sufficient and authentication to go stateless. What's different about this approach is that on each login the server issues a signed document (JWT) instead of a session id which contains all the necessary information to identify the user and their access level. 

On each subsequent request, user presents this JWT to the server and now we don't have to do any database lookup as all the necessary information is already there in this signed document. This second approach is highly scalable as we have completely gotten rid of the server side session datastore.

<p align="center">
  <img height="400px" src="{{site.baseurl}}/assets/images/jwt.png">
</p>

The JWT approach may look better than sessions but both have their pros and cons which we'll come to in a while. Choosing between sessions and JWT is more of making a trade off between their offerings. Popular websites like Google, Facebook, Amazon still use sessions in their SPA. They might have their own reasons for doing so. Please refer Pitfalls section of this article to learn more.

## Using the Right Encryption Scheme

When using JWTs we also need to decide what kind of encryption scheme are we going to use for our tokens.  
There are broadly 2 types of them:  

- **Symmetric Encryption**  
  A single secret key is used to hash and verify signature. Example: HMAC + SHA256  

- **Asymmetric Encryption**  
  A public-private key pair is used in this case where the private key is used to create hash signature and public key is used to verify the hash. Example: RSA + SHA256

With a symmetric encryption scheme we have a single secret which is responsible for generating hash and verifying the same whereas with asymmetric scheme we can verify the signature using a public key. Asymmetric encryption is a great fit for us because the public key is "public" so anyone can verify whether the signature is valid or not.  

This will help us identify if the token has been forged or not, if the verification succeeds we know that all the payload is valid.  

If the signature verification fails then we usually send out an HTTP code of `401` to indicate that the **Authentication** has failed.  

## Stateless Authorization

So far so good, we have been able to generate JWT tokens and verify them without any state. For the next stage we need to be able to tell whether the operation called is allowed by that user or not.  
Traditionally, permissions are modeled as user defined strings. Additionally, groups allow for aggregation of permissions. In the usual flow, on requesting a resource with a user token, the resource server would at runtime retrieve the permissions associated with the user and match against the permission it expects the user to have.

To exemplify using our Dragon Ball example:
* We have a resource `/evolve_to_ssj3/` and calling `GET` operation (HTTP) on this resource turns our Saiyan into Super Saiyan 3  
* We create a permission against this resource called `can_ssj_3`  
* This permission is associated with our saiyans (users)  
* On requesting GET on `/evolve_to_ssj3/`, our system retrieves and then checks if the user specified has the permission `can_ssj_3`  
* If False it raises a forbidden(403) error else proceeds to process the request  

This approach is problematic as on each request we are fetching the permissions from a stored database which becomes a different complexity to deal with.  
We instead propose the usage of scopes.

```
{
  "iss": "DragonBallInc",
  "user_id": "Goku",
  "iat": 1558434624,
  "exp": 1558441824,
  "scope": "can_ssj_3 can_ssj_1 can_ssj_2"
}
```

Now our system can just read the payload and see if our user has a specific permission in scope array only then will that operation be allowed otherwise it throws a forbidden (403).  

## Making the permission model more powerful

Assuming we have a microservice model where we have divided our workload into small little services.  
We will be able to identify whether the incoming token is valid or not at the API gateway (the first point of contact) itself but in order to take care of the `Authorisation`, we would have to send the request to the individual microservice which is requested.  
This works but its better if we take care of `Authorisation` at the high level (API Gateway) itself so that we send only valid requests to the individual microservices. 
Plus our individual microservice can focus just on business logic without worrying about user management.  

The big idea is to make use of `Regular Expressions` as permissions for our system.  

## How does RegEx help us?

According to REST principles we can treat everything as a resource, and our HTTP methods (GET, PUT, POST, DELETE, PATCH) can act as operations on these resources.  
To understand better, we can look at existing permission as a combination of 3 things:  
* Microservice Application
* Resource (or HTTP request path /evolve_to_ssj3/ in case of our system)
* CRUD Actions ( POST, GET, PUT, DELETE & PATCH)

So our custom authorizer at the API gateway can read all these 3 things from any incoming request and try match that against all the regex present in the token scope to identify whether an incoming request has the required permission to call that specific action on that resource or not.  
Refer this table to get an idea around how you could write a permission using regex:  

<table>
<colgroup>
<col width="50%" />
<col width="50%" />
</colgroup>
<thead>
<tr class="header">
<th>Permission RegEx</th>
<th>Meaning</th>
</tr>
</thead>
<tbody>
<tr>
<td markdown="span">^\/products:GET$</td>
<td markdown="span">Can get products</td>
</tr>
<tr>
<td markdown="span">^\/products:DELETE$</td>
<td markdown="span">Can delete a product (something that only admin would have)</td>
</tr>
</tbody>
</table>

## Performance

This stateless system is extremely performant since the computing unit already has all the required information to perform `Authentication` and `Authorization`.  
There are no I/O calls to get extra information, everything is part of the request itself.  
Just by replicating the custom authorizer logic to multiple machines you can easily scale the throughput to millions of requests per second.  

## Pitfalls

There are some known limitations to our approach here. Some of these are highlighted below:
* No Logout: In a session-based authentication, each session is stored on the server-side and validated against every request. However, OAuth JWTs are stateless and self-sufficient. Each JWT contains an encoded expiry date and a signature. Thus without implementing a blacklisting/whitelisting database, we can not enforce invalidation of a live session. On logout, we revoke the refresh token so that no more access tokens can get generated. So if access tokens are short-lived then logout shouldn't really be a problem.
* OAuth JWT token state is eventually consistent (depending on how frequent clients refresh the tokens) as opposed to session tokens which are strongly consistent.
* JWT can get bloated with permissions, and we would end up utilising a lot more bandwidth compared to our session token model