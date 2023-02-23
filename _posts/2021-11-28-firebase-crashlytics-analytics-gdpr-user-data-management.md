---
layout: post
title:  "[ios] Firebase Crashlytics/Analytics, GDPR, and user data management"
categories: ios firebase
---

Most developers and companies building iOS apps want to have insights into the usage of their apps and get detailed reports when their app crashes. There are many solutions on the market, but Firebase [Analytics](https://firebase.google.com/products/analytics){:target="_blank"}<!-- markup clean_ --> and [Crashlytics](https://firebase.google.com/products/crashlytics){:target="_blank"}<!-- markup clean_ --> services are common choices because they are very well known and can be used for free.

When integrating these services, an important topic to have in mind is user data management and privacy regulations compliance. Depending on your target audience, you might need to comply with the EU General Data Protection Regulation (GDPR) or the California Consumer Privacy Act (CCPA). You will also need to adhere to Apple's Terms of Service.

Before we go further, bear in mind, I'm not a lawyer, and this is not legal advice. If you have access to a legal advisor, please consult with them. Even as I'm writing this article, I still have more questions than answers on this topic. Nonetheless, I think it's worth sharing my findings and insights as they might help you navigate the integration of Firebase SDKs.

The goal is to integrate Firebase Analytics and Crashlytics services with minimal risk of exposure to government penalties or Apple rejecting your app. What follows is an overview of the legal obligations and technical guidelines for a safe and compliant integration.

GDPR on personal data processing
--------------------------------

[GDPR](https://eur-lex.europa.eu/eli/reg/2016/679/oj){:target="_blank"}<!-- markup clean_ --> says the following in Article 6:
>  Processing shall be lawful only if and to the extent that at least one of the following applies:
>
> (a) the data subject has given consent to the processing of his or her personal data for one or more specific purposes;
>
> ...

As the quote indicates, several other bullets can make the user data processing lawful under GDPR, but bullet *a)* is most likely to be applicable in tracking user behavior and app crashes.

Apple Developer Program License Agreement on data collection
------------------------------------------------------------

As we all know, Apple has a strong focus on protecting user's privacy, so it's no surprise to read the following in Section 3.3.9 of Apple Developer Program License Agreement:
> You and Your Applications (and any third party with whom You have contracted to serve advertising) may not collect user or device data without prior user consent, whether such data is obtained directly from the user or through the use of the Apple Software, Apple Services, or Apple SDKs (...) You may not use analytics software in Your Application to collect and send device data to a third party. Further, neither You nor Your Application will use any permanent, device-based identifier, or any data derived therefrom, for purposes of uniquely identifying a device.

This excerpt is from the [agreement dated June 2021](https://developer.apple.com/support/downloads/terms/apple-developer-program/Apple-Developer-Program-License-Agreement-20210607-English.pdf){:target="_blank"}<!-- markup clean_ -->. For the latest version of the Apple Developer Program License Agreement, visit [https://developer.apple.com/support/terms/](https://developer.apple.com/support/terms/){:target="_blank"}<!-- markup clean_ -->.

Crashlytics Terms for developers on user data management
--------------------------------------------------------

In Section 2.6, [Crashlytics Terms](https://firebase.google.com/terms/crashlytics#section_2_specific_terms_for_developers){:target="_blank"}<!-- markup clean_ --> dated September 23, 2019, says the following:
> For Developer's users in the European Union, Developer shall provide such users with clear notice of, and obtain such users' consent to, the transfer, storage, and use of their information in the United States and any other country where Google, or any third party service providers acting on its behalf, operates, and shall further notify such users that the privacy and data protection laws in some of these countries may vary from the laws in the country where such users live.

Analytics Terms on user data privacy
------------------------------------

In Section 7 titled "Privacy", [Google Analytics for Firebase Terms](https://firebase.google.com/terms/analytics){:target="_blank"}<!-- markup clean_ --> dated April 17, 2019, says the following:
> You will use commercially reasonable efforts to ensure that a User is provided with clear and comprehensive information about, and consents to, the storing and accessing of cookies or other information on the User’s device where such activity occurs in connection with the Service and where providing such information and obtaining such consent is required by law.

Summary of the legal and contractual obligations
-------------------------------------------------

It seems evident that the app should provide information about what kind of data will be collected from the device, how and where it will be processed, and the app should allow the user to consent to the described data collection and processing before it happens.

Which data is collected by Firebase and when does the data transfer happen?
---------------------------------------------------------------------------

Firebase Crashlyics and Analytics by default start collecting user data as soon as the SDKs are activated on the first launch, even if the app does not record any events itself.

Firebase Crashlytics collects:
* Crashlytics Installation UUIDs
* Crash traces
* Breakpad minidump formatted data (NDK crashes only)

Google Analytics for Firebase collects:
* [Mobile ad IDs](https://support.google.com/adxseller/answer/6274238?hl=en){:target="_blank"}<!-- markup clean_ -->
* [IDFVs](https://developer.apple.com/documentation/uikit/uidevice/1620059-identifierforvendor){:target="_blank"}<!-- markup clean_ -->
* [Firebase installation IDs](https://firebase.google.com/docs/projects/manage-installations){:target="_blank"}<!-- markup clean_ -->
* [Analytics App Instance IDs](https://support.google.com/firebase/answer/6317486?hl=en){:target="_blank"}<!-- markup clean_ -->

Further details on how this data is used by Firebase can be found on [Firebase Privacy page](https://firebase.google.com/support/privacy#data_processing_information){:target="_blank"}<!-- markup clean_ -->.

To avoid data collection without consent, Firebase provides several configurations and APIs that can be used to deactivate and activate Analytics and Crashlytics services.  These services should be disabled by default and only enabled once the user has given consent.

From the user experience perspective, a common pattern is to have a privacy policy screen as part of the app onboarding process. This screen should allow the user to review the privacy policy and provide explicit consent to data collection and processing by switching a toggle or tapping on a button.

How to control when Crashlytics collects user data?
-------------------------------------------------

Crashlytics can be configured to [disable automatic crash reporting](https://firebase.google.com/docs/crashlytics/customize-crash-reports?platform=ios#enable_opt-in_reporting){:target="_blank"}<!-- markup clean_ --> by adding an entry to the project's `Info.plist` file:

```
FirebaseCrashlyticsCollectionEnabled: false
```

After the user gives consent to data collection and processing, the crash reporting can be enabled with

```swift
import FirebaseCrashlytics

// in a method processing the transition from the onboarding screen
Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)
```

How to control when Analytics collects user data?
-------------------------------------------------

Similar to Crashlytics, Analytics can be configured to [disable automatic data collection](https://firebase.google.com/docs/analytics/configure-data-collection?platform=ios#disable_data_collection){:target="_blank"}<!-- markup clean_ --> by adding an entry to the project's `Info.plist` file:

```
FIREBASE_ANALYTICS_COLLECTION_ENABLED: false
```

After the user gives consent to data collection and processing, the analytics service can be enabled with

```swift
import FirebaseAnalytics

// in a method processing the transition from the onboarding screen
Analytics.setAnalyticsCollectionEnabled(true)
```

To allow the users to change their minds, add an option in the app's Settings to withdraw consent or give consent again for data collection and processing.

```swift
// to stop collecting data
Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(false)
Analytics.setAnalyticsCollectionEnabled(false)

// to start collecting data again
Crashlytics.crashlytics().setCrashlyticsCollectionEnabled(true)
Analytics.setAnalyticsCollectionEnabled(true)
```

Conclusion
----------

It is easy to get drowned in the verbose legal documents and technical documentation on user data privacy protection. I hope this article helps you stay above the water and address this boring topic faster so that you can get back to having fun programming. If you have anything to add or you find any inaccuracies in the article, please leave a comment.

Resources
----------
* [https://gdpr4devs.com/](https://gdpr4devs.com/){:target="_blank"}<!-- markup clean_ -->
* [https://gdpr-info.eu/](https://gdpr-info.eu/){:target="_blank"}<!-- markup clean_ -->
* [https://www.smartlook.com/blog/consent-for-mobile-apps-in-2021/](https://www.smartlook.com/blog/consent-for-mobile-apps-in-2021/){:target="_blank"}<!-- markup clean_ -->
* [https://www.iubenda.com/en/help/23040-firebase-cloud-gdpr-how-to-be-compliant](https://www.iubenda.com/en/help/23040-firebase-cloud-gdpr-how-to-be-compliant){:target="_blank"}<!-- markup clean_ -->
* [https://firebase.google.com/support/privacy](https://firebase.google.com/support/privacy){:target="_blank"}<!-- markup clean_ -->
