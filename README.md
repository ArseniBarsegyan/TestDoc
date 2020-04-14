# TM Manager mobile app

## Architecture:
Application architecture based on .Net Core DI, MVVM implemented with IViewPresenter interface in core project, as DroidViewPresenter on Android (every view model has appropriate named fragment), on iOS as IosViewPresenter (every view model has appropriate named view controller)

![Architecture image](/TMManagerDiagram.png)

## Solution structure:
![Solution structure image](/Solution_screenshot.png)

**AssigmentManagerMobile.Core.Data**
Contains models for entire app

**AssigmentManagerMobile.Core.Interfaces**
Contains interfaces of services for entire app (logging, api, commanding, navigation services), refit services (REST)

**AssigmentManagerMobile.Core**
Project with common logic of application.
Implemented interfaces from AssigmentManagerMobile.Core.Interfaces, contains view models, shared resources (Colors, Dimensions, Strings, View Ids).
All shared dependencies configured here

**AssigmentManagerMobile.Droid**
Xamarin android native project, reference to AssigmentManagerMobile.Core.Data, AssigmentManagerMobile.Core.Interfaces and AssigmentManagerMobile.Core.
All platform-specific dependencies registered here.
Contains platform-specific logic (navigation implementation and UI), use shared code.

**AssigmentManagerMobile.iOS**
Xamarin iOS native project, reference to AssigmentManagerMobile.Core.Data, AssigmentManagerMobile.Core.Interfaces and AssigmentManagerMobile.Core.
All platform-specific dependencies registered here.
Contains platform-specific logic (navigation implementation and UI), use shared code.

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
For cross-platform UI testing every UI element which is being checked should contain special id (android:contentDescription on Android,
AccessibilityIdentifier property on iOS).
Every feature runs independently. Some tests covers different UI conditions that depends on api-response.
For this purpose tests use backdoor methods. Backdoor methods implemented for Android and iOS and included in source code
only for UITESTS configuration. In order to run UI tests properly app should be installed on device with configuration UITESTS.

## Configurations:
App has next configurations: DEBUGMOCK, DEBUGREAL, STAGING, STORE, TEST, UITESTS.

**DEBUGMOCK** configured to use fake AssigmentManager.Fake project instead of real implementations.

**DEBUGREAL** configured to use real API of test environment.

**STAGING** configured to use real API of staging environment.

**STORE** configured to use production API and contains production-ready app config.

**TEST** configured to run AssignmentManagerMobile.Tests. In this configuration app use mocks instead of real or fake services.

**UITEST** In this configuration app use fake services from AssigmentManager.Fake. Platform projects also implement backdoor methods for this configuration for UI testing.

## Hacks:
**AssigmentManagerMobile.Core** has resources that are shared across Android and iOS.
These resources are: Colors, Dimensions, Strings, ViewsIds. To generate these C# values into platform-specific values
project use T4 templates. This template create XML-resource files on Android C# classes on iOS from shared code.

**AssigmentManager.Fake** included in **AssigmentManagerMobile.Core** only for DEBUGMOCK or UITESTS configurations.

## Running tests:
To run UI tests first install app on device with configuration **UITESTS**. On iOS, replace device id with your simulator id at AppInitializer.cs

## Workflow:
**Add new screen in app** 

- Create new ViewModel in AssigmentManagerMobile.Core/ViewModels folder and extend BaseViewModel. All required services should be injected into constructor. 
- Register view model in DI pipeline in Startup class.

Android: 
- Create new fragment in AssigmentManagerMobile.Droid/Fragments and extend BaseFragment.
- Register fragment in DI pipeline in MainActivity.
- Use MVVM Light extensions to create bindings from fragment to view model.

iOS: 
- Create new view controller in AssigmentManagerMobile.iOS/ViewControllers.
- Register view controller in DI pipeline in AppDelegate.
- Use MVVM Light extensions to create bindings from fragment to view model.

**Write unit test for screen**
AssignmentManagerMobile.Tests using NUnit, Moq, Shouldly.
All unit tests covers view models tests. Tests run in **TEST** configuration.

- Register mocks in AssignmentManagerMobile.Tests/RegistrationProviders/UnitTestRegistrationProvider.
- Create test for view model in AssignmentManagerMobile.Tests/ViewModelsTests and extend TestBase class.
- In you test class resolve mocks with Resolve<T> method and setup this mock as you want.
- Use shouldly extensions to verify test results
