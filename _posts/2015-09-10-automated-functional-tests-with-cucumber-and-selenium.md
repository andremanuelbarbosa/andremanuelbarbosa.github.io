---
layout:      post
title:       Automated Functional Tests with Cucumber and Selenium
description: 
headline:    "Automated Functional Tests with Cucumber and Selenium"
categories:  [Cucumber,Selenium]
tags:        [ATDD,BDD,Cucumber,Selenium,Java]
image:       
comments:    true
mathjax:     
featured:    true
published:   true
---


This approach and technologies involved are not new and have been around for several years but have gained momentum in recent years on some development teams, which have embraced the BDD (Business Driven Development) approach to shorten the understanding gaps between the business stakeholders, developers and testers. 

## BDD
Sometimes referred to as an improved version of TDD (Test Driven Development) or the test cases for dummies, [BDD](https://en.wikipedia.org/wiki/Behavior-driven_development) is a development approach which encourages (almost) plain English to be used in order to describe the behavior of an application. While TDD focus more on the results of specific units of code (methods, classes, modules, etc), BDD is more focused on the feature (behavior) being delivered. Is BDD a replacement for TDD? Well, I think that is a topic for another Blog Post but the idea is that both can and should be used in conjunction, if applicable. What I tend to do is to use BDD on a higher level and use TDD to validate the lower level units of code when the underlying design is complex enough. Oh, did I also mention [ATDD](https://en.wikipedia.org/wiki/Acceptance_test-driven_development) (Acceptance Test Driven Development)?

![ITCrowdThrowingMonitor](/images/posts/2015-09-10-automated-functional-tests-with-cucumber-and-selenium/ITCrowdThrowingMonitor.gif "ITCrowdThrowingMonitor")

### Given-When-Then
One of the most important concepts in BDD is the Scenarios and how these should reflect the expected behavior of the application. As usual, there are a lot of [discussions](http://programmers.stackexchange.com/questions/182158/relationship-between-user-story-feature-and-epic) around the differences between User Stories, Features, Requirements, Epics and Scenarios but for the purpose of this discussion lets focus on the [GWT](https://en.wikipedia.org/wiki/Given-When-Then) (Given-When-Then) format and how this is crucial to create our Scenarios and the underlying test code. No rocket science, anyone can write them up, including the Customer or the PM:
* (Given) some context.
* (When) some action is carried out.
* (Then) a particular set of observable consequences should obtain.

## Cucumber
So, what is Cucumber? Are we talking about the green vegetable or the British TV [series](http://www.imdb.com/title/tt3848112/) that takes place in Manchester? No for the vegetable and in Manchester there is ONLY the United FC! In a nutshell, [Cucumber](https://cucumber.io/) is a BDD framework with support for multiple languages that enables test automation and cooperation between all people involved in the development process.

### Features, Scenarios and Steps
Cucumber requires specifications written in Gherkin (the plain-text English language with 60+ other also supported) and the basic structure boils down to Features, Scenarios and Steps. You can find a lot more information about these concepts and others on the reference [documentation](https://cucumber.io/docs/reference) which is available on their website. The scenarios and steps are created in a feature file with the following format:

![FeaturesScenariosSteps](/images/posts/2015-09-10-automated-functional-tests-with-cucumber-and-selenium/FeaturesScenariosSteps.png "FeaturesScenariosSteps")

### Steps Definitions
Show me some code man! For each Step created on the feature file, Cucumber expects to find a Step Definition which is the underlying code that supports the Step. The two images below shown an example Step created on a Scenario and some Java code used as the Step Definition. Cucumber looks for the annotation @Given on the underlying code and matches the Step based on the Regular Expression defined inside the annotation. Makes sense? No? Go back to the reference documentation, which explains this a lot better.

![StepsDefinitions](/images/posts/2015-09-10-automated-functional-tests-with-cucumber-and-selenium/StepsDefinitions.png "StepsDefinitions")

## The Real Stuff
Enough talking about concepts, lets see an actual example of Cucumber and Selenium in action. While doing some customizations for a customer a while ago, I came across the Change Password bug which was a showstopper for the customer. What is the first thing to do? Write a Cucumber Scenario that fails!
```gherkin
Feature: Change Password  
  
  Scenario: User changes its Password and should be able to Sign In with the new Password  
   Given a new User  
   And the User navigates to "/"  
   And the User is Signed In  
   And the User selects the Change Password Option  
   And the User enters the Current Password and a New Password  
   And the User Signs Out  
   When the User Signs In using the New Password  
   Then the User should be Signed In  
```

### Under the Hood
So we have our nice feature file that reflects the Change Password scenario and the expected outcome but how do we make this runnable? We need to write steps definitions for all the steps which have been listed on that feature file. For the step "Given a new User", we need to create a User which will be used for the test execution and I have decided to do this using an API call to Create Person endpoint:
```java
@Given("^a new User$")  
public void a_new_User() throws Throwable {  
  
  HashMap<String, Object> user = createUser("user.cucumber", "user.cucumber", "User", "Cucumber", "user.cucumber@localhost");  
  
  HttpPost httpPost = new HttpPost(API_PEOPLE_PATH);  
  httpPost.addHeader("Content-Type", MediaType.APPLICATION_JSON);  
  httpPost.setEntity(new StringEntity(createJsonUser(user)));  
  HttpResponse httpResponse = httpClient.execute(HTTP_HOST, httpPost, HTTP_CLIENT_CONTEXT);  
  
   assertEquals(Response.Status.CREATED.getStatusCode(), httpResponse.getStatusLine().getStatusCode());  
  
  String jsonUserCreated = EntityUtils.toString(httpResponse.getEntity());  
  
  HashMap<String, Object> userCreated = objectMapper.readValue(jsonUserCreated, TYPE_REFERENCE);  
  user.put(PARAMS_KEY_USER_MAP_KEY_ID, userCreated.get("id").toString());  
   scenario.write("User: " + user.toString());  
  
   scenarioParams.put(PARAMS_KEY_USER, user);  
}
```

Other than the first step, all other steps are based on browser actions so we need to interact with Selenium, in particular with the WebDriver class. The code below shows what is happening under the step "When the User Signs In using the New Password":
```java
@When("^the User Signs In using the New Password$")  
public void the_User_Signs_In_using_the_New_Password() throws Throwable {  
  
  userSignsIn((Map) scenarioParams.get(PARAMS_KEY_USER));  
}  
  
void userSignsIn(Map user) throws InterruptedException {  
  
  String userId = user.get(PARAMS_KEY_USER_MAP_KEY_ID).toString();  
  
   webDriver.findElement(By.id("username01")).sendKeys(user.get(PARAMS_KEY_USER_MAP_KEY_USERNAME).toString());  
   webDriver.findElement(By.id("password01")).sendKeys(user.get(PARAMS_KEY_USER_MAP_KEY_PASSWORD).toString());  
   webDriver.findElement(By.id("loginform")).submit();  
  
  clearOnboardingWelcome();  
  
  waitAndTakeScreenshot(WAIT_TIME_MILLIS);  
  
   assertTrue(isUserSignedIn(userId));  
}  
```

One of the most important steps while writing automated tests is to make sure that the test data is removed after the tests have been executed. In this example, we have created a User as part of this scenario so we need to remove this Person. Once again, I have decided to call the API to achieve this, using the Delete Person endpoint:
```java
@After  
public void tearDown(Scenario scenario) throws Throwable {  
  
   if (scenarioParams.containsKey(PARAMS_KEY_USER)) {  
  
  HashMap<String, Object> user = (HashMap<String, Object>) scenarioParams.get(PARAMS_KEY_USER);  
  
  String userId = user.get("id").toString();  
  
  HttpDelete httpDelete = new HttpDelete(API_PEOPLE_PATH + "/" + userId);  
  HttpResponse httpResponse = httpClient.execute(HTTP_HOST, httpDelete, HTTP_CLIENT_CONTEXT);  
  
   assertEquals(Response.Status.ACCEPTED.getStatusCode(), httpResponse.getStatusLine().getStatusCode());  
  
  waitForUserToBeRemoved(userId);  
  }  
}  
```

### Failed Execution
Now that we have the Cucumber feature file and the underlying code lets see what happens when we execute this feature. Since we now that the Change Password feature is not working we expect this scenario to fail. IntelliJ IDEA provides some cool integration with Cucumber and is quite simple to execute features, scenarios and even steps. The screen recording below shows the execution of Cucumber scenario defined above:
<video width="494" height="279" controls="" poster="/images/posts/2015-09-10-automated-functional-tests-with-cucumber-and-selenium/FailedExecutionThumbnail.jpg" id="4697a5631e324"><source src="/images/posts/2015-09-10-automated-functional-tests-with-cucumber-and-selenium/FailedExecution.mp4"></video>

### Passed Execution
We have seen the test failing and now we need to fix the actual problem. The outcome should now be that the tests pass for that particular scenario, as we can see in the screen recording below:
<video width="494" height="279" controls="" poster="/images/posts/2015-09-10-automated-functional-tests-with-cucumber-and-selenium/PassedExecutionThumbnail.jpg" id="4697a5631e324"><source src="/images/posts/2015-09-10-automated-functional-tests-with-cucumber-and-selenium/PassedExecution.mp4"></video>


Guess what? The problem has been solved!

![Celebration](/images/posts/2015-09-10-automated-functional-tests-with-cucumber-and-selenium/celebration.gif "Celebration")

## Conclusions
The biggest challenge I have encountered in order to complete this demo is the build up time as I needed several hours to setup the required tools, prepare the basic steps and getting Cucumber to run. Once this basic infrastructure was in place, the steps definitions required for the remaining steps were quite straightforward as most of the code is reusable. I guess this raises the question of effort vs what we can get in return. With the right framework in place, maintained and improved over time, and if the adoption would be strong amongst the team members, I believe the benefits are much higher than the cost of following this approach but this is open for debate of course.