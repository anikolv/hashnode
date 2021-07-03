## The testing pyramid strategy - practical application

Software testing is an activity which ensures that a) the software we are implementing is doing the job as per functional and non-functional requirements and b) all changes are backward compatible, meaning they doesnt introduce regressions in the already implemented functionalities. While I am mentioning those as a separate points, they are one and the same thing. All tests, initially implemented to test a given feature, make sure that later-on this feature is not being regressed. This article aims to share hands-on experience on some testing strategies with focus on effectiveness, rather than dogmas (TDD is the holly grail, we need to test everything VS tests are waste of time, writing tests takes 3x more time for implementing than the actual production code).

##  The testing frame

As I mentioned, our testing plan primary goal should be to ensure requirements fulfillment and preventing regressions. Now, although there are some sceptics on the well-known Testing pyramid concept, I will use it as a framework for better understanding. Here is the very basic Testing pyraimd, which is simple enough for our purposes:


![1_Tcj3OsK8Kou7tCMQgeeCuw.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1625078152092/X5BEg2uM1.png)

There is one simple rule - the speed of a given test inversely proportional to the scope it covers. Functional testing of stateless method - fast. E2E testing including third party integrations, DB invocations, web services, etc - slow. In addition, the biggest the scope is, the less stable and harder to maintain it is. Lets try to drill down into the pyramid.

##  Unit testing layer

Traditional, isolated (using mocks) unit testing is per say the lowest testing tier. We must have big enough amount of unit tests, because a) they are fast and easy to maintain (if not poluted with infrastructure) and b) they are the shortest feedback loop for regressions being part of the build. But unit tests can be devious, especially when relying on test coverage as a metric. Why? Because it gives fake feeling of security. If a peace of code can go faulty in a hundred ways, you need just a few tests to fake cover it. And those test cases are the ones picked up by the dev depending on what he managed to determine as worthy to test while writing them. Additionally, when devs are forced to achieve a given test coverage, they start to write tests to just pass the gate. And the gate is passed, but the coverage is not a coverage at all. Also, there is no way to force writing more tests for critical piece of code, for example fee service calculating taxes. Going faulty, this service costs money. While other piece of code might not lead to significant issues, even if some cases are not covered. Because of all those rabbit holes, I prefer more practical approach. Here are some ideas.

1. Good amount of unit tests

2. Stable, quick and decoupled from infrastructure as much as possible

3. Maintain tests readable and clean as much as possible. Shitty test classes makes next devs trying to avoid dealing with them, spinning the wheel of legacy code with skip tests flags by default.

3. Focus on testing code which is more valuable and critical. This means trying to cover it with as much test cases as possible. Of course, any code can have tests, but covering simple controller with huge amount of tests is not as valuable, as covering critical service.

4. Make sure its easy to plugin new tests with less technical debt in the test infrastructure. Yes, for legacy its hard, but at least do it for new test classes you create aka The Boy Scout Rule.

5. Dont follow dogmas, neither TDD solves all problems, nor unit tests neglection should be tolerated. Apply the strategy that fits with your context and does the job most efficiently for you.

6. Component tests could be also pain in the ass, especially if they need to initialize some infrastructure to run (for example Spring context). As time goes, there is ALWAYS issues with such tests which end up skipped because they require too much effort to be fixed.. So - component tests are useful in some cases, but be careful if they require too much infra initialization to run properly, see point 2.

##  Integration testing layer

Fine, we have good amount of fast unit tests, being the foundation of our testing pyramid. But we still have mocked our DB, third party invocations and even external configuration stores. Here it comes - integration testing. Validating some DAO/Repository, making sure some integration chain is working as expected, etc. The biggest issue with integration tests is that they are SLOW. The moment we include DB invocation or web service call, test performance degradation is a fact. Here are some ideas for more efficient integration testing strategy.

1. Isolate your integration test scope as much as possible. If you want to test your repository/DAO, mock anything else.

2. Try to use in-memory solutions for DB related tests, they are much faster.

3. Try to segregate your unit tests suites from your integration tests suites. This enables running them in a separate profiles/build phases. For example we can run integration tests only as part of the CI-CD pipeline, but running them locally can be optional. The moment we force devs to run slow integration tests every time on development environment, we loose them performing tests at all.

4. If you want to test single DAO, its fine. However, integration tests should be relatively small amount, so be careful and dont integration test everything.

5. Dont confuse integration tests with E2E tests. Do not test whole workflows and processes in a single test.

##  E2E testing layer

And finally the top of our pyramid - the E2E testing. We might have our manual QAs, which use tools like TestRail to document and follow E2E test cases. We also might have automation testing pipelines. Or both. It doesnt matter, in both cases there are some guidelines which might be useful, depending on the specific context. 

1. E2E tests are slow, unstable and hard to maintain. Thats true for both manual and automation.

2. Automated E2E tests should not be part of the build in any form. They should be part of the CI-CD pipeline, but as a standalone process.

3. Automated E2E tests (at least those, which are part of the CI-CD, like regression and sanity suites) should be covering general business process cases which are ensuring the health of the system. Minimal or no corner cases.

4. Regression and sanity automated suites should be executing exactly those tests which ensure the system is able to perform its basic functions.
 
5. It is good to have a dedicated automation QA team supporting the suites. They are  getting unstable and outdated in time, so they require day-to-day support. Especially for bigger companies.

6. Manual regression and sanity testing is way too slower, so it requires very good test case planning, preparation and at least basic automation for some activities.

7. Manual testing is very convenient for testing specific user interactions, play with some corner cases, try out validations. This is useful for testing new features primarily. Regression and sanity - better automated.

This article aims to brief some hands-on tips related to the common testing pyramid we all deal with on a daily basis. There are a lot of amazing articles from Martin Fowler, uncle Bob, Kent Beck and other industry gurus, which can dive you into the software testing science in dept.



