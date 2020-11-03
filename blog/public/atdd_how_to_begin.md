# [A]TDD - how to begin
Basically there are 3 types of developers:
- those, who write the code and ship it
- those, who write the code and the cover it with tests before shipping
- and those, who write tests before writing the feature code

I believe I could guess you are the second one and I think I'd be correct most of the time. I do hope you are not the first type, and if, God forbid, you are - please stop being that person and become a  better one - start by testing your code.

## Coders
The first type of developers are what I like to call coders. They like to code. Quality, edge cases, exceptions to feature requirements, users' unpredictability - these are the least concerns of a coder. I've been one, you have been one, all the developers you know have been coders at one time or another. We are, to some extent, control freaks - we like to tell machines to do what we want them to do. We like the authority over them, so we drown in that authoritary role by writing instructions, and that drowning is a mental phase we like to call '_the zone_'. We don't really care whether machines obey them all - we're too busy writing instructions to spare our precious time for stupid verifications. We have QAs for that, for Pete's sake!

## Developers
Like the title says, these people are into development business. Their concern is to grow a product, to make sure it is reliable. One of the measures indicating this reliability is number of bugs - the less - the better. We are no longer focusing on writing code, not really, we have now grown enough to realise it is pointless to write code if you're going to throw it away soon enough due to bugs here and there and everywhere. We need our code to be working as it's supposed to - this requirement of compliance and desire to meet it is what has pulled us out of the coder's chair [or a bean-bag] into something a tad more advanced. Now we are responsible for the growth of the product, not just for solving puzzles and producing code. And how do we make sure the code works as expected? We test it. We write it and we put it to a test to make sure a colleague blabbing behind us, a meeting notification pop-up, a production bug discussion in that Slack channel, a Facebook message or some other event did not distract us too much to make a mistake, or a typo, that could be customer-facing. We now accept those mistakes as personal failures and we want to minimize them. So we write unittests. We put extra effort to ensure quality of our work. We realize that we all are human and that _errare humanum est_, and that there's no escape from that. Only when we see all the tests pass we are confident [enough] to ship our gem.

## Developers v2
_[yeah, I couldn't come up with a more sensible title. It's 11pm and I'm too sleepy :) ]_
We already know we cannot ship low quality code, but we got annoyed by writing all the tests, because we have to either spend more and more time on refactoring our code to make it testable, which is holding us back and makes us miss the deadlines, or we have to summon some reflection magic [directly or via some mocking/injection frameworks] to hack our code and test it. Needless to say, this is tedious. And to some extent embarrassing, because you have to admit you have written poor code, as it is not testable, and now you have to tear it all apart to be able to write tests for it. Wow, that doesn't sound very productive, does it? How could you think THAT far ahead to know that you might need to replace implementation of that class! It's simple - you couldn't. You were too focured on the task in hand, i.e. to make the feature work: you had it all planned out, you knew all the services, facades and adapters you'll need, you built them all. But it's a pipe dream to hope to predict ahead all the possible pots and bumps on the road. That's why we have Agile for one :) Equally naive is it to expect to plan all the code components ahead good enough to make them reusable, loosely coupled and, most importantly, testable.

So now we take a different approach - we write tests first and let them guide us into the code we need to write.

### Features and Failures
Ideally there is only one type of tasks we are supposed to be working on - features. We should be cooking and baking new features, adapting them to new requirements (which in fact is a new feature), and, when they are no longer required - removing them from the product. So how come we see another type of tasks in our Jira/Trello/GitLab/other boards? It says "Bug". What is that? Bugs are nothing but failures to deliver a feature. It could be a developer's fault, it could be a BA's, PM's, QA's or the client's fault. No matter who is it to blame, no matter how nicely you are trying to wrap them, that doesn't change the fact that **bugs are failures**. So we can go ahead and edit our Jira board and change it to have task types:

- Feature
- Failure

### The flaw of code tests
How do we prevent failures? We test our code, that's obvious. Unit tests, Smoke tests, E2E tests, integration tests, etc. - we have quite a bouquet of tests' types to keep our butts out of the fire when users start complaining after a new deployment. Yeah, great, but the users are still complaining, because they can't create a cart if their name in the profile contains non-alphabetical characters. 

> emm..
> huh..? What?
> 0_o
> .......
> What now?
> Why would anyone even...?

It's not your concern **why would anyone even**. Your job is to make a feature that's resistant to users' ignorance and carelessness (yes, I'm trying to put "stupidity" nicely). Show me your test for this scenario. Is it passing? Did it not fail in the deployment pipeline? Wait, what? You don't have this case covered? How come....?

> It never crossed my mind this could ever be the case

Of course it never did! Why? Because it's only natural. You were testing your **code** - your goal was to prove it works as expected. And the BA never would have even thought of this case too. He/she probably didn't even know users' names can contain a dash (`–`) sign! It's all fine - the code works. But the use-case is not covered. Because noone knew this use-case even exists.

And that's another example of _errare humanum est_. We can't possibly think of every real-life possible scenario. And we can't protect ourselves from the unknown. Naturally, we can't think of all the possible combinations of events and states, stars' alignment in the sky and tides in the East shore, and write tests covering all those cases. That's the flaw of code tests. 

### Discovering the unknown unknowns 
Yeah, that's never gonna happen. You will never discover all of them. Forget about it. 
Hold on buddy. Listen. As much as you love to write code, develop solutions, test them, talk to computers or whatever your coputers-related fetish is, you must remember: **life is not binary**. You can't always have 1s and 0s. You can't know everything for sure. You can't guarantee anything really. Because there's always a **possibility** something unexpected will happen. Did you think anyone was expecting the twin towers to fall? Did anyone take this possibility into account when preparing a software release for that day? Of course not. That's why they are called unknown unknowns.

They key word in the previous paragraph is *possibility*. Let's change it to **probability**, which defines how likely is the possibility to come true.

How likely are you to miss a state-induced use case in the new feature? Very. How do you reliably reduce it? Well' that's a billion dollar question right there. If there was a solution for all the cases, we would have no bugs in production ;) 

But there is one thing you could try. You could open up the feature description, open up all the use cases. Run through them and write them as code. Are you aware of the *given/when/then* trio?

### Convert use-cases to test-cases
This is one technique that has proven to me to be good way to catch undefined use-cases. It's also the best approach to write test-driven-code I have seen. It contradicts to what Robert C. Martin suggests (writing test cases in iterations, one at a time), but I find it works best for me. And in some sense it even compliments the approach suggested by Uncle Bob.

First, open up the feature documentation. Find the section with all the use-cases.
Then write down all the use-cases in the NewFeatureTest class as method names. For instance

> User can edit its profile and change the email address, username, password and can delete the account.

becomes
```
public class UpdateProfileTest {
	@Ignore @Test public void updatesEmail() throws Exception{}
	@Ignore @Test public void updatesUsername() throws Exception{}`
	@Ignore @Test public void updatesPassword() throws Exception{}`
	@Ignore @Test public void deletesAccount() throws Exception{}
}
```
This is just bare bones. This is the list of use cases that have been specified in the docs. And it is very important at this stage to **mark all the test methods with `@Ignore`**. Testing frameworks usually give you some kind of a warning when you have ignored tests. This way you will be reminded of all the test cases you have not implemented yet.

Don't think about the code yet. As far as you are concerned, there is no code yet. And you may not even need to write any. Just focus on the requirements for now.

### Expand test-cases and discover undefined use-cases
Now we will copy and rename things. We'll analyze each previously defined test method and think carefully about each and every one of them. Out goal is to think of as many states in the system that could
- have any impact on the feature's use case;
- be impacted by the feature's use case.

Focus. Don't think about the code you want to write. I know you want! Be patient. You'll get your chance. Even better - I promise you that if you're patient enough you will have to refactor it less / remove less of it! 

Now let's remember what the given-when-then is. It's a triplet of factors that add up to a test case. 
- The *given* is usually some state of the system (the user, the engine, the environment, etc.).
- The *when* is the action that's carried out. This is the use case copied in from the feature docs.
- The *then* is the result. It defines what happens when the use case is invoked while the system is in the *given* state.

It's a good idea to give your test cases names reflecting all the 3 parts: given-when-then. I for one like to use snake_andCammelCase for test names, i.e.
- `@Test public void given_when_then() {}`
- `@Test public void userAlreadyLoggedIn_logIn_newSessionIsCreated() {}`

#### - userEditsProfileUpdatesEmail
- Is the user logged in? Create a test case: `userNotLoggedIn_updatesEmail_unauthorizedErrorIsReturnedAndEmailUnchanged()`.
- Is the user's session expired? `userSessionExpired_updatesEmail_sessionIsRenewedAndEmailUpdated()`
- Is the user owner of that profile? `editsOtherUsersProfile_updatesEmail_forbiddenErrorIsReturnedAndEmailUnchanged()`
- Is the new email valid? `invalidNewEmail_updatesEmail_ValidationErrorIsReturnedAndEmailUnchanged()`
- Is the email the same? `sameNewEmail_updatesEmail_successAndEmailUnchanged()`
- Is the email the same and the previous email was invalid? `sameInvalidNewEmail_updatesEmail_successAndEmailUnchanged()` or perhaps you'd like to enforce the user to change the email to something valid? Now either make a judgement call, or take this question to the BA. Has the BA thought of such a use case? I bet he/she hasn't!
- ...
- add more cases - consider this an exercise for you.

Now instead of a single `updatesEmail()` we have 7 test cases, which are equal to 7 use cases. And one of those use cases needs to be clarified by the business! This way it's not only the business that's developing the feature definition - it's also all the developers. Anyone in the staff can add some use cases to that feature. It's a great approach as it adds more heads to the problem that needs as many different ideas as possible. Moreover, this approach involves people looking at the problem from very different angles!

Now do the same thing with `updatesUsername()` test case.  Expand it with different states and different outcomes. Cover as many cases as you can possibly think of and bring the questionable ones on the BA's desk.

**N.B. mark all the test cases with @Ignore**, otherwise chances are good you will forget to implement them, which will result in many passing tests (i.e. never failing tests; which is natural, because they don't test anything)

### Let's write some code now
You probably want to start with the sunny-path test scanario - the one that is straight-forward and is successful. Go ahead! Implement the `@Test public void updatesEmail() {}` method. Then implement all the others. It is quite likely you will come up with other, new test scenarios while you are implementing the initial set of test methods. Write those scenarios down as soon as they cross your mind! Don't forget them. You will implement them later on. Mark them as `@Ignored` until you implement them.

How to write the code you ask? Start with the method calls. What do you need for an `updatesUsername()`? You probably need an Account entity. So in the test method write: `Account account = createAccountWithEmail("some@old.email");`. Is the account logged in? `account.setLoggedIn(true)`. Then you may want to change that email to something else, so call another (probably not yet existing) method: `updateEmail(account, "new@email.com");`. And finally - the assertion - the part where you verify that things happened the way they were supposed to: `assertEquals("new@email.com", getAccount(account.getId()).getEmail());`. 
```
@Test
public void updatesUsername() {
	Account account = createAccountWithEmail("some@old.email");
	account.setLoggedIn(true);
	
	updateEmail(account, "new@email.com");
	
	assertEquals("new@email.com", getAccount(account.getId()).getEmail());
}
```
How does that look? Is it clean enough to understand right away what is this method testing? Is it clear what are the system states that are being tested? And what is expected after the `when` part is invoked? Well it is to me. I hope it's clear for you as well.

Now go ahead and implement all those methods one-by-one:
```
- createAccountWithEmail(email: String): Account
- updateEmail(account: Account, email: String): void
- getAccount(id: long): Account
```
Probably create some AccountService or AccountRepository instances and use them in those methods to invoke the business logic. Or build those services now if you don't have them yet. I find it easier to build them as inner classes and then refactor. Less tabs switching, less file contexts to keep track of :) 

### What else to write test-first?
Well, anything you want, really. But I suggest you to only cover units as high to the "surface" as possible. And by "surface" I mean a boundary: either user to the system, or the system to some other system (i.e. integrations). It's up to you to decide whether or not you can call a contract (an interface) between your internal classes an integration. If you feel thats the case - by all means write them in TDD as well. The more stable the boundary contract layer you need to be, the less likely is it to change - the more you should consider it as a subject to TDD.

By no means do I mean to say that some parts of the project are less important than the others. Importance is not a criteria I am basing the decision on. The likelihood to change - that's the criteria I'm using. The more likely the unit is to change its public properties, the less sense it makes to test it directly.

## But Uncle Bob says we must write all the units test-first! Not just the use cases
That is correct. Usually TDD evangelists ask us to write tests for all the classes, for all the public methods in the code. I tried that approach for quite a while and I found it... slow. Don't get me wrong, I can write tests fast - no problem there. But as it comes to refactoring the code that has been covered by tests long time ago, some problems arise:
- we need to remove all the old tests that our refactoring will break
- we need to write new tests to cover the refactored code. This is often times a test-after model, as you are *refactoring* - not creating something new.

It's all nice when you're creating new features, there's no problem there. But it gets far too tedious when you need to restructure your code. Even worse - when you split or merge your classes, DTOs or change their structure. You know you have your tests right there, covering all the functionality of the unit, but you also know you are going to break all those tests badly. Even more - you will break other tests that use that unit, EVEN if you were using all the correct abstractions! The chain reaction begins, and now you have a headache.

The higher-level TDD, or Acceptance-driven-TDD (ATDD) approach alleviates this problem by merging a BDD and TDD unit tests. I prefer choosing which units I'd like to test rather than just testing everything. Usually these units are
- all the classes that have public **interfaces** exposed
- all the utility classes

That's it. The classes with public interfaces usually have the feature use-cases converted into test-cases. And utilities - well, they are the tools used globally, so you *must* be sure they work, no matter what. 
What about the other units? What about the EmailValidator? Find a way to test them indirectly, through the public interfaces. Give the account an invalid email and save it - the validator will be invoked. Give the account a valid name and save it - the vaildator will also be invoked. And if you can't find a way to invoke the validator indirectly - then perhaps it's not really needed in the flow? Of course noone will slap on your wrist if you want to create the validators in the test-first approach, no sir! And there are cases when test-first in lower-level units makes sense. My point is you *don't have* to sweat too much about that. If you want to write them test-last - go ahead. After all they serve a sole purpose - to satisfy the tests that already are in your high-level test suite. If your lower units don't work, you'll see that immediately in your high-level unit tests. 

## How's that better than the test-all-units approach?
Your project remains flexible for any refactoring activities / code changes. All your features are covered by tests and your tests suite protects you from breaking any of them unknowingly. Refactoring lower-level units is less likely to impact your tests (assuming you are using correct abstractions). This approach hides your lower-level units as implementation details, just like a class encapsulates private methods and fields. Please note, that in order to keep the project maintainable (flexible, mobile) you need to structure the lower-level units accordingly. This approach (or any TDD approach for that matter) does not replace a need to stay SOLID and KISS at the rest of your code. 


## How's all that better than the code-first approach?
That is a very good question, thanks for asking. The test-first approach is significantly better [I don't make such claims liberally], although it might feel less convenient at first. I'll try to list some of the benefits.
- It makes you think before you write, which results in not-yet-considered use cases discovered before shipping the feature.
- Less coupled code, as you are forced to write your code modular. If you write a test for an AccountService, you need to think of a way to make the AccountRepository replaceable with a mocked repository, otherwise you won't be able to test your service (tests must NOT be making calls to the actual repositories).
- Units are designed to be close to a boundary where a boundary is applicable, as the test runs from the other side of the boundary. Units meant to be used by API consumers are immediately designed to be this way, as tests ARE the API consumers.
- It's features that's tested, not the code. And the features are the essence of the software, i.e. they define what the software can do, naturally defining the purpose of that software's existance.
- You see your goal right in front of you - in the tests - and you *know* when you achieve it. Test-first also eliminates the "is it done yet?" uncertainty.


## Summary
**ALWAYS TEST YOUR CODE**. How you test it - that's up to you. But you must test. Mock the boundaries, replace some parts of your code with test doubles (spies, mocks, dummies, stubs) where you have to, but cover your code with tests. Ideally you would only mock boundaries and test all the moving parts in your code, making your unit tests a kind of localized integration tests. The higher-level units you are testing, the more certain you are your code works as expected. 
If you design your tests to verify high-level units with test cases matching all the feature's use-cases, you will know at any point in time which use-cases will definitely work in production. 
If you are testing code units - you know the code will work. If you are testing business units, i.e. features and their use-cases, then you know the features will work. And I believe the purpose of the code is to make the working features.

Test-first approach makes you prevent many undefined behaviours in the feature as well as structure your code correctly. ATDD gives us the luxury of certainty about our feature's state concerning its development progress. "Is this use-case working already?", "Is the feature development complete?", "What use-case do I do next?", "Did I go sideways while I was in the zone?" and other questions are automatically answered by [A]TDD.


> Written with [StackEdit](https://stackedit.io/).
