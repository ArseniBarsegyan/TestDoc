# TM Manager mobile app

## Architecture
Application architecture based on .Net Core DI, MVVM implemented with IViewPresenter interface in core project, as DroidViewPresenter on Android (every view model has appropriate named fragment), on iOS as IosViewPresenter (every view model has appropriate named view controller)

![Architecture image](/TMManagerDiagram.png)

## Solution structure
![Solution structure image](/Solution_screenshot.png)

**AssigmentManagerMobile.Core.Data**
Contains models for entire app

**AssigmentManagerMobile.Core.Interfaces**
Contains interfaces of services for entire app (logging, api, commanding, navigation services), refit services (REST)

**AssigmentManagerMobile.Core**
Project with common logic of application. Implemented interfaces from AssigmentManagerMobile.Core.Interfaces, contains view models, shared resources (Colors, Dimensions, Strings, View Ids). All shared dependencies configured here

**AssigmentManagerMobile.Droid**
Xamarin android native project, reference to AssigmentManagerMobile.Core.Data, AssigmentManagerMobile.Core.Interfaces and AssigmentManagerMobile.Core. All platform-specific dependencies registered here. Contains platform-specific logic (navigation implementation and UI), use shared code.

**AssigmentManagerMobile.iOS**
Xamarin iOS native project, reference to AssigmentManagerMobile.Core.Data, AssigmentManagerMobile.Core.Interfaces and AssigmentManagerMobile.Core. All platform-specific dependencies registered here. Contains platform-specific logic (navigation implementation and UI), use shared code. No storyboards or xib files. ViewControllers and views are written in code.

**AssigmentManager.Fake**
Contains mock (fake) implementations for AssigmentManagerMobile.Core.Interfaces.
Use mostly for UI Tests project and for DebugMock configuration.
AssigmentManagerMobile.Core, AssigmentManagerMobile.Droid, AssigmentManagerMobile.iOS
refer this project in DEBUGMOCK and UITESTS configurations.

**AssignmentManagerMobile.Tests**
Contains unit tests for Shared logic (AssigmentManagerMobile.Core). 
All mock objects are registered in DI pipeline and passed to AssigmentManagerMobile.Core
Tests covers http policy tests and view models logic testing. In order to run unit tests properly
run project in TEST configuration.

**AssignmentManagerMobile.UITests**
Contains UI tests for entire app for Android and iOS. Project use SpecFlow and separate testing to features.
For cross-platform UI testing every UI element which is being checked should contain special id (android:contentDescription on Android, AccessibilityIdentifier property on iOS). Every feature runs independently. Some tests covers different UI conditions that depends on api-response. For this purpose tests use backdoor methods. Backdoor methods implemented for Android and iOS and included in source code only for UITESTS configuration. In order to run UI tests properly app should be installed on device with configuration UITESTS.

## Configurations
App has next configurations: DEBUGMOCK, DEBUGREAL, STAGING, STORE, TEST, UITESTS.

**DEBUGMOCK** configured to use fake AssigmentManager.Fake project instead of real implementations.

**DEBUGREAL** configured to use real API of test environment.

**STAGING** configured to use real API of staging environment.

**STORE** configured to use production API and contains production-ready app config.

**TEST** configured to run AssignmentManagerMobile.Tests. In this configuration app use mocks instead of real or fake services.

**UITEST** In this configuration app use fake services from AssigmentManager.Fake. Platform projects also implement backdoor methods for this configuration for UI testing.

## Hacks
**AssigmentManagerMobile.Core** has resources that are shared across Android and iOS.
These resources are: Colors, Dimensions, Strings, ViewsIds. To generate these C# values into platform-specific values
project use T4 templates. This template create XML-resource files on Android C# classes on iOS from shared code.

**AssigmentManager.Fake** included in **AssigmentManagerMobile.Core** only for DEBUGMOCK or UITESTS configurations.

**Running tests**
To run UI tests first install app on device with configuration UITESTS. On iOS, replace device id with your simulator id at AppInitializer.cs

## Http requests and authentication
App use Http clients registration in .Net Core DI pipeline with http handlers and retry policy setup (https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-3.1). All http clients are registered in Startup.cs in AssigmentManagerMobile.Core. All http clients configured with http request handlers. There are several http request handlers in app.

**Primary message handler** 

Native message handler for every platform (DroidClientHandler on Android, NSUrlSessionHandler on iOS).

**AuthenticatedHttpClientHandler** 

Used for authorize user and retrieving access token.

**LoggingHttpClientHandler** 

Used for logging http request. Logs are sent to AppCenter analytics.

**Policy**

Http clients are using policy for http errors. On 400, 403, 408, 5XX http error code http client retry attempt 2 times (0.2, 0.5 sec time span). If error repeat, it'll be thrown.

**Authentication flow**

During http request http handlers handle request (chain of responsibility pattern), modify it, if necessary, and then throw it to next http handler. API requires user authentication in system (access token is required), **AuthenticatedHttpClientHandler** checks if token present in http request headers. If no token founded, this handler redirect request to identity server. After successful login token got saved to secure local storage, added to http request header and request performed to API.

## Deeplinking

App use deeplinking for performance review section. Deeplinking has next flow:
- User recieve e-mail with link
- User opens link and select installed app as link handler
- App open and navigate user to performance review section

Also, deeplinking works for push-notification.
- User recieve push notification
- User tap on notification
- App open and navigate user to performance review section

## Workflow
**Using shared code**

AssigmentManagerMobile.Core contains T4 templates (https://docs.microsoft.com/en-us/visualstudio/modeling/writing-a-t4-text-template?view=vs-2019). Core project contains shared resources, stored in /Resources folder. Some of shared resources can't be used directly in platform projects. T4 templates will generate platform-dependant resources. All templates are stored in AssigmentManagerMobile.Core/Templates. 

In order to use auto-generated resources in platform projects:
- Update resources (/Resources folder)
- Open SaveMe.tt T4 template and save it.This action will trigger resources update for platform projects.
- Now you can access resources in platform projects.

T4 templates auto-update strings.xml, colors.xml, dimens.xml on Android. 
On iOS T4 templates auto-update Helpers/Colors. Other shared code can be accessed directly from Core project.

**Add new screen in app** 

AssigmentManagerMobile.Core:
- Create new ViewModel in AssigmentManagerMobile.Core/ViewModels folder.
- Your view model should extend BaseViewModel.
- If you're going to use navigation data, extend BaseViewModel<TNavigationData>.
- All required services should be injected into view model's constructor.
- Register view model in DI pipeline in Startup class, ConfigureServices() method.

Android: 
- Create new fragment in AssigmentManagerMobile.Droid/Fragments and extend BaseFragment.
- Your fragment should extend BaseFragment<TViewModel>.
- Fragment name should have view model's name and "Fragment" suffix.
- Register fragment in DI pipeline in MainActivity, ConfigurePlatformServices() method.
- Override SetBindings() and use MVVM Light extensions to create bindings from fragment to view model.
- All bindings should be stored as private fields of ViewController. That should be done because of weak references issue.
  (https://docs.microsoft.com/en-us/archive/msdn-magazine/2015/june/xamarin-implementing-and-using-data-binding-in-xamarin).
- Override CleanBindings() and clean bindings (if it's possible).

iOS: 
- Create new view controller in AssigmentManagerMobile.iOS/ViewControllers.
- Your view controller should extend BaseViewController<TViewModel>.
- View Controller name should have view model's name and "ViewController" suffix.
- Register view controller in DI pipeline in AppDelegate, ConfigurePlatformServices() method.
- Override SetBindings() and use MVVM Light extensions to create bindings from fragment to view model.
- All bindings should be stored as private fields of ViewController. That should be done because of weak references issue.
  (https://docs.microsoft.com/en-us/archive/msdn-magazine/2015/june/xamarin-implementing-and-using-data-binding-in-xamarin).
- Override ClearBindings() and clean bindings (if it's possible).

**Write unit test for screen**

AssignmentManagerMobile.Tests using NUnit, Moq, Shouldly. Unit tests mock http requests with MockHttpClientHandler. Tests covers  Http policy tests and view models tests. Policy tests covers cheks if token present in http headers when http request is performed, http request should be invoked once if token is present in http header during requests, and retry http request headers policies are working ok.Tests should be written with next requirements:
- Tests should cover all public methods of view models.
- Every test method should cover only one state of unit of business logic (for instance, method in view model).
- Unit of business logic should be covered by all possible tests.
- When unit of business logic contains several logic states, start from all negative conditions, then positive-negative and it's combinations (for example, false-false-true, false-true-true, true-false-true etc.), at the end test all positive conditions).
- Every logic state test should be tested separate method.
Tests run in **TEST** configuration.

Order of writing unit test for screen:
- Register mocks in AssignmentManagerMobile.Tests/RegistrationProviders/UnitTestRegistrationProvider.
- Create test for view model in AssignmentManagerMobile.Tests/ViewModelsTests and extend TestBase class.
- In you test class resolve mocks with Resolve<T> method and setup this mock as you want.
- Use shouldly extensions to verify test results
  
To run Unit tests:
- Run AssignmentManagerMobile.Tests project in **TEST** configuration.

**Write UI test for screen**

AssignmentManagerMobile.UITests project use SpecFlow, Xamarin UITest. 

Strongly recommended installation of SpecFlow extension to Visual Studio (see https://specflow.org/getting-started/#InstallSetup)
All tests written as Features and Steps. To test screen you should:
- Add new .feature and Feature.cs files to /Features.
- In .feature file describe steps to reproduce your test conditions. Use existing steps.
- Feature.cs file should extend FeatureBase (see other features)
- Generate Steps file in /Steps folder.
- Change Steps file to extend StepsBase.
- Write steps to reproduce test conditions.
- Use shouldly extensions to verify test results
- If you need to test UI for several states (for instance, availability of buttons that depends on user permissions), use BackdoorMethods (see https://docs.microsoft.com/en-us/appcenter/test-cloud/uitest/working-with-backdoors)
- All backdoor methods should be created in MainActivity (Android) and AppDelegate (iOS), for **UITESTS** configuration only.
- Use BackdoorHelper.cs to call these methods in your UI test. Use Pascal case method name (default C# method naming convention) as argument when calling Invoke() method.

To run your tests:
- Install app with **UITESTS** configuration.
- Run UI tests in **UITESTS** configuration.
- For iOS simulator replace DeviceIdentifier in AppInitializer.cs
