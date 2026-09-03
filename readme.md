# MagnetAd SDK for Unity — Publisher Guide

This is a step-by-step guide for installing and using the **MagnetAd** package in Unity projects, to show **full-screen (Interstitial)** ads on **Android** — either plain or **rewarded video**.

The pattern is the same as other well known SDKs (AdMob / AppLovin): call `Initialize` once, then create one `InterstitialAd` object per placement, call `RequestAd`, and call `Show` when the ad is ready.

> You choose whether a placement is plain or rewarded when you create it in the publisher panel. Whether a given ad arrives as an image or a video is decided per request, and your integration code is the same for both.

---

## Contents

1. [Overview](#1-overview)
2. [Requirements](#2-requirements)
3. [Step 1 — Install the package](#step-1--install-the-package)
4. [Step 2 — Gradle dependencies (the most important step)](#step-2--gradle-dependencies)
5. [Step 3 — Project settings (Android)](#step-3--project-settings-android)
6. [Step 4 — Get your ids](#step-4--get-your-ids)
7. [Step 5 — Write the integration code](#step-5--write-the-integration-code)
8. [Step 6 — Ad lifecycle and events](#step-6--ad-lifecycle-and-events)
9. [Step 7 — Video ads, sound and rewards](#step-7--video-ads-sound-and-rewards)
10. [Testing on a device](#testing-on-a-device)
11. [Full API reference](#full-api-reference)
12. [Error codes](#error-codes)
13. [Troubleshooting](#troubleshooting)
14. [Limits](#limits)

---

## 1. Overview

- Ad type: **full-screen (Interstitial)**, image or video. A rewarded video pays out once the user watches it through ([Step 7](#step-7--video-ads-sound-and-rewards)).
- Platform: **Android** only.
- A few standard Gradle dependencies must be added to your build ([Step 2](#step-2--gradle-dependencies)).
- No dangerous permissions and no permission dialog for the user. The `INTERNET` permission and the other required settings are merged into your app manifest automatically — **you do not need to edit the manifest by hand.**
- All methods are **non-blocking** and never stop your game loop. No method throws an exception into your code; errors are always reported through the event channel.
- All events run on the **Unity main thread**, so you can work with UI and Unity objects directly inside them.
- Ads are only shown in an **Android build**. Inside the Editor in Play mode the calls do nothing and fail through the normal error channel with a warning in the Console — build to a device for real testing.

---

## 2. Requirements

| Item | Value |
|------|-------|
| Unity version | **2021.3 LTS** to **Unity 6** |
| Minimum Android API (minSdk) | **21** |
| Scripting Backend | **IL2CPP** |
| Target Architectures | must include **ARM64** |
| Build system | **Gradle** (Unity default) |

These permissions and settings are added to your app automatically, with no manual work:

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="com.google.android.gms.permission.AD_ID" />
<queries> ... Myket / Cafe Bazaar / Google Play ... </queries>
```

None of these permissions are **dangerous**, and no permission dialog is shown to the user.

> **`AD_ID` and the Data safety form:** the `com.google.android.gms.permission.AD_ID` permission is added to your manifest. If you publish your app on **Google Play**, you must declare this in the *Data safety* form. If your app is aimed at children and must not have this permission, you can remove it in your own manifest:
>
> ```xml
> <uses-permission android:name="com.google.android.gms.permission.AD_ID" tools:node="remove" />
> ```
>
> The SDK still works in that case. Just note that you must still add the `play-services-ads-identifier` dependency ([Step 2](#step-2--gradle-dependencies)), or the app will crash.

> **ProGuard/R8:** if you turned Minify on in Publishing Settings, no manual rules are needed.

---

## Step 1 — Install the package

Choose one of the two methods below.

### Method A) Install with a Git URL (recommended)

In Unity:
`Window → Package Manager → + button → Add package from git URL`
and enter this address:

```
https://github.com/MagnetAds/magad-unity-package.git#v1.1.0
```

> - The tag at the end of the address (`#v1.1.0`) locks your project to this version. To move to the next version, just change the tag number.
> - Requirement: **Git** must be installed on your machine and you need network access to GitHub.

### Method B) Install from a .unitypackage file or by copying the folder

Download the `.unitypackage` for the version you want from the Releases page and import it (`Assets → Import Package → Custom Package…`), or copy the whole `MagnetAd` folder into your project's `Assets`.

Releases page:

```
https://github.com/MagnetAds/magad-unity-package/releases
```

> **Note:** the package namespace is `MagnetAdSDK` and the main class is `MagnetAd`. Always write `using MagnetAdSDK;` in your code.

---

## Step 2 — Gradle dependencies

**This is the most important setup step.** The standard libraries the SDK needs at runtime must be downloaded by Gradle. If you skip this step, the app will crash on `Initialize` with a `NoClassDefFoundError`.

There are two methods; do **only one** of them.

### Method A) External Dependency Manager (EDM4U)

If you already have [EDM4U](https://github.com/googlesamples/unity-jar-resolver) in your project (it comes with most Google/Firebase SDKs), no manual work is needed: the `Editor/MagnetAdDependencies.xml` file inside the package is found and resolved automatically.

### Method B) Add them by hand to mainTemplate.gradle

1. In `Project Settings → Player → Android → Publishing Settings`, turn on **Custom Main Gradle Template** so that `Assets/Plugins/Android/mainTemplate.gradle` is created.
2. Add these lines inside the `dependencies` block, before `**DEPS**`:

```gradle
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'org.jetbrains.kotlin:kotlin-stdlib:2.0.21'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1'
    implementation 'androidx.core:core-ktx:1.10.1'
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.1'
    implementation 'com.google.android.gms:play-services-ads-identifier:18.0.1'
**DEPS**}
```

> - If another SDK (Firebase, AdMob and so on) brings the same libraries with a **different version**, that is fine — Gradle picks the higher one. The only real problem is when another plugin puts the same libraries in `Plugins/Android` as **raw jar/aar files** (the Duplicate class error — see [Troubleshooting](#troubleshooting)).
> - To download these libraries, the build machine needs access to `google()`/`mavenCentral()` or a mirror such as `maven.myket.ir`.
> - **`play-services-ads-identifier` is not optional.** Without it, the app crashes on the first ad request with a `NoClassDefFoundError`. Because access to Google's repository is limited in Iran, get this library and the one it depends on (`play-services-basement`) from the `https://maven.myket.ir/` mirror, and put that mirror **before** `google()`.
> - **Do not raise these versions.** They are chosen so the build also works with **Target API 33**. For example `androidx.core:core-ktx:1.12.0` and newer force the build to compile against **API 34**, and on Unity 2021.3 that fails with `requires libraries and applications that depend on it to compile against version 34`. If your project's Target API is 34 or higher, raising the versions is fine.

---

## Step 3 — Project settings (Android)

### 3-1) Switch to the Android platform

`File → Build Settings → Android → Switch Platform`

### 3-2) Manifest

You do not need to add anything to the manifest by hand — all the required settings are merged into the build automatically.

### 3-3) Input and EventSystem

The ad is shown natively and has **no dependency at all** on uGUI, the EventSystem, or Unity's Input settings.

---

## Step 4 — Get your ids

You get two ids from the MagnetAd publisher panel:

- **App Id** (your property id) — used in the `MagnetAd.Initialize` method.
- **Placement Id** (the ad placement id) — used in the `InterstitialAd` constructor.

Each placement is created in the panel as either plain or rewarded, so you already know which kind the id in your code refers to. Use a rewarded placement only where you actually hand out a prize.

> These ids are sent as strings and the server is what validates them.

---

## Step 5 — Write the integration code

Create a script (for example `AdsManager.cs`) and put it on a GameObject in your scene. The pattern is: **create one `InterstitialAd` object per placement, subscribe to that object's events, call `RequestAd` and then `Show`.**

```csharp
using UnityEngine;
using MagnetAdSDK;

public class AdsManager : MonoBehaviour
{
    [SerializeField] private string appId       = "YOUR-APP-ID";
    [SerializeField] private string placementId = "YOUR-PLACEMENT-ID";

    private InterstitialAd _interstitial;

    void Start()
    {
        MagnetAd.Initialize(appId, success =>
        {
            if (!success) { Debug.LogWarning("MagnetAd init failed"); return; }
            LoadAd();
        });
    }

    void LoadAd()
    {
        if (_interstitial == null)
        {
            _interstitial = new InterstitialAd(placementId);
            _interstitial.OnAdLoaded       += () => Debug.Log("Ad ready.");
            _interstitial.OnAdFailedToLoad += err => Debug.LogWarning($"Load failed: {err.Code}");
            _interstitial.OnAdShown        += () => Time.timeScale = 0f;
            _interstitial.OnAdClicked      += () => Debug.Log("Ad clicked");
            _interstitial.OnRewarded       += () => GrantCoins(50);   // rewarded placements only
            _interstitial.OnAdDismissed    += () =>
            {
                Time.timeScale = 1f;
                LoadAd();
            };
            _interstitial.OnAdFailedToShow += err =>
            {
                Time.timeScale = 1f;
                Debug.LogWarning($"Show failed: {err.Code}");
                LoadAd();
            };
        }

        _interstitial.RequestAd();
    }

    public void ShowAdNow()
    {
        if (_interstitial != null && _interstitial.IsLoaded)
            _interstitial.Show();
        else
            LoadAd();
    }

    void OnDestroy()
    {
        _interstitial?.Destroy();
        _interstitial = null;
    }
}
```

A short note on each part:

- `Start` — calls `Initialize` once, and preloads the first ad after it succeeds.
- `LoadAd` — creates the ad object only once (one is enough for the whole session), hooks up the events, and calls `RequestAd`. The game is paused in `OnAdShown` and resumed in `OnAdDismissed` and `OnAdFailedToShow`.
- `ShowAdNow` — connect this method to the right event in your game (the end of a level, for example). If no ad is ready, it requests one.
- `OnDestroy` — releases the resources.

### Key points

- **Events belong to the object:** you subscribe on each `InterstitialAd` instance separately, and `Destroy` on that instance releases everything.
- **Every show uses up the ad:** after each successful `Show` (the `OnAdDismissed` event), call `RequestAd` again for the next one.
- **Do not forget `Destroy`:** call `Destroy()` in the `OnDestroy` of whatever owns the ad, so the resources are released. The class is also `IDisposable` (usable with `using`). If you forget, the SDK prints a warning in the Console when the object is collected.
- **If an ad is on screen when you call `Destroy`:** the `OnAdDismissed` event is still raised. So if you paused the game in `OnAdShown`, the game still comes back even if the scene changes in the middle of an ad.
- **Preloading:** request the ad early (after a successful `Initialize`, or after the previous ad closes) so it is ready the moment you need it.
- **Clicking the ad:** when the user clicks, the market page (Myket / Cafe Bazaar / Google Play) or the browser opens and your game goes to the background. The ad closes right away, so you get `OnAdDismissed` after `OnAdClicked`.

---

## Step 6 — Ad lifecycle and events

The normal order of events:

```
MagnetAd.Initialize(appId, onComplete)
        │
        └─►  onComplete(true/false)          + MagnetAd.OnInitializationComplete event

new InterstitialAd(placementId)  →  RequestAd()
        │
        ├─►  OnAdLoading                     (the request started)
        ├─►  OnAdLoaded                      ✅ ready (IsLoaded = true)
        └─►  OnAdFailedToLoad(err)           ❌

Show()
        │
        ├─►  OnAdShown                       (the ad is on screen)
        │         ├─►  OnAdClicked  ─────►  OnAdDismissed
        │         │                          (the click opens the market/browser and closes the ad)
        │         ├─►  OnRewarded            (rewarded video only: watched through)
        │         └─►  OnAdDismissed         (the user closed the ad with the ✕ button)
        │
        └─►  OnAdFailedToShow(err)           (AD_NOT_READY / AD_EXPIRED / VIDEO_TIMEOUT / …)
```

| Event | Type | Description |
|-------|------|-------------|
| `MagnetAd.OnInitializationComplete` | `Action<bool>` | Raised with the result when `Initialize` finishes, in addition to the method's own callback. If you subscribe after `Initialize` has already finished, you get the result on the next frame, so the order of your calls does not matter. Handlers are released after the event is raised. |
| `OnAdLoading` | `Action` | The ad request started. |
| `OnAdLoaded` | `Action` | The ad and its image are ready; `IsLoaded` becomes true. |
| `OnAdFailedToLoad` | `Action<AdError>` | The ad request failed (network / server / image). |
| `OnAdShown` | `Action` | The ad was shown. |
| `OnAdClicked` | `Action` | The user tapped the ad image (once per ad). `OnAdDismissed` follows right after it. |
| `OnAdDismissed` | `Action` | The ad closed (with the ✕ button or after a click). |
| `OnAdFailedToShow` | `Action<AdError>` | The ad could not be shown (not ready, expired, the image did not render, and so on). |
| `OnRewarded` | `Action` | Rewarded video only: the user watched the video through. Never raised for an image ad. See [Step 7](#step-7--video-ads-sound-and-rewards). |

> While the ad is on screen, the close button (✕) first shows a **countdown of a few seconds** and then becomes active. The hardware Back button is disabled during the ad.

> **Two important points about the order of events:**
> - **A click closes the ad.** `OnAdDismissed` always comes after `OnAdClicked`, so if you pause the game in `OnAdShown`, resuming it in `OnAdDismissed` works for both paths (closing by hand and clicking).
> - **If the ad image does not render after it opens**, the ad closes and only `OnAdFailedToShow` with the code `ASSET_LOAD_FAILED` is raised — `OnAdDismissed` does not come in this case. So if you paused the game, make sure you also resume it in `OnAdFailedToShow` (the sample code does this).

---

## Step 7 — Video ads, sound and rewards

An ad may arrive as an image or as a video. Your code is the same for both — the same `RequestAd` and the same `Show`. Only two things differ: **sound** and **rewards**.

### 7-1) Video sound

Most games have their own sound setting, and a video ad that ignores it is a bad surprise. Two properties on `MagnetAd` handle this:

```csharp
MagnetAd.VideoVolume = 0.5f;   // 0f (silent) to 1f (loudest), default 1f
MagnetAd.VideoMuted  = true;   // silence without losing the VideoVolume value
```

- Set either **before** `Initialize` so the first video already starts at the right level, or **any time after** — including while a video ad is on screen; the change applies immediately.
- `VideoMuted` does not clear `VideoVolume`. Setting it back to `false` restores the same level, which makes it the right choice for a mute button.
- Values outside `0f..1f` are clamped.

### 7-2) Telling a rewarded placement apart

You picked the type when you created the placement, so the simplest approach is to write your code around what you already know: attach a reward handler on a rewarded placement, and skip it on a plain one.

If you also want to confirm it at runtime, `PlacementType` tells you after `OnAdLoaded`:

```csharp
_interstitial.OnAdLoaded += () =>
{
    if (_interstitial.PlacementType == PlacementType.REWARDED)
        ShowRewardPromiseDialog();   // "watch this through and get 50 coins"
};
```

`PlacementType` is `UNKNOWN` until an ad is loaded, and `INTERSTITIAL` or `REWARDED` after that.

### 7-3) Receiving the reward

When the user watches the video **through**, `OnRewarded` is raised on that ad object:

```csharp
_interstitial.OnRewarded += () => GrantCoins(50);
```

> Subscribe to it **with the rest of the events, before `RequestAd`**. If no handler is attached at that moment the reward is lost — the SDK does not hold it for later.

> The reward is announced the moment the view finishes and does not wait for the server, so a dropped connection cannot cost the user their prize. If granting the reward matters to you commercially, record it on your own server as well.

### 7-4) Video playback notes

- A rewarded video usually cannot be closed before it ends, so the user cannot bail out early and still ask for the prize.
- Screen brightness during playback follows the device's system brightness.
- If playback stalls and nothing comes back, the SDK closes the ad itself and raises `OnAdFailedToShow` with the code `VIDEO_TIMEOUT`, so your game is never left waiting.

---

## Testing on a device

The SDK can only be tested on an **Android device**: make an Android build and check the whole path — `Initialize`, the ad request, and the show — on the device.

- In Play mode inside the Editor there is no network connection at all; the message `[MagnetAd] Ads are only available in Android builds.` is printed in the Console, `Initialize` returns `false`, and `RequestAd`/`Show` fail through the error events.
- Unity-side messages are prefixed with `[MagnetAd]` and can be seen in the Console and in logcat.

> `MagnetAd.GetSDKVersion()` returns `"unsupported"` in the Editor.

---

## Full API reference

```csharp
namespace MagnetAdSDK
{
    public static class MagnetAd
    {
        public const string Version;                       // package version
        public const long DefaultInitTimeoutMs;            // 30000
        public static bool IsInitialized();
        public static string GetSDKVersion();              // "unsupported" in the Editor
        public static event Action<bool> OnInitializationComplete;

        public static void Initialize(string appId, Action<bool> onComplete = null);
        public static void Initialize(string appId, bool debugMode,
                                      Action<bool> onComplete = null);
        public static void Initialize(string appId, bool debugMode, long timeoutMs,
                                      Action<bool> onComplete = null);

        public static float VideoVolume { get; set; }      // 0f..1f, default 1f
        public static bool VideoMuted { get; set; }

        public static void ClearCache();                   // clears the cached ads and images
        public static void Shutdown();                     // called automatically when the app quits
    }

    public sealed class InterstitialAd : IDisposable
    {
        public const long DefaultRequestTimeoutMs;         // 30000

        public InterstitialAd(string placementId);         // never throws
        public string PlacementId { get; }
        public PlacementType PlacementType { get; }        // UNKNOWN until an ad is loaded
        public AdState State { get; }                      // IDLE / LOADING / LOADED / SHOWING / DESTROYED
        public bool IsLoaded { get; }                      // an ad is ready to show
        public bool IsLoading { get; }                     // a request is in progress

        public event Action OnAdLoading;
        public event Action OnAdLoaded;
        public event Action<AdError> OnAdFailedToLoad;
        public event Action OnAdShown;
        public event Action OnAdClicked;
        public event Action OnAdDismissed;
        public event Action<AdError> OnAdFailedToShow;
        public event Action OnRewarded;                    // rewarded video only

        public void RequestAd();                  // requests and preloads an ad
        public void RequestAd(long timeoutMs);    // same, with your own timeout
        public void Show();                       // shows the loaded ad
        public void Destroy();                    // releases the ad; safe to call more than once
    }

    public sealed class AdError
    {
        public string Code { get; }      // one of the AdErrorCode values
        public string Message { get; }
    }

    public enum AdState { IDLE, LOADING, LOADED, SHOWING, DESTROYED }

    public enum PlacementType { UNKNOWN, INTERSTITIAL, REWARDED }

    public static class AdErrorCode { /* the constants listed below */ }
}
```

| Method | Description |
|--------|-------------|
| `MagnetAd.Initialize(appId, cb)` | Non-blocking. Talks to the server in the background and returns the result as a `bool`. The callback is always called and never left unanswered. To try again, just call it again. |
| `RequestAd()` | Non-blocking. The result always comes through `OnAdLoaded` or `OnAdFailedToLoad`. If a request is already running, a second call is ignored (no second request goes to the server) and a warning is printed in the Console. |
| `Show()` | Shows the preloaded ad. If no ad is ready, `OnAdFailedToShow(AD_NOT_READY)` is raised. |
| `Destroy()` | Releases the ad resources. The instance cannot be used after that. If an ad is on screen when you call it, `OnAdDismissed` is also raised so a paused game always comes back. |
| `MagnetAd.VideoVolume` | Video-ad volume, `0f` to `1f` (default `1f`). Settable before `Initialize` or at any time after, including mid-playback. Values outside the range are clamped. |
| `MagnetAd.VideoMuted` | Silences video ads without clearing `VideoVolume`, so unmuting returns to the same level. |
| `PlacementType` | The kind of the loaded ad. `UNKNOWN` before `OnAdLoaded`. See [Step 7](#step-7--video-ads-sound-and-rewards). |

> **About `IsLoaded`:** an expired ad reports `false` quickly, so this value does not go stale. Every successful show uses up the ad and `IsLoaded` becomes `false` again; you must call `RequestAd()` for the next one. Read this property from the main thread only.

> **Calling from other threads:** you can call any public method from any thread (inside Firebase's `ContinueWith`, for example). The SDK moves the work to the main thread itself. All events are always raised on the Unity main thread.

---

## Error codes

Always compare `AdError.Code` against the constants in the `AdErrorCode` class, not against a string you write by hand.

| C# constant | Meaning |
|-------------|---------|
| `AdErrorCode.NOT_INITIALIZED` | RequestAd/Show was called before a successful `Initialize`. |
| `AdErrorCode.NETWORK_ERROR` | The server could not be reached / a network error. |
| `AdErrorCode.SERVER_ERROR` | A server-side error (5xx). |
| `AdErrorCode.INVALID_RESPONSE` | The server response could not be processed. |
| `AdErrorCode.ASSET_LOAD_FAILED` | Downloading or showing the ad image failed. |
| `AdErrorCode.NO_FILL` | There is no suitable ad for this request right now. This is not a fault; request again a little later. |
| `AdErrorCode.AD_NOT_READY` | `Show()` was called with no loaded ad. |
| `AdErrorCode.AD_EXPIRED` | The cached ad has expired; call RequestAd again. |
| `AdErrorCode.ACTIVITY_UNAVAILABLE` | The app screen was not available and the ad could not be shown. |
| `AdErrorCode.SHOW_FAILED` | An unknown error while showing the ad. |
| `AdErrorCode.TIMEOUT` | Getting the ad took too long. |
| `AdErrorCode.UNEXPECTED_AD_TYPE` | The server sent an ad kind this version of the SDK does not know. |
| `AdErrorCode.VIDEO_PLAYBACK_ERROR` | Video playback stopped with an error. |
| `AdErrorCode.VIDEO_SERVER_DIED` | The operating system's video player died (usually a device-side problem). |
| `AdErrorCode.VIDEO_TIMEOUT` | The video never started, or stalled and stopped responding; the SDK closed the ad. |
| `AdErrorCode.UNKNOWN` | Any other error. |

---

## Troubleshooting

| Symptom | Cause and fix |
|---------|---------------|
| The error `The type or namespace name 'MagnetAdSDK' could not be found` | Either the package is not fully installed (fix the other error in the Console first), or you forgot `using MagnetAdSDK;`. |
| Crash on `Initialize` on the device with `NoClassDefFoundError: kotlinx/coroutines/...` (or okhttp/serialization) | The Gradle dependencies were not added — do [Step 2](#step-2--gradle-dependencies). |
| Crash on the first ad request with `NoClassDefFoundError: com/google/android/gms/ads/identifier/AdvertisingIdClient` | The `play-services-ads-identifier` dependency was not added — do [Step 2](#step-2--gradle-dependencies). |
| Build error `Duplicate class kotlin...` or `Duplicate class okhttp3...` | Another plugin put the same libraries in `Plugins/Android` as **raw jar/aar files**. Delete the duplicated raw file so the Maven version is used instead. |
| Build error `Duplicate class kotlin.collections.jdk8...` (or any class from `kotlin-stdlib-jdk7` / `kotlin-stdlib-jdk8`) | Deleting a raw file does not fix this one. Since Kotlin 1.8, `kotlin-stdlib-jdk7` and `kotlin-stdlib-jdk8` are merged into `kotlin-stdlib` itself. If another plugin brings an old version (`1.6.x` for example) of those two, the same classes enter the build twice. In a normal Android project the Kotlin plugin resolves this automatically, but the Unity build does not have that plugin, so it has to be done by hand. Add this block to `mainTemplate.gradle`:<br>`configurations.all {`<br>`  resolutionStrategy.eachDependency { details ->`<br>`    if (details.requested.group == 'org.jetbrains.kotlin' && details.requested.name.startsWith('kotlin-stdlib-jdk')) {`<br>`      details.useTarget 'org.jetbrains.kotlin:kotlin-stdlib:2.0.21'`<br>`    }`<br>`  }`<br>`}` |
| Gradle cannot download the dependencies (`Could not find androidx...` or `Could not resolve ...`) | `dl.google.com` is not reachable (regional blocking). In Publishing Settings turn on **Custom Gradle Settings Template**, and in `settingsTemplate.gradle` add the mirror `maven { url 'https://maven.myket.ir/' }` **before** `google()` in **both** the `pluginManagement` and `dependencyResolutionManagement` blocks. (If the mirror comes after `google()`, the download fails.) |
| `OnAdFailedToLoad(NETWORK_ERROR)` on the device | Check the **device's** access to the service server. |
| The warning `Ads are only available in Android builds` in the Editor | This is normal — ads are only supported in an Android build; test on a device. |
| The installed market does not open and the browser opens instead | The `<queries>` section was dropped during the merge; check the merged manifest output of the build (it must contain the three market packages). |
| `Show()` immediately gives `AD_NOT_READY` | Call `RequestAd()` before showing and wait for `OnAdLoaded`. |
| `AD_EXPIRED` when showing | The cached ad went stale; call `RequestAd()` again in the error handler (the sample code does exactly this). |
| Will not install on the device (Honor/Huawei) | Turn off **Pure Mode** in the device settings, and make sure the build includes **ARM64** and the Backend is **IL2CPP**. |

For easier debugging, Unity-side messages are prefixed with `[MagnetAd]` and can be seen in the Console and in logcat.

---

## Limits

- **Android builds only** (iOS and the Unity Editor are not supported; in the Editor and on other platforms every call returns with a warning, does nothing, and fails).
- **Interstitial** ads only (plain and rewarded video).
- No Banner or Native in this version.
- Your code cannot decide whether an ad arrives as an image or as a video.

---

*Package version: `1.1.0` — readable at runtime from the `MagnetAd.Version` constant.*
