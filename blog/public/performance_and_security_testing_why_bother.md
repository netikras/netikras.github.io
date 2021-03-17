# Performance and security testing. Why bother?

## Anyone can write an application
Application development is a process during which a product is created or enhanced feature-wise. Usually, clients are so locked on the goal to "have an application that does THIS and THAT", that they forget the application needs a lot more than just have features. Or even worse - they know that and assume it's a part of coding!
What does an application require besides code?

- Infrastructure. Where will it run? Cloud? On-prem? How expensive will that be? Who will maintain it all?
- Stability. What's the estimated app outage per year? How likely is it to break out of the blue? How likely is it to break when userbase peaks? Who will recover the application? How fast? Will there be a DR?
- Security. Will the application be capable to enforce access control? Will there be ways to bypass that control? Will developers put security bugs in the code? Or will developers perhaps be unaware of security issues in (non)commercial tools they will have to use for the app? Or perhaps noone will be aware of such bugs until the app is live and exploited, and your developers will be the ones who fix them for the tools' vendors? Or maybe security will be flawed not in the code, but in the infra? Or at the infra provider level? Or at any other point..?
- Performance. How many requests will the application be able to handle? How many resources does the app require to GO-LIVE and be capable to handle all those requests? How will the app performance degrade over time? Who will notice it besides the end-users? Will there be proactive performance monitoring, performance control, so that OPS can take actions before end-users notice performance degradations?
- ...

There are a few more aspects to consider besides just "write the damn code". Anyone can write the code. Even the client itself could write the app if he/she spent a few months learning the language. Does that mean the end result is good for the go-live? Would you release it as soon as you think it works? No? Why not?

## Hire the professionals
Yes, the answer to any business needs - just pay someone who knows how to do it. They are professionals - they know how to do it! "I am paying you, so you make sure you do it well!"
- make it fast
- make it secure
- make it pretty
- make it have 4.17 gazillion of features
- make it maintainable
- make it stable
- make it without errors
- make it... flawless
- make it cheap
- ...
 
The list of requirements continues. And that is expected. The business is non-technical people (usually), they know they want the product to be of high quality and feature-rich. And they are paying for it for the professionals - the people who claim to be good enough to deserve to be paid for the job. It's only natural to expect the best results!

## The pickle
Yeah, there are a few problems with the "hire the professionals" approach.

1. No matter how good the developer is, you still require a QA. A developer **NEVER** creates bugs intentionally. No matter how good the developer is, app development is but a mental labour. The developer might be tired, moody, feeling sick, stressed, etc. FWIW, a developer might make a (code) design decision which ends up to be suboptimal after a few new features. And fixing the code design usually ends up in bugs. Unit tests do help here a lot, but they are far from enough.
2. Say, you got the app created. Say, you've released it to LIVE. What's next? Will it just sit there doing nothing? Will the world stop evolving from tht point on? No.
The application users will find ways to intentionally or unintentionally break the system. It is impossible to predict every possible flaw at the development time. Believe it or not, even fish in the ocean, wind, solar bursts and the very fact that we are on a rock flying in the space around the Sun can be exploited to harm your application. All these factors can harm your application even without user in the equation - it can _just happen_.

What I'm trying to say is that no matter how good the professionals you hire, they will not be able to create and release a perfect product for you. The best you can get is **good enough**. The _closer_ you want to get to that, the more funds and time it will cost you.

## Application is alive
There's a reason why they call it a _go-**LIVE**_. Once released to the public, your application begins living. And living requires constant maintenance: supply of resources, means to clean up the waste and means to protect itself from hostile forces. And all of those can be both physical and virtual. 

Consider a database with a table containing billions of records. How long would it take you to find a user "Andrew1121" in that registry? Why can't you find it any faster? "I just can't, I'm a human, not a machine" is an almost valid answer. Yes, a machine can find it faster for you. But why can't it find it even faster than that? "Because it can't, it's just a machine, not a {...}". Even machines have limits which, once reached, slow them down considerably. There will come a time when your database is too big to be fast. Your app will become more andmore sluggish over time, your app maintenance will grow, expenses will grow and usage (think _end users_) will start to decline because of the slowness. Your app was blazing fast in the beginning and its performance degraded over time. The professionals you hired did the job and made a well performing product, but the _life_ of the application had its way.

Consider a webserver. It's the part of your application through which end users are accessing all the features of your webapp. It's also like the skin of your application. If anyone tried to harm your app, they would most likely to have get past it, probably by finding some weak spot (a wound, mole, a pore, etc.). The skin heavily depends on its surroundings: atmospheric pressure, temperature, humidity, radiation, etc. Your webservers also heavily depend on the environment: temperature, humidity, radiation, DNS servers, routers' firmware stability, vendors (ISP, cloud, hardware, firmware, software,...), physical network fibers, and everything that can affect them. There are so many variables at play that it's only a matter of time before either one of them is successfully exploited to violate the webserver's security.

Once released to the public, application becomes alive and a subject to hostile forces and entropy. So just building, deploying and releasing the application is not enough to keep the service alive. Maintenance is mandatory.

## Performance and Security is not just QA
QA usually is a part of the release pipeline in the development cycle. The developer writes code and build the application, QA tests it to make sure it meets the quality requirements and then the product is shipped. The thing is, QA is usually oriented towards functionalities of the application. And IMO that's how it's supposed to be. Functionalities are static and the core of the application: the only place they can be changed at is the developer's workstation. Once developed/changed, they must be tested to make sure the features are working as expected and don't break the business flows. 

Performance and security are usually considered a part of QA. And that is not right. These factors can ONLY be tested in LIVE environment. Yes, the code can be challenged performance or security wise in non-prod environments too, but the results will definitely definitely be off. Why is that so?

### Security
#### The problem
In PROD we rely on substantially more external forces than in non-prod environments. Different network infrastructure, different servers, locations, different load patterns,.. and the fact, that in PROD we potentially have billions of security testers: both human and automated. Do we have those factors in non-prod? Nope. If our pre-prod security testing gives us green results, can we conclude that our application will be secure in PROD? No way. The best we can conclude is that our non-prod application is protected from the attack vectors we have tested for (i.e. we could think of) and that prod is likely to be protected from the same set of attack vectors. What about the other vectors? The ones we haven't thought of, but a 16yo prodigy living at his parents' basement has? What about attacks on the ISP? Or the IaaS? Or human errors in the LIVE environment? Or vendors' employees deciding to profit by downloading your information they can access illegaly and selling it in the black market? Or your concurrents?

We can test security in pre-prod environments, but don't make a mistake and think that these tests reliably reflect your production environment's security. Testing security in QA phase is better than nothing, as it can catch the *most common* security issues before releasing them to prod. But it doesn't mean other issues (the ones we haven't tested for) won't be shipped to prod and it doesn't mean the issues we've fixed or haven't found in non-prod surroundings won't be exploitable in prod conditions.

#### The solution
It is impossible to make an application 100% secure. The bese you can get is **good enough**, and the _enough_ part is a matter of the SLA. Business and tech people are to agree on security assurance terms and define

- _how often and when/where at the lifecycle app security will be tested_
  both testing in QA phase AND periodic tests in a live PROD environment are strongly adviced
  
- _how extensive will the tests be_
  only automatic, only manual, or hybrid? None of the approaches will assure 100% security, but the more extensive the testing, the more expensive it will be and the close to the _good enough_ app security will be.
  
- _what security flaws are tollerable_
  sometimes business might have to make compromises: lose some users for better security or afford a lower security standard; have higher expenses for better security or save some $ hoping noone finds a way to exploit the problem.
  Sometimes technicians might have to make compromises too: cover a more serious vulnerability with a less serious one, because the serious one is impossible to fix ATM or ever (e.g. Intel Spectre, Rowhammer, NAT Slipstreaming,...), mitigate a vendor or a third-party dependency vulnerability with less vulnerable workarounds, and so on.

There is no actual solution to be secure. You will never be secure. The best you can do is to test for as many attack vectors as you can both, manually and automatically: features, infrastructure, middleware, databases, webservers, caches, network. And prepare for a possible breach: protect your assets, data and prepare a detailed security incident management plan.

### Performance
#### The problem
While some security testing can be automated as a part of QA, performance testing is even more stubborn. 
If code has security flaws, these flaws will be present in all the environments if deployed. If infrastructure has security flaws, then they might not be possible to catch early. 
If either code or infrastructure have performance problems, it's impossible to reliably test them automatically. In performance testing there's another component that *always* plays a part - **under what conditions**. This component is less expressed in security testing, while in performance testing it must be *always* taken into account. Always.

If we are testing performance as part of QA phase - under what conditions? Will we be able to recreate PROD conditions in non-prod environment? Are we willing to spend $ for a copy of the PROD environment to test its performance? Will we be able to generate enough load to mimic PROD load? Will we be able to mimic PROD-like e2e flows?

It's possible, but it's expensive. And difficult. And, more often than not, performance tests show non-conclusive results, because conditions are always dynamic, so we need to differentiate between conclusive and non-conclusive tests.

Performance microtesting is as good as unittesting. It's a good way to catch the most common problems early in the development phase, but it will not account for the system as a whole (multiple components, infrastructure, load patterns,...).

#### The good side

Performance testing results, however, tend to be more sustainable than security testing. While it's impossible to know all the possible variables in security testing, we usually know all the variables in performance testing. These are:

- cpu: frequency, cache sizes, core count
- memory: frequency, capacity, limits
- maintenance tasks: Garbage Collection (duration, frequency), state cleanups (TTL, amount freed, flow changes caused by changed state), ...
- IO: type, frequency of occurrence, added latency, payload size
- storage: capacity, number of sequential reads (and size and latency), number of scattered writes (and size and latency)
- network: call rate, payload size, hops' count, protocol overhead
- limits

And all the possible combinations of all of the above. Usually, applications operate in a closed system, which means we know all the components we are working with and we have control over most (if not all) of them. This provides us with an ability to reliably measure latencies, durations, resources' utilization -- all of them **under certain conditions**. I didn't say it's easy to reliably test performance. I said it's possible.

#### The solution

The only reliable way to test performance in non-prod environment is to:

- have a copy of PROD (no more than a day old caches and databases) in non-prod environment; identical infrastructure (resources, configuration, limits, versions)
- ability to restore the environment state after each test (restore databases, caches)
- have another environment to generate load from
- have test scripts for most of the e2e flows (should be a result of prod access logs and analytics tools' statistics analysis)
- set up several different load patterns, e.g. 100 users adding items to the cart while another 70 are browsing in the store listings; 5'000 users adding items to the cart while 400 are checking their carts out and 7'000 are browsing products listings and there are 2000 new logins being processed; and so
- use as many different items as possible, ar different properties of the items can trigger different speciffic flows, have larger payloads

And yet this kind of test results will be only good enough to claim that the prod application will _most likely_ perform as well/poor as it did in non-prod. This is because

- the non-prod data is falling behind prod at every moment
- we are not testing _every possible_ flow and combination of flows, combination of parallel flows
- we are not testing the application with _every_ item (e.g. product, accessoary) and combination of items that are possible in prod.

If we had such a setup in non-prod and ran a well written suite of tests, we could be almost sure we'll know how PROD will perform. 

I'd like to stress the importance of having a copy of as recent as possible PROD database. Databases are complex mechanisms which usually are a bottleneck. I've seen many cases where performance was flawless in PLABs but it was significantly worse in PROD. Even though both the environments had nearly identical setups, similar amounts of data. Why is that? Database query execution plans. Most databases keep an eye on each table and their sizes. They also keep a track of how long SQL executions related to each of the tables are. As tables grow in size and SQL executions grow in duration, the DB engine can change the query execution plan. Usually, the newly assigned plan is faster and less resource-intensive. However, there are cases when new plans are slower or significantly slower. There are cases, when new plans need some DBA input, like creating a new index (or removing/changing an existing one) to boost the performance of the new plan. The more data tables have in the tables, the more likely some SQL execution plans are to be changed. The large the data amount difference in databases (prod and non-prod) is, the less aligned their performance will be.

The question is: which business is willing to invest this much in performance testing? I am yet to find one. And this is why clients usually agree to test performance in live PROD environment (which is no longer a part of the QA pipeline) along with some preminary testing in non-prod (Performance Lab (PLAB) environment, which has a similar setup to PROD, but not identical; and cheaper). The latter could be classified as QA, while the former - not really. Even more, performance testing can be carried out before QA phase, i.e. a developer has several possible solutions to a problem but performance is the main factor deciding on which to implement. In such cases perf testing can be carried out ad-hoc. And this is probably a gray zone between QA and non-QA.

## Monitoring
Since Security and Performance are not exactly QA, it's a good idea to set up monitoring for them. 

For _security_ - monitor logs for failed authorization/authentication assertions, request-per-ip rates, deviations from normal e2e flows, limit breaches (request rate, payload size, etc.), and errors in general. Keep a track of access logs with original IP addresses, so you could perform a post-mortem analysis in case of an incident.

For _performance_ - monitor metrics: response times, transactinons per second, error rates, infrastructure resources' utilization: as many and as fine-grained as possible; middleware resources' utilization (e.g. JVM GC, heap, off-heap, threads, JMX metrics,...; nginx connection count, latencies, etc.). In fact, monitor as many metrics as possible, as fine-grained as possible and store them for at least a month. Monitor PLABs and live environments.

That's a lot of investment in monitoring. And, frankly, this price is only a lesser part of the cost of an alternative: 

- poor performance and months wasted on investigations; 
- damaged image; 
- fines for data protection violations (data leaks), not to mention individual lawsuits if security breaches go public.

I always urge my clients to never be cheap when it comes to monitoring. It ALWAYS pays off. Not in increased revenue, but in significantly less expenses for maintenance and incident handling.

## It's an expensive investment. Is it worth the trouble?
Yes. For the most of it - yes. Of course, each client, each business must decide whether performance and security are important to them. If they are, they **must** be separate bulletpoints (with sub-points) in the contract. Performance and security are not a part of coding the application. These are completely different services from app coding, and they both require a great deal of investment. So if either security or performance or both are a critical for the business, there must be no cutbacks here. Most attempts to "make perf/sec testing cheaper" will rule results of those tests inconclusive and a completely useless waste of funds.

> Written with [StackEdit](https://stackedit.io/).
