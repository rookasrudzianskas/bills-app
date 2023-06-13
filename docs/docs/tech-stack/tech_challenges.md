# Technical challenges

Most problems comes from about how to keep functionality usable **offline** and **without the need of account registration**. 

With iterative development (I work on Billy as the sole developer, only whenever I have the free time and mood), sometimes it has led to a lot of rethinking and rewriting required when it comes to the strategy of syncing/invalidating cache and handling notifications.

Below are some significant challenges that I have met during the iterative development process. For how I versioned the releases below, please refer to the ROADMAP.

## 0.0.2 (Internal track)

Some syncing issues have happened due to the lack of check on **debouncing** and also inaccurate comparison of `id` and `tempID`. Fixes have been implemented in [PR#10](https://github.com/lyqht/Billy/pull/10) if you are keen on the test user's report and how they are addressed.

## 0.1 (Alpha Track)

This release is identical to 0.0.2, after the fixes.

In **0.1**, users can create bills, complete upcoming and archive missed bills. They also have access to the analytics tab to view their expenses by month. They can create an account to sync their bills to the cloud.

### Logic for caching and syncing
- If they have yet to create an account, no sync-ing to cloud will be performed.
- Without an account, bills created will have a `tempID`, and the notifications that will be created will have metadata containing a `billId` that corresponds to the `tempID`.
- After creating an account, these local bills will be synced, and notifications created for them will have their `billId` updated to the new `id` generated by the cloud.
- Now that the user has an account, everytime they add a bill, **it will be first cached and then synced to the cloud**. This results in funny problematic behavior that shows up while working on 0.2 as the bill has both a server generated ID and a `tempID`.

## 0.2 (Alpha Track)

The Edit Bill feature is introduced in **0.2**, where users can edit their bill details like payee, category, and also notifications. They will be given the same Bill Form Screen to edit their bills. They can only update upcoming bills, and not missed bills.

### Updated Logic for caching and syncing
- Because of how synced bills could still have `tempID` due to implementation in 0.1, _editing bills create new bills_ instead.
  - This has been resolved by rewriting the implementations so that creating & updating a bill only updates the local cache. 
  - Then only when they sync changes to the cloud, then the local cache will be entirely be replaced by the server state.
  - This ensures that there will be no bills with a `tempID` afterwards.

### Addressing problems with editing notifications

#### Problem 1: Currently, to ensure offline-first functionality, no notifications are not stored on the cloud. **It is a concept specific only to the user's current device**.

- With the edit bill feature, notifications can be modified. That means that we need to update the timestamps of the notifications that we have already set in the device.
- However, on the bill form screen, users see relative reminder dates `({timeUnit, value})` rather than absolute dates. _e.g. 1 week after deadline vs 15 June 2022_. So while we could retrieve the timestamp of the existing pending notifications, we cannot use this timestamp directly.
- Also, it will be pretty complicated to reverse engineer the relative reminder date in terms of the value & timeUnit, if the user also changes the deadline while editing the bill.
- Hence, **the metadata of notifications created has been updated** to include `timeUnit, value, deadline`. That way, the app can still retrieve and show existing notifications' relative reminder dates correctly even when they update the bill deadline.
- If the user is editing a bill, when the user submits the updated bill, the device will delete all pending notifications for this bill, and recreate them with the updated deadline and relative reminder dates.

#### Problem 2: Number of notifications do not show correctly after editing bill

Currently, the number of pending notifications is shown on Upcoming Bills screen and Bill Details Screen.

- In Bill Details Screen, there is a `useEffect` to retrieve the relative reminder dates on componentDidMount phase. When the user is navigated back to this screen after editing the bill, the `useEffect` is not triggered again so the number of notifications is not updated. 
  - To fix this, I change this to `useFocusEffect` instead so the notifications will be fetched again.
- In Upcoming Bills Screen, the number of notifs rendered is displayed based on the context. In previous implementation, the app waits for the bill to be updated in the cache before creating the notifications (which sounds like a logical sequence). However, in doing so, **the cache on value change listener callback is invoked** and checks that this bill still has the same number of notifications since the notifications have yet to be created.
  - At the same time, the order of creation cannot be swapped since the `billID` is required in the metadata of the notifications. This `billID` could be absent and is only generated if user is adding bill.
  - While we could create the notifs with no metadata, create the bill and then update the notifs with metadata, with the current Notifee API, what this will do is to recreate the notifs. **The impact is rather unclear**, but it certainly doesn't feel clean to me.
  - Hence, to fix this, again we use `useFocusEffect`, and we resync the local context state for reminders.

#### Problem 3: If the user logins on another device with the same account, the new device must be able to create notifications that they have already set for bills that they have previously created.

[WIP]