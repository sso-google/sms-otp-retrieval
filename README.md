# SMS OTP Retrieval

## Introduction

Developers use phone numbers for many aspects of building an application:

* account identifier (especially in emerging markets where email usage is low)
* social graph (based on phone number contact list for example)
* communication channel (i.e. call the user, send text message, etc.)
* anti-abuse signal (phone numbers are limited, may require physical identity verification in some places)
* multi-factor auth (e.g. 2-step verification with SMS OTP)
* account recovery (e.g. look up a forgotten account, as a password reset option)

The challenge is that usage of phone numbers for these purposes typically requires proof that a user currently controls the phone number (phone number verification), and existing verification mechanisms on the web are cumbersome, requiring users to manually input one-time verification codes. Easing this has been a long standing feature request for the web from many of the largest global developers.

There are a variety of ways to verify control over phone numbers, but a randomly generated one-time passcode (OTP) sent by SMS to the number in question is the most common. Presenting this code back to developer’s server proves control of the phone number. In this proposal, we focus on the ability to programmatically obtain one-time codes from SMS as solution to ease the friction and failure points of manual user input of SMS codes, which is prone to error and phishing. Such APIs for SMS retrieval already exist on Android (e.g. [Play Services API](https://developers.google.com/identity/sms-retriever/overview) used by thousands of apps and millions of daily users).

## Goals

Make the most common existing phone number verification flow (SMS OTP) match ease of use in native apps.

## Non-Goals
This proposal does not attempt to move developers off of existing phone number use cases and verification mechanisms, nor does it cover obtaining the phone number itself.

## Scenarios

### 1. As an authentication signal
Online banks use SMS OTP to convey a secret to the user for the purpose of multi-factor authentication (e.g. when signing in to new device), re-auth (at time of a transaction), or account recovery (in the event user were to lose access to other multifactor options).

### 2. To establish reachability
Ride-sharing services often ask user to provide a phone number, and before taking a first ride, check that the user actually owns and is reachable at this number by sending and confirming a one-time code. 

## Proposals

The developer must obtain the phone number via their existing mechanisms (e.g. user form input, autofill, etc.) and send an SMS (e.g. via Twilio) with a one-time code to the number, which they must generate and track on their servers.

In this formulation, the browser will help the site obtain the contents of the SMS or OTP programmatically.  

The following is an early exploration / early baseline of what these APIs could look like. We expect them to change drastically as we learn more about the space.

### Autofill Annotations
This would provide a declarative API for developers: annotate form fields with “one-time-code” to signal to browser where to autofill an SMS OTP. The SMS would have to be structured in such a way that the SMS can be identified and the OTP could be parsed and filled.

```
<input id="single-factor-code-text-field" autocomplete="one-time-code"/>
```

iOS already uses this model ([documentation](https://developer.apple.com/documentation/security/password_autofill/enabling_password_autofill_on_an_html_input_element) - some analysis)

### SMS Retrieval API
In addition, browsers could provide an imperative API to request the contents of an incoming SMS. Here is one possible formulation / shape, based on Android’s [SMS Retriever API](https://developers.google.com/identity/sms-retriever/overview): 

```
  // This is just a draft/example of what a API could look like.
  let retriever = new SmsRetriever({timeout: 60});
  let sms = await retriever.receive();
  extract_otp_and_verify(sms.content);
```

In order to use native SMS retrieval mechanism, the SMS message contents must be formatted appropriately, and the current format is oriented around native apps. For example:

```
  <#> Your ExampleApp verification code is: 123ABC78
  FA+9qCX9VSu
```

Where “FA+9qCX9VSu” is a hashcode derived from the native app package and cert fingerprint.

In one possible formulation, to make SMS available to a browser, the SMS would have to be targeted at the current browser (e.g. use the browser app hash at the end of the SMS), and developer would also have to specify an intended origin. For example:

```
  <#> Your ExampleApp code is: 123ABC78
  https://example.com
  FA+9qCX9VSu
```

For a single or small set of supporting browsers, developers may be able to hardcode or determine the appropriate targeting. This targeting mechanism for native apps may be made more flexible / ergonomic in the future.

The current native SMS Retriever API does not provide any UI using the retrieval process, but the native SMS notifications are still triggered, and the SMS messages will remain in the user’s message history.

In another formulation, if the browser had access to SMS on the device, it could serve as an intermediary between the received SMS and the web apps, and provide a consent user interface before the SMS is shared.

Once the browser has the SMS contents it can return the entire SMS message to the caller. The caller can then extract the OTP from the message (using whatever formatting they chose) and complete the phone number verification flow as if the user had typed the OTP (and if programmatic retrieval fails, user can always simply read the SMS and type OTP manually as currently done).

Note that an advantage to an imperative API is that developer has more flexibility over when the API will be called and the UX around it. For example, unlike a OTP form field, the UI need not be blocked waiting for the user input. Phone number verification would happen in the background while user interacts with other content.

### OTP Retrieval API

Provide a higher level API for obtaining OTP, which could be provided by a variety of transport mechanisms (email, time-based authenticator apps running on the device, not just SMS)

```
  // This is just a draft/example of what a API could look like.
  let otp = await navigator.credentials.get({otp: true});
  verify(otp);
```

## Alternatives Under Consideration

### Heuristic Autofill

In addition to autofill annotations, the browser could also heuristically extract and autofill OTP, with user confirmation and without explicit developer support. 

However, getting access to SMS without coordination from the developer (i.e. without explicit formatting of the SMS or indication that developer is expecting an SMS) will be a challenge, as browser would require ongoing access to SMS. 

Note that iOS provides heuristic-based OTP autofill, but iOS provides browser / keyboard access to SMS; but on Android, browsers may not have or even be able to request indefinite SMS access.

### Phone Number Assertion API

If phone number has already been verified for a given device or user account, browser could return a verifiable assertion of the phone number. 
```
  // This is just a draft/example of what a API could look like.
  let phone = await navigator.credentials.get({phone: true});
  verify(phone);
```

This could be implemented by having developer interact with identity providers (IDPs), which have already verified and are aware of the user’s phone number, and could vouch for this information, in the same way Google Sign-In and similar federated identity flows currently work for email addresses. 

Several identity providers such as Facebook (Account Kit), Truecaller, and others already provide APIs like this verified phone numbers.

## Security and Privacy Considerations

### User Tracking

Phone numbers are an effective stable identifier for a user that enable cross-site and online/offline tracking. Obtaining the phone number is the point at which user is typically (or should be) asked for consent and best educated about the implications of sharing this information, so is not addressed in this proposal.

However, the ability to verify the user’s phone number automatically in the background via an SMS retrieval API is a mechanism by which ongoing presence of the user could be determined, at least on a particular device where the developer already knows the phone number. Existing Android APIs mitigate this by allowing existing SMS notifications and SMS history to continue to be visible to the user, giving them insight that this may be happening (and providing a disincentive for a service to “guess” the user’s phone number, spam the number, and try to detect if this user is present by seeing if retrieval works). In practice, in the Android native app ecosystem, this hasn’t been found this to be a vector for abuse, especially given the cost of sending SMS and the visibility of the attack to users.

### Phishing

SMS OTP are readily phishable and an existing widespread concern. While not making this worse, this proposal attempts to mitigate by avoiding and lessening the occurrence of users to manually enter OTP (so as to be less conditioned to phishing, and/or more conscious of where they enter OTP), and by making the OTP only available via programmatic mechanism to the intended recipient (i.e. by specifying the target origin in the message contents).

## Related APIs

### Credential Management and WebAuthn APIs 

CM API and WebAuthn facilitate alternative forms of authentication. CM API facilitates interaction with password credentials and WebAuthn allows developers to interact with authentication hardware that provides much stronger, phishing resistant, and more usable multi-factor authentication on the web. While better alternatives for authentication, these APIs do not provide any communication mechanism or reputation signals that developers also use phone numbers for, so are not a comprehensive alternative to phone number or SMS OTP. 

### Notifications

Browser notification APIs provide a communication channel to developers, but developers often still prefer or also request a verified phone numbers since it checks that the user is reachable at this number and facilitates voice communication and reasonably real-time two-way communication on practically any time of mobile phone (no dependence on OS or version, pretty much all phones can handle SMS). 

### OAuth etc.

OAuth and similar protocols allow developers to obtain information such as verified phone number from an identity provider (IDP). However, this relies on user having provided, verified, and maintain their phone number with the IDP, and be comfortable using this model to share data (e.g. knowing their usage of 3p services). The IDPs themselves (unless an authoritative provider for a phone number, such as a carrier), need to verify phone number ownership, and typically still use SMS OTP for that purpose. 

### reCAPTCHA

Captcha APIs provide an alternative anti-abuse signals (i.e. that user is not a bot) that developer sometime rely on phone numbers for (in that phone numbers are often limited and require human involvement to procure, and as such as hard to produce at scale).

### Payment APIs

Phone numbers are sometimes used for carrier billing schemes. Payment APIs offer an alternative as well as signal of user quality (having a payment instrument often involves identity verification and ability / history of being able to pay).

## Frequently Asked Questions

### Why are we telling developers to use phone numbers? They can be hijacked, change ownership, used to track users, etc.
We aren’t telling developers to use phone number; they are already using them. Despite problems, phone numbers are useful for various purposes (see intro). In the absence of alternatives to achieve these useful things, developers will continue to use phone numbers and users will struggle with them. This proposal makes the web more usable in the short term (in particular matching functionality that already existings natively), but in the long run, we hope to move the ecosystem off of phone numbers as alternatives for the need functionality become available. Given the time-scale of such a transition, action in the short-term to reach experience parity seems necessary.

### Why is this using SMS OTP as a verification mechanism? SMS OTP are slow, expensive, phishable, etc.
SMS is the most common existing verification mechanism; this proposal aims to streamline flows that already exist by mitigating the need for user involvement/action. Although this does not address the more fundamental problems, it may make some situations better (e.g. users don’t handle OTP manually, hence less conditioned to be phished), but starting to move developers to a programmatic model would be a potential stepping stone to better mechanisms.

### Is this implementable on iOS?
Yes, iOS already uses a declarative model (form annotation “[one-time-code](https://developer.apple.com/documentation/security/password_autofill/enabling_password_autofill_on_an_html_input_element)”) and heuristically extracts and suggests OTP from SMS. They could implement an imperative API or support other alternatives like facilitating interaction with identity providers if desired.

### Is this implementable on desktop?
Yes, browser need not look for SMS on the local device. For example, if Chromesync is enabled across desktop and mobile devices, Chrome could retrieve on one device and return the SMS content on another. Or locally, a desktop instance of a browser could use Bluetooth to talk to a nearby phone and have an instance of the browser on the mobile device retrieve and return the SMS.

### Why can’t we use a model like having developer tell us the OTP and let them know if there is a match?
The developer needs something that can be verified on their server, a yes/no response is not sufficient. Claiming that the phone number is active on the device would require a response signed in some way to make it verifiable. This could be done in a number of ways, but is a major departure from the current OTP model and necessitate major changes from developers (i.e. change their backend server logic to process these claims; whereas just returning the SMS contents with the OTP means primarily only a frontend change an minimal backend work ... perhaps only changing the message template format, rather than security-sensitive backend logic). However, the longer-term intent with more comprehensive Identity APIs is to provide verifiable assertions of phone ownership, which would be compelling for developers to adopt and change their system to accept if it meant avoiding the need for sending SMS.
