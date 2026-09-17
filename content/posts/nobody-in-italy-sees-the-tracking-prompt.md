---
title: "Nobody in Italy sees the tracking prompt"
date: 2026-09-17
draft: false
ai: true
image: images/posts/nobody-in-italy-sees-the-tracking-prompt.png
tags: ["iOS", "App Tracking Transparency", "Privacy", "Apple", "EU"]
description: "Since iOS 27.0, the ATT permission prompt never appears on devices signed in with an Italian Apple Account. Not in my app: in any app. It looks like a piece of the iOS 27.2 EU changes landed three releases early."
---

I spent an evening convinced I had broken the tracking permission request. The prompt had stopped appearing on my phone. Same build that worked a week earlier, no changes, and `ATTrackingManager.requestTrackingAuthorization()` coming back `.notDetermined` in seven milliseconds with no prompt and no error.

Then I opened Instagram, which had never asked me either.

## What the system actually says

The app only ever sees the outcome, so the app is the wrong place to look. The decision is taken somewhere else, by a daemon called `tccd`.

TCC stands for Transparency, Consent and Control. It is the part of iOS that owns every privacy permission you have ever tapped through: camera, microphone, photos, location, contacts. Your app does not draw those alerts and never has. It asks `tccd`, `tccd` decides whether an alert should be shown at all, a system process draws it if the answer is yes, and your app is eventually handed a result. App Tracking Transparency is one of those permissions like all the others, filed under the service name `kTCCServiceUserTracking`.

So the interesting log is not the app's. It is the device's, and `tccd` says out loud what it did:

```bash
log collect --device --last 5m --output att.logarchive
```

```
[ATTrackingManager] requestTrackingAuthorizationWithCompletionHandler API call invoked.
[ATTrackingManager] Performing TCC Access Request.
tccd  AUTHREQ_CTX: function=TCCAccessRequest, service=kTCCServiceUserTracking
tccd  #AuthorizationPromptServiceClient prompt is NOT eligibleToShow for kTCCServiceUserTracking <private>
tccd  AUTHREQ_RESULT: authValue=1, authReason=0, error=(null)
[ATTrackingManager] requestTrackingAuthorizationWithCompletionHandler returning - ATT not determined.
```

`NOT eligibleToShow`. Read the sequence: the request arrives, TCC evaluates it, TCC declines to present anything, and the framework hands back `.notDetermined`. No error anywhere, because from the system's point of view nothing went wrong. It was asked whether to show an alert and it answered no.

That is the part with consequences. "No prompt appeared" is a legitimate outcome of this API, not a failure state you can code around, and it is returned as the exact same value you get from a user who has simply not answered yet. Those two situations are indistinguishable to your app, by design.

The reason for the refusal is on that same line, redacted as `<private>`. You can unredact it with a configuration profile, but the generic one lifts redaction for the whole system, every app's private data in the clear, on a phone with your own life on it. I passed.

## The one variable that matters

Somebody on the [Apple Developer Forums](https://developer.apple.com/forums/thread/845135) had already run the experiment I could not:

> Same device, same physical location, same build, same test project, no other change: signed in with an Italian Apple Account the prompt is never presented and the status stays `notDetermined`, while signed in with a Spanish Apple Account it is presented correctly.

Not the device region. Not where you are standing. The country of the Apple Account signed into the phone. It is filed as [FB24689594](https://developer.apple.com/forums/thread/845135), and per that thread it starts working again in iOS 27.2 beta 1. (Feedback Assistant reports are only visible to whoever filed them, so the thread is the citable source.)

Which is a very specific version number to fix a very specific bug in.

## What Apple announced for 27.2

On [September 16](https://developer.apple.com/news/?id=idsft9ai) Apple published the changes coming to App Tracking Transparency in the European Union, as part of agreements with European competition authorities. From iOS 27.2 and iPadOS 27.2, developers get an alternative version of the system prompt: different formatting, different language, and an optional "Additional Information" button that can surface more detail about what you intend to do with the data. EU developers will also be allowed to ask again one year after a user's previous answer, whether that answer was yes or no. The rules about *when* you need permission do not change. The details are on Apple's [user privacy and data use](https://developer.apple.com/app-store/user-privacy-and-data-use/) page.

Then there is the sentence that explains my evening:

> Due to legal requirements, only the alternative version of the system prompt is available for apps distributed in Germany, France, Italy, Poland, and Romania.

Five countries where the old alert is not an option at all. Italy is one of them.

## A gate without a door

Put the two halves together and you get a theory that fits everything I saw.

In 27.2, for accounts in those five countries, the classic prompt must not be shown. Something has to enforce that. It looks like the enforcement shipped in 27.0 and the replacement screen did not, so on 27.0 and 27.1 the system correctly refuses to present the old alert to an Italian account and has nothing to present instead. A gate with no door behind it. `NOT eligibleToShow` is exactly what that would look like from the outside.

I want to be clear that this is a reading, not a finding. I cannot see Apple's code and neither can you. What is verifiable: the refusal happens, the account country is the only variable that changes it, the five countries in Apple's announcement include Italy, and the fix version is the same release that introduces the new prompt.

It also explains how this shipped at all. To hit it you need a device on 27.0 signed in with an account registered in Germany, France, Italy, Poland or Romania. If your test devices are American, every prompt works perfectly, forever.

## Everyone is broken, not just you

This is the part worth repeating, because it is the part that will save you the evening I lost: it is not your app.

Facebook, Instagram, and every other app that asks for tracking permission behave identically on an affected account. Nothing appears. If you have an Italian phone on 27.0 or 27.1 in your hands right now, open a free app you have never launched and watch it not ask you.

So if a tester files "the tracking prompt is gone", resist the urge to go read your own code. Check whether any app on that phone can show it.

## Living with it until 27.2

A few things I would do regardless of how this resolves.

Do not let your UI depend on the prompt appearing. If you have an onboarding step whose only action is requesting tracking permission, and it refuses to advance until the status changes, that screen becomes a dead end the moment the system stops presenting anything: a button that cannot succeed, nothing to answer, no way forward. Treat "still not determined" as a perfectly normal result and let people move on.

Give the request a second way in, a row in your settings screen or similar. Anyone who skipped the onboarding step, or was never prompted at all, currently has no path back to it.

Do not trust the simulator here. It kept showing me the prompt the whole time, because it is not signed in with an Italian Apple Account. It implements the API. It does not reproduce the policy, and the policy is the entire bug.

And App Review is less scary than it feels. Reviewers are not using Italian Apple Accounts, so they will see your prompt. Even if they did not, iOS zeroes out the IDFA without ATT authorisation, so nothing is being tracked and there is nothing to be non-compliant about. [FB24689594](https://developer.apple.com/forums/thread/845135) is there to cite if it ever comes up.

The EU changes themselves are worth reading properly before 27.2 lands, especially if you ship in those five countries, because the prompt you have been designing around for five years is about to be a different prompt.

**Sources:** [Updates to App Tracking Transparency in the European Union](https://developer.apple.com/news/?id=idsft9ai) (Apple, 16 September 2026), [User Privacy and Data Use](https://developer.apple.com/app-store/user-privacy-and-data-use/) (Apple), [Developer Forums thread 845135](https://developer.apple.com/forums/thread/845135), [9to5Mac coverage](https://9to5mac.com/2026/09/16/ios-27-2-lets-developers-use-an-alternative-app-tracking-transparency-prompt-in-the-eu/).
