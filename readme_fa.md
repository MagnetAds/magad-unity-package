<div dir="rtl">

# راهنمای استفاده از MagnetAd SDK در یونیتی (ویژه ناشران)

این سند راهنمای گام‌به‌گام نصب و استفاده از پکیج تبلیغاتی **MagnetAd** در پروژه‌های Unity است؛ برای نمایش آگهی **تمام‌صفحه (Interstitial)** روی **اندروید**، چه از نوع **تصویری** و چه از نوع **ویدیویی جایزه‌دار (Rewarded)**.

الگوی استفاده مانند SDKهای شناخته‌شده (AdMob / AppLovin) است: یک بار `Initialize`، سپس به‌ازای هر جایگاه یک آبجکت `InterstitialAd` بسازید، `RequestAd` کنید و وقتی آماده شد `Show` بزنید.

> جایزه‌دار بودن یا نبودن هر جایگاه را خودتان در پنل ناشر مشخص می‌کنید. اما اینکه محتوای هر آگهی تصویر باشد یا ویدیو، از قبل معلوم نیست و کد یکپارچه‌سازی برای هر دو یکی است.

---

## فهرست

1. [معرفی کوتاه](#۱-معرفی-کوتاه)
2. [پیش‌نیازها](#۲-پیشنیازها)
3. [گام ۱ — نصب پکیج](#گام-۱--نصب-پکیج)
4. [گام ۲ — وابستگی‌های Gradle (مهم‌ترین گام)](#گام-۲--وابستگیهای-gradle)
5. [گام ۳ — تنظیمات پروژه (اندروید)](#گام-۳--تنظیمات-پروژه-اندروید)
6. [گام ۴ — دریافت شناسه‌ها](#گام-۴--دریافت-شناسهها)
7. [گام ۵ — نوشتن کد یکپارچه‌سازی](#گام-۵--نوشتن-کد-یکپارچهسازی)
8. [گام ۶ — چرخهٔ عمر آگهی و رویدادها](#گام-۶--چرخهٔ-عمر-آگهی-و-رویدادها)
9. [گام ۷ — آگهی ویدیویی، صدا و جایزه](#گام-۷--آگهی-ویدیویی-صدا-و-جایزه)
10. [آزمایش روی دستگاه](#آزمایش-روی-دستگاه)
11. [مرجع کامل API](#مرجع-کامل-api)
12. [کدهای خطا](#کدهای-خطا)
13. [عیب‌یابی مشکلات رایج](#عیبیابی-مشکلات-رایج)
14. [محدودیت‌ها](#محدودیتها)

---

## ۱. معرفی کوتاه

- نوع آگهی: **تمام‌صفحه (Interstitial)**، تصویری یا ویدیویی. ویدیوهای جایزه‌دار در صورت تماشای کامل، **جایزه (Reward)** می‌دهند ([گام ۷](#گام-۷--آگهی-ویدیویی-صدا-و-جایزه)).
- پلتفرم: فقط **اندروید**.
- چند وابستگی استاندارد Gradle باید به بیلد شما اضافه شود ([گام ۲](#گام-۲--وابستگیهای-gradle)).
- بدون هیچ مجوز خطرناک و بدون دیالوگ درخواست دسترسی از کاربر. مجوز `INTERNET` و سایر تنظیمات لازم به‌صورت خودکار در مانیفست برنامهٔ شما ادغام (merge) می‌شوند — **نیازی به ویرایش دستی مانیفست نیست.**
- تمام متدها **غیرمسدودکننده (non-blocking)** هستند و حلقهٔ بازی شما را متوقف نمی‌کنند. هیچ متدی برای کد شما exception ایجاد نمی‌کند؛ خطاها همیشه از کانال رویدادها گزارش می‌شوند.
- همهٔ رویدادها روی **main thread یونیتی** اجرا می‌شوند — می‌توانید مستقیم از داخل آن‌ها با UI و آبجکت‌های Unity کار کنید.
- نمایش آگهی فقط در **بیلد اندروید** انجام می‌شود. در حالت Play داخل Editor، فراخوانی‌ها بی‌اثرند و با یک هشدار در Console، از کانال خطای معمول ناموفق برمی‌گردند — برای آزمایش واقعی، روی دستگاه بیلد بگیرید.

---

## ۲. پیش‌نیازها

<div dir="ltr">

| مورد | مقدار |
|------|-------|
| نسخهٔ Unity | **2021.3 LTS** تا **Unity 6** |
| حداقل API اندروید (minSdk) | **21** |
| Scripting Backend | **IL2CPP** |
| Target Architectures | شامل **ARM64** |
| سیستم بیلد | **Gradle** (پیش‌فرض Unity) |

</div>

مجوز و تنظیماتی که به‌صورت خودکار به برنامهٔ شما اضافه می‌شوند (نیازی به کار دستی نیست):

<div dir="ltr">

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="com.google.android.gms.permission.AD_ID" />
<queries> <!-- Myket / Cafe Bazaar / Google Play --> </queries>
```

</div>

هیچ‌کدام از این مجوزها **dangerous** نیستند و هیچ دیالوگ درخواست دسترسی به کاربر نشان داده نمی‌شود.

> **‏`AD_ID` و فرم Data safety:** مجوز `com.google.android.gms.permission.AD_ID` به مانیفست شما اضافه می‌شود. اگر برنامه‌تان را در **Google Play** منتشر می‌کنید، باید این مورد را در فرم *Data safety* اعلام کنید. اگر برنامهٔ شما مخاطب کودک دارد و نباید این مجوز را داشته باشد، می‌توانید آن را در مانیفست خودتان حذف کنید:
>
> <div dir="ltr">
>
> ```xml
> <uses-permission android:name="com.google.android.gms.permission.AD_ID" tools:node="remove" />
> ```
>
> </div>
>
> در این حالت هم SDK کار می‌کند؛ فقط دقت کنید وابستگی `play-services-ads-identifier` را همچنان باید اضافه کنید ([گام ۲](#گام-۲--وابستگیهای-gradle))، وگرنه برنامه کرش می‌کند.

> **‏ProGuard/R8:** اگر Minify را در Publishing Settings روشن کرده‌اید، هیچ قانون دستی لازم نیست.

---

## گام ۱ — نصب پکیج

یکی از دو روش زیر را انتخاب کنید:

### روش الف) نصب از طریق Git URL (پیشنهادی)

در یونیتی:
‏`Window → Package Manager → دکمهٔ + → Add package from git URL`
و آدرس زیر را وارد کنید:

<div dir="ltr">

```
https://github.com/MagnetAds/magad-unity-package.git#v1.1.0
```

</div>

> - تگ انتهای آدرس (`#v1.1.0`) پروژهٔ شما را روی همین نسخه قفل می‌کند. برای به‌روزرسانی به نسخهٔ بعدی، فقط شمارهٔ تگ را تغییر دهید.
> - پیش‌نیاز: روی سیستم شما باید **Git** نصب باشد و به گیت‌هاب دسترسی شبکه‌ای داشته باشید.

### روش ب) نصب از فایل .unitypackage یا کپی پوشه

از صفحهٔ Release فایل `.unitypackage` نسخهٔ موردنظر را دانلود و ایمپورت کنید (`Assets → Import Package → Custom Package…`)، یا کل پوشهٔ `MagnetAd` را داخل `Assets` پروژهٔ خود کپی کنید.

آدرس نسخه‌ها (Releases):

<div dir="ltr">

```
https://github.com/MagnetAds/magad-unity-package/releases
```

</div>

> **توجه:** namespace پکیج `MagnetAdSDK` است و کلاس اصلی `MagnetAd`. در کد خود حتماً `using MagnetAdSDK;` بنویسید.

---

## گام ۲ — وابستگی‌های Gradle

**این مهم‌ترین گام راه‌اندازی است.** کتابخانه‌های استانداردی که SDK در زمان اجرا لازم دارد باید توسط Gradle دانلود شوند. اگر این گام را انجام ندهید، برنامه هنگام `Initialize` با خطای `NoClassDefFoundError` کرش می‌کند.

دو روش وجود دارد؛ **فقط یکی** را انجام دهید:

### روش الف) External Dependency Manager (EDM4U)

اگر در پروژهٔ خود [EDM4U](https://github.com/googlesamples/unity-jar-resolver) دارید (همراه اکثر SDKهای گوگل/فایربیس نصب می‌شود)، هیچ کار دستی‌ای لازم نیست: فایل `Editor/MagnetAdDependencies.xml` داخل پکیج به‌صورت خودکار شناسایی و Resolve می‌شود.

### روش ب) افزودن دستی به mainTemplate.gradle

1. در `Project Settings → Player → Android → Publishing Settings` گزینهٔ **Custom Main Gradle Template** را فعال کنید تا فایل `Assets/Plugins/Android/mainTemplate.gradle` ساخته شود.
2. خطوط زیر را داخل بلوک `dependencies` (قبل از `**DEPS**`) اضافه کنید:

<div dir="ltr">

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

</div>

> - اگر SDK دیگری (فایربیس، AdMob و …) همین کتابخانه‌ها را با **نسخهٔ متفاوت** بیاورد مشکلی نیست — Gradle نسخهٔ بالاتر را انتخاب می‌کند. مشکل فقط زمانی است که پلاگین دیگری همین کتابخانه‌ها را به‌صورت **فایل خام jar/aar** در `Plugins/Android` گذاشته باشد (خطای Duplicate class — بخش [عیب‌یابی](#عیبیابی-مشکلات-رایج)).
> - برای دانلود این کتابخانه‌ها، ماشین بیلد باید به `google()`/`mavenCentral()` (یا میرورهایی مثل `maven.myket.ir`) دسترسی داشته باشد.
> - **‏`play-services-ads-identifier` اختیاری نیست.** اگر این کتابخانه نباشد، برنامه در اولین درخواست آگهی با `NoClassDefFoundError` کرش می‌کند. چون دسترسی به مخزن گوگل در ایران محدود است، این کتابخانه و وابستگی‌اش (`play-services-basement`) را از میرور `https://maven.myket.ir/` بگیرید و این میرور را **قبل از** `google()` قرار دهید.
> - **نسخه‌ها را بالاتر نبرید.** این نسخه‌ها طوری انتخاب شده‌اند که با **Target API 33** هم بیلد شوند. مثلاً `androidx.core:core-ktx:1.12.0` (و بالاتر) بیلد را مجبور می‌کند با **API 34** کامپایل شود و روی Unity 2021.3 با خطای `requires libraries and applications that depend on it to compile against version 34` شکست می‌خورد. اگر Target API پروژهٔ شما ۳۴ یا بالاتر است، بالا بردن نسخه‌ها مشکلی ندارد.

---

## گام ۳ — تنظیمات پروژه (اندروید)

### ۳-۱) سوییچ به پلتفرم اندروید

‏`File → Build Settings → Android → Switch Platform`

### ۳-۲) مانیفست

نیازی به افزودن دستی چیزی به مانیفست نیست — همهٔ تنظیمات لازم به‌صورت خودکار در بیلد ادغام می‌شوند.

### ۳-۳) ورودی و EventSystem

آگهی به‌صورت بومی (native) نمایش داده می‌شود و به uGUI، EventSystem یا تنظیمات Input یونیتی **هیچ وابستگی‌ای ندارد**.

---

## گام ۴ — دریافت شناسه‌ها

از پنل ناشر MagnetAd دو شناسه دریافت می‌کنید:

- ‏**App Id** (شناسهٔ Property شما) — در متد `MagnetAd.Initialize` استفاده می‌شود.
- ‏**Placement Id** (شناسهٔ جایگاه آگهی) — در سازندهٔ `InterstitialAd` استفاده می‌شود.

هر جایگاه را هنگام ساختن در پنل، معمولی یا جایزه‌دار تعریف می‌کنید، پس از قبل می‌دانید شناسه‌ای که در کد گذاشته‌اید از کدام نوع است. جایگاه جایزه‌دار را فقط جایی بگذارید که واقعاً به کاربر جایزه می‌دهید.

> این شناسه‌ها به‌صورت رشته (string) ارسال می‌شوند و اعتبارسنجی آن‌ها بر عهدهٔ سرور است.

---

## گام ۵ — نوشتن کد یکپارچه‌سازی

یک اسکریپت بسازید (مثلاً `AdsManager.cs`) و آن را روی یک GameObject در صحنه بگذارید. الگو: **یک آبجکت `InterstitialAd` برای هر جایگاه بسازید، در رویدادهای همان آبجکت subscribe کنید، `RequestAd` و سپس `Show`.**

<div dir="ltr">

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
            _interstitial.OnRewarded       += () => GrantCoins(50);   // فقط جایگاه جایزه‌دار
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

</div>

توضیح کوتاه هر بخش:

- ‏`Start` — یک بار `Initialize` می‌کند و بعد از موفقیت، اولین آگهی را پیش‌بارگذاری می‌کند.
- ‏`LoadAd` — آبجکت آگهی را فقط یک بار می‌سازد (برای کل نشست کافی است)، رویدادها را وصل می‌کند و `RequestAd` می‌زند. در `OnAdShown` بازی متوقف و در `OnAdDismissed` و `OnAdFailedToShow` دوباره اجرا می‌شود.
- ‏`ShowAdNow` — این متد را به رویداد مناسب بازی خود وصل کنید (مثلاً پایان مرحله). اگر آگهی آماده نباشد، یکی درخواست می‌دهد.
- ‏`OnDestroy` — منابع را آزاد می‌کند.

### نکات کلیدی

- **رویدادها به آبجکت تعلق دارند:** روی هر نمونهٔ `InterstitialAd` جداگانه subscribe می‌کنید و با `Destroy` همان نمونه همه‌چیز آزاد می‌شود.
- **هر نمایش، آگهی را مصرف می‌کند:** بعد از هر `Show` موفق (رویداد `OnAdDismissed`) برای نمایش بعدی دوباره `RequestAd` بزنید.
- ‏**`Destroy` را فراموش نکنید:** در `OnDestroy` مالکِ آگهی، متد `Destroy()` را صدا بزنید تا منابع آزاد شوند. کلاس `IDisposable` هم هست (قابل استفاده با `using`). اگر فراموش شود، SDK هنگام جمع‌آوری آبجکت یک هشدار در Console چاپ می‌کند.
- **اگر هنگام `Destroy` آگهی روی صفحه باشد:** رویداد `OnAdDismissed` باز هم صادر می‌شود. پس اگر بازی را در `OnAdShown` متوقف کرده باشید، حتی با تعویض صحنه در وسط نمایش آگهی، بازی از حالت توقف خارج می‌شود.
- **پیش‌بارگذاری:** آگهی را زودتر (پس از `Initialize` موفق یا پس از بسته‌شدن آگهی قبلی) درخواست کنید تا هنگام نیاز فوراً آماده باشد.
- **کلیک روی آگهی:** با کلیک کاربر، صفحهٔ مارکت (مایکت / بازار / گوگل‌پلی) یا مرورگر باز می‌شود و بازی شما به پس‌زمینه می‌رود. آگهی هم بلافاصله بسته می‌شود، پس پس از `OnAdClicked` رویداد `OnAdDismissed` را هم دریافت می‌کنید.

---

## گام ۶ — چرخهٔ عمر آگهی و رویدادها

ترتیب معمولِ رخدادها:

<div dir="ltr">

```
MagnetAd.Initialize(appId, onComplete)
        │
        └─►  onComplete(true/false)          + رویداد MagnetAd.OnInitializationComplete

new InterstitialAd(placementId)  →  RequestAd()
        │
        ├─►  OnAdLoading                     (درخواست شروع شد)
        ├─►  OnAdLoaded                      ✅ آماده (IsLoaded = true)
        └─►  OnAdFailedToLoad(err)           ❌

Show()
        │
        ├─►  OnAdShown                       (آگهی روی صفحه آمد)
        │         ├─►  OnAdClicked  ─────►  OnAdDismissed
        │         │                          (کلیک، مارکت/مرورگر را باز و آگهی را می‌بندد)
        │         ├─►  OnRewarded            (فقط ویدیوی جایزه‌دار: تماشای کامل ویدیو)
        │         └─►  OnAdDismissed         (کاربر با دکمهٔ ✕ آگهی را بست)
        │
        └─►  OnAdFailedToShow(err)           (AD_NOT_READY / AD_EXPIRED / VIDEO_TIMEOUT / …)
```

</div>

| رویداد | نوع | توضیح |
|--------|-----|-------|
| <span dir="ltr">`MagnetAd.OnInitializationComplete`</span> | <span dir="ltr">`Action<bool>`</span> | <span dir="rtl">پس از پایان `Initialize`، همراه نتیجه صادر می‌شود (علاوه بر کال‌بک خود متد). اگر بعد از پایان `Initialize` هم subscribe کنید، نتیجه را در فریم بعد دریافت می‌کنید؛ پس ترتیب فراخوانی برایتان مهم نیست. هندلرها بعد از صدور رویداد آزاد می‌شوند.</span> |
| <span dir="ltr">`OnAdLoading`</span> | <span dir="ltr">`Action`</span> | <span dir="rtl">درخواست آگهی آغاز شد.</span> |
| <span dir="ltr">`OnAdLoaded`</span> | <span dir="ltr">`Action`</span> | <span dir="rtl">آگهی + تصویر آماده شد؛ `IsLoaded` برابر true می‌شود.</span> |
| <span dir="ltr">`OnAdFailedToLoad`</span> | <span dir="ltr">`Action<AdError>`</span> | <span dir="rtl">درخواست آگهی ناموفق بود (شبکه/سرور/تصویر).</span> |
| <span dir="ltr">`OnAdShown`</span> | <span dir="ltr">`Action`</span> | <span dir="rtl">آگهی نمایش داده شد.</span> |
| <span dir="ltr">`OnAdClicked`</span> | <span dir="ltr">`Action`</span> | <span dir="rtl">کاربر روی تصویر آگهی زد (هر آگهی یک بار). بلافاصله پس از آن `OnAdDismissed` هم صادر می‌شود.</span> |
| <span dir="ltr">`OnAdDismissed`</span> | <span dir="ltr">`Action`</span> | <span dir="rtl">آگهی بسته شد (با دکمهٔ ✕ یا در پی یک کلیک).</span> |
| <span dir="ltr">`OnAdFailedToShow`</span> | <span dir="ltr">`Action<AdError>`</span> | <span dir="rtl">نمایش ممکن نشد (آماده نبود، منقضی شده، تصویر رندر نشد و …).</span> |
| <span dir="ltr">`OnRewarded`</span> | <span dir="ltr">`Action`</span> | <span dir="rtl">فقط برای ویدیوی جایزه‌دار: کاربر ویدیو را کامل دید. برای آگهی تصویری هیچ‌وقت صادر نمی‌شود. جزئیات در [گام ۷](#گام-۷--آگهی-ویدیویی-صدا-و-جایزه).</span> |

> هنگام نمایش آگهی، دکمهٔ بستن (✕) ابتدا یک **شمارش معکوس چندثانیه‌ای** نشان می‌دهد و سپس فعال می‌شود. در طول نمایش، دکمهٔ سخت‌افزاری Back غیرفعال است.

> **دو نکتهٔ مهم دربارهٔ ترتیب رویدادها:**
> - **کلیک، آگهی را می‌بندد.** پس از `OnAdClicked` همیشه `OnAdDismissed` هم می‌آید؛ پس اگر بازی را در `OnAdShown` متوقف می‌کنید، ادامهٔ آن در `OnAdDismissed` برای هر دو مسیر (بستن دستی و کلیک) درست کار می‌کند.
> - **اگر تصویر آگهی پس از باز شدن رندر نشود**، آگهی بسته می‌شود و فقط `OnAdFailedToShow` با کد `ASSET_LOAD_FAILED` صادر می‌شود — در این حالت `OnAdDismissed` نمی‌آید. بنابراین اگر بازی را متوقف کرده‌اید، حتماً در `OnAdFailedToShow` هم آن را از سرگیری کنید (نمونه‌کد همین کار را می‌کند).

---

## گام ۷ — آگهی ویدیویی، صدا و جایزه

آگهی هر درخواست ممکن است تصویری باشد یا ویدیویی. کد شما برای هر دو یکسان است: همان `RequestAd` و همان `Show`. تفاوت فقط در دو چیز است — **صدا** و **جایزه**.

### ۷-۱) کنترل صدای ویدیو

بیشتر بازی‌ها خودشان یک تنظیم صدا دارند و بد است که آگهی ویدیویی این تنظیم را نادیده بگیرد. دو پراپرتی روی کلاس `MagnetAd` این کار را انجام می‌دهند:

<div dir="ltr">

```csharp
MagnetAd.VideoVolume = 0.5f;   // عددی بین 0f (بی‌صدا) و 1f (بلندترین)، پیش‌فرض 1f
MagnetAd.VideoMuted  = true;   // بی‌صدا کردن بدون از دست دادن مقدار VideoVolume
```

</div>

- هر دو را می‌توانید **قبل از** `Initialize` تنظیم کنید تا همان اولین ویدیو با صدای درست شروع شود، یا **هر زمان بعد از آن** — حتی وقتی ویدیو روی صفحه در حال پخش است؛ تغییر بلافاصله اعمال می‌شود.
- ‏`VideoMuted` مقدار `VideoVolume` را پاک نمی‌کند. وقتی دوباره `false` شود، صدا به همان سطح قبلی برمی‌گردد. برای دکمهٔ بی‌صدای بازی، همین `VideoMuted` گزینهٔ درست است.
- مقادیر خارج از بازهٔ `0f..1f` به‌طور خودکار به همان بازه محدود می‌شوند.

### ۷-۲) تشخیص آگهی جایزه‌دار

نوع هر جایگاه را خودتان هنگام ساختن آن در پنل ناشر انتخاب کرده‌اید، پس از قبل می‌دانید شناسه‌ای که در کد گذاشته‌اید مربوط به جایگاه جایزه‌دار (Rewarded) است یا تمام‌صفحهٔ ساده (Interstitial). ساده‌ترین کار این است که بر همان اساس کد بنویسید: برای جایگاه جایزه‌دار هندلر جایزه بگذارید و برای جایگاه ساده لازم نیست.

اگر لازم شد در زمان اجرا هم مطمئن شوید، پس از `OnAdLoaded` مقدار `PlacementType` همین را به شما می‌گوید:

<div dir="ltr">

```csharp
_interstitial.OnAdLoaded += () =>
{
    if (_interstitial.PlacementType == PlacementType.REWARDED)
        ShowRewardPromiseDialog();   // «این ویدیو را کامل ببین و ۵۰ سکه بگیر»
};
```

</div>

مقدار `PlacementType` تا قبل از بارگذاری آگهی `UNKNOWN` است و پس از آن یکی از `INTERSTITIAL` یا `REWARDED`.

### ۷-۳) دریافت جایزه

وقتی کاربر ویدیو را **کامل** ببیند، رویداد `OnRewarded` روی همان آبجکت آگهی صادر می‌شود:

<div dir="ltr">

```csharp
_interstitial.OnRewarded += () => GrantCoins(50);
```

</div>

> این رویداد را **هم‌زمان با بقیهٔ رویدادها** و پیش از `RequestAd` وصل کنید. اگر هندلری وصل نباشد، آن جایزه از دست می‌رود؛ SDK آن را برای بعد نگه نمی‌دارد.

> جایزه در همان لحظهٔ پایان تماشا اعلام می‌شود و منتظر پاسخ سرور نمی‌ماند، تا قطعی شبکه کاربر را از جایزه‌اش محروم نکند. اگر اعطای جایزه برای شما حساس است، آن را روی سرور خودتان هم ثبت کنید.

### ۷-۴) نکات نمایش ویدیو

- ویدیوی جایزه‌دار معمولاً تا پایان پخش بسته نمی‌شود، تا کاربر نتواند نیمه‌کاره خارج شود و بعد جایزه بخواهد.
- روشنایی صفحه در طول پخش به روشنایی سیستم دستگاه احترام می‌گذارد.
- اگر پخش ویدیو گیر کند و هیچ پاسخی نیاید، SDK خودش آگهی را می‌بندد و `OnAdFailedToShow` با کد `VIDEO_TIMEOUT` صادر می‌کند تا بازی شما معطل نماند.

---

## آزمایش روی دستگاه

آزمایش SDK فقط روی **دستگاه اندرویدی** ممکن است: یک بیلد اندروید بگیرید و کل مسیر — `Initialize`، درخواست آگهی و نمایش — را روی دستگاه بررسی کنید.

- در حالت Play داخل Editor هیچ ارتباط شبکه‌ای برقرار نمی‌شود؛ پیام `[MagnetAd] Ads are only available in Android builds.` در Console چاپ می‌شود، `Initialize` با `false` برمی‌گردد و `RequestAd`/`Show` از طریق رویدادهای خطا ناموفق می‌شوند.
- پیام‌های سمت یونیتی با پیشوند `[MagnetAd]` در Console و logcat دیده می‌شوند.

> مقدار `MagnetAd.GetSDKVersion()` در Editor برابر `"unsupported"` است.

---

## مرجع کامل API

<div dir="ltr">

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
        public event Action OnRewarded;                    // video ads only

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

</div>

| متد | توضیح |
|-----|-------|
| <span dir="ltr">`MagnetAd.Initialize(appId, cb)`</span> | <span dir="rtl">غیرمسدودکننده. در پس‌زمینه با سرور ارتباط برقرار می‌کند و نتیجه را با `bool` برمی‌گرداند. کال‌بک همیشه صدا زده می‌شود و هیچ‌وقت بی‌پاسخ نمی‌ماند. برای تلاش مجدد، دوباره صدا بزنید.</span> |
| <span dir="ltr">`RequestAd()`</span> | <span dir="rtl">غیرمسدودکننده. نتیجه همیشه از طریق `OnAdLoaded` یا `OnAdFailedToLoad` می‌آید. اگر درخواستی در جریان باشد، فراخوانی دوباره نادیده گرفته می‌شود (درخواست دوم به سرور نمی‌رود) و یک هشدار در Console چاپ می‌شود.</span> |
| <span dir="ltr">`Show()`</span> | <span dir="rtl">آگهیِ از پیش بارگذاری‌شده را نمایش می‌دهد. اگر آگهی آماده نباشد `OnAdFailedToShow(AD_NOT_READY)` صادر می‌شود.</span> |
| <span dir="ltr">`Destroy()`</span> | <span dir="rtl">منابع آگهی را آزاد می‌کند. پس از آن، نمونه غیرقابل استفاده است. اگر هنگام صدا زدنش آگهی روی صفحه باشد، `OnAdDismissed` هم صادر می‌شود تا بازیِ متوقف‌شده حتماً از سر گرفته شود.</span> |
| <span dir="ltr">`MagnetAd.VideoVolume`</span> | <span dir="rtl">صدای آگهی ویدیویی، عددی بین `0f` و `1f` (پیش‌فرض `1f`). قبل از `Initialize` یا هر زمان بعد از آن — حتی وسط پخش — قابل تغییر است. مقادیر بیرون از بازه به همان بازه محدود می‌شوند.</span> |
| <span dir="ltr">`MagnetAd.VideoMuted`</span> | <span dir="rtl">بی‌صدا کردن ویدیو بدون پاک شدن `VideoVolume`؛ با `false` شدن دوباره، صدا به همان سطح قبلی برمی‌گردد.</span> |
| <span dir="ltr">`PlacementType`</span> | <span dir="rtl">نوع آگهیِ بارگذاری‌شده. پیش از `OnAdLoaded` برابر `UNKNOWN` است. جزئیات در [گام ۷](#گام-۷--آگهی-ویدیویی-صدا-و-جایزه).</span> |

> **دربارهٔ `IsLoaded`:** یک آگهیِ منقضی‌شده به‌سرعت `false` گزارش می‌شود، پس این مقدار کهنه نمی‌ماند. هر نمایش موفق، آگهی را مصرف می‌کند و `IsLoaded` دوباره `false` می‌شود؛ برای نمایش بعدی باید `RequestAd()` بزنید. این پراپرتی را فقط از ترد اصلی بخوانید.

> **فراخوانی از تردهای دیگر:** همهٔ متدهای عمومی را می‌توانید از هر تردی صدا بزنید (مثلاً داخل `ContinueWith` فایربیس). SDK خودش کار را به ترد اصلی منتقل می‌کند. همهٔ رویدادها هم همیشه روی ترد اصلی Unity صادر می‌شوند.

---

## کدهای خطا

مقدار `AdError.Code` را همیشه با ثابت‌های کلاس `AdErrorCode` مقایسه کنید، نه با رشتهٔ دستی.

| ثابت C# | معنی |
|---------|------|
| <span dir="ltr">`AdErrorCode.NOT_INITIALIZED`</span> | <span dir="rtl">قبل از `Initialize` موفق، RequestAd/Show صدا زده شده است.</span> |
| <span dir="ltr">`AdErrorCode.NETWORK_ERROR`</span> | <span dir="rtl">سرور در دسترس نبود / خطای شبکه.</span> |
| <span dir="ltr">`AdErrorCode.SERVER_ERROR`</span> | <span dir="rtl">خطای سمت سرور (5xx).</span> |
| <span dir="ltr">`AdErrorCode.INVALID_RESPONSE`</span> | <span dir="rtl">پاسخ سرور قابل پردازش نبود.</span> |
| <span dir="ltr">`AdErrorCode.ASSET_LOAD_FAILED`</span> | <span dir="rtl">دانلود یا نمایش تصویر آگهی ناموفق بود.</span> |
| <span dir="ltr">`AdErrorCode.NO_FILL`</span> | <span dir="rtl">در حال حاضر آگهی مناسبی برای این درخواست وجود ندارد. خطا نیست؛ کمی بعد دوباره درخواست دهید.</span> |
| <span dir="ltr">`AdErrorCode.AD_NOT_READY`</span> | <span dir="rtl">`Show()` بدون آگهیِ بارگذاری‌شده صدا زده شد.</span> |
| <span dir="ltr">`AdErrorCode.AD_EXPIRED`</span> | <span dir="rtl">آگهیِ کش‌شده منقضی شده است؛ دوباره RequestAd کنید.</span> |
| <span dir="ltr">`AdErrorCode.ACTIVITY_UNAVAILABLE`</span> | <span dir="rtl">صفحهٔ برنامه در دسترس نبود و نمایش ممکن نشد.</span> |
| <span dir="ltr">`AdErrorCode.SHOW_FAILED`</span> | <span dir="rtl">خطای نامشخص هنگام نمایش آگهی.</span> |
| <span dir="ltr">`AdErrorCode.TIMEOUT`</span> | <span dir="rtl">دریافت آگهی خیلی طول کشید.</span> |
| <span dir="ltr">`AdErrorCode.UNEXPECTED_AD_TYPE`</span> | <span dir="rtl">سرور نوعی از آگهی فرستاد که این نسخه از SDK نمی‌شناسد.</span> |
| <span dir="ltr">`AdErrorCode.VIDEO_PLAYBACK_ERROR`</span> | <span dir="rtl">پخش ویدیو با خطا متوقف شد.</span> |
| <span dir="ltr">`AdErrorCode.VIDEO_SERVER_DIED`</span> | <span dir="rtl">پخش‌کنندهٔ ویدیوی سیستم‌عامل از کار افتاد (معمولاً مشکل خود دستگاه است).</span> |
| <span dir="ltr">`AdErrorCode.VIDEO_TIMEOUT`</span> | <span dir="rtl">ویدیو شروع به پخش نکرد یا وسط کار گیر کرد و هیچ پاسخی نداد؛ SDK آگهی را بست.</span> |
| <span dir="ltr">`AdErrorCode.UNKNOWN`</span> | <span dir="rtl">سایر خطاها.</span> |

---

## عیب‌یابی مشکلات رایج

| نشانه | علت و راه‌حل |
|-------|--------------|
| خطای `The type or namespace name 'MagnetAdSDK' could not be found` | یا پکیج کامل نصب نشده (خطای دیگری در Console را برطرف کنید)، یا `using MagnetAdSDK;` را فراموش کرده‌اید. |
| کرش هنگام `Initialize` روی دستگاه با `NoClassDefFoundError: kotlinx/coroutines/...` (یا okhttp/serialization) | وابستگی‌های Gradle اضافه نشده‌اند — [گام ۲](#گام-۲--وابستگیهای-gradle) را انجام دهید. |
| کرش هنگام اولین درخواست آگهی با `NoClassDefFoundError: com/google/android/gms/ads/identifier/AdvertisingIdClient` | وابستگی `play-services-ads-identifier` اضافه نشده است — [گام ۲](#گام-۲--وابستگیهای-gradle) را انجام دهید. |
| خطای بیلد `Duplicate class kotlin...` یا `Duplicate class okhttp3...` | پلاگین دیگری همین کتابخانه‌ها را به‌صورت **jar/aar خام** در `Plugins/Android` گذاشته است. فایل خام تکراری را حذف کنید تا نسخهٔ Maven جایگزین شود. |
| خطای بیلد `Duplicate class kotlin.collections.jdk8...` (یا هر کلاسی از `kotlin-stdlib-jdk7` / `kotlin-stdlib-jdk8`) | این حالت با حذف فایل خام درست نمی‌شود. از Kotlin 1.8 به بعد، `kotlin-stdlib-jdk7` و `kotlin-stdlib-jdk8` داخل خودِ `kotlin-stdlib` ادغام شده‌اند. اگر پلاگین دیگری نسخهٔ قدیمی (مثلاً `1.6.x`) از آن دو را بیاورد، همان کلاس‌ها دوبار وارد بیلد می‌شوند. در پروژه‌های اندروید معمولی، پلاگین Kotlin این تداخل را خودکار حل می‌کند، اما بیلد Unity آن پلاگین را ندارد، پس باید دستی حل شود. این بلوک را در `mainTemplate.gradle` اضافه کنید: <div dir="ltr"><br>`configurations.all {`<br>`  resolutionStrategy.eachDependency { details ->`<br>`    if (details.requested.group == 'org.jetbrains.kotlin' && details.requested.name.startsWith('kotlin-stdlib-jdk')) {`<br>`      details.useTarget 'org.jetbrains.kotlin:kotlin-stdlib:2.0.21'`<br>`    }`<br>`  }`<br>`}`</div> |
| ‏Gradle نمی‌تواند وابستگی‌ها را دانلود کند (`Could not find androidx...` یا `Could not resolve ...`) | ‏`dl.google.com` در دسترس نیست (مسدودسازی منطقه‌ای). در Publishing Settings گزینهٔ **Custom Gradle Settings Template** را فعال کنید و در `settingsTemplate.gradle` میرور `maven { url 'https://maven.myket.ir/' }` را **قبل از** `google()` در **هر دو** بلوک `pluginManagement` و `dependencyResolutionManagement` اضافه کنید. (اگر میرور بعد از `google()` باشد، دانلود شکست می‌خورد.) |
| روی دستگاه `OnAdFailedToLoad(NETWORK_ERROR)` | دسترسی **دستگاه** به سرور سرویس را بررسی کنید. |
| در Editor هشدار `Ads are only available in Android builds` | رفتار طبیعی است — نمایش آگهی فقط در بیلد اندروید پشتیبانی می‌شود؛ روی دستگاه آزمایش کنید. |
| مارکت نصب‌شده باز نمی‌شود و به مرورگر می‌رود | بخش `<queries>` هنگام merge حذف شده است؛ خروجی merged manifest بیلد را بررسی کنید (باید سه پکیج مارکت را داشته باشد). |
| ‏`Show()` بلافاصله `AD_NOT_READY` می‌دهد | قبل از نمایش، `RequestAd()` بزنید و منتظر `OnAdLoaded` بمانید. |
| ‏`AD_EXPIRED` هنگام نمایش | آگهیِ کش‌شده کهنه شده؛ در هندلر خطا دوباره `RequestAd()` بزنید (الگوی نمونه‌کد همین کار را می‌کند). |
| روی دستگاه نصب نمی‌شود (Honor/Huawei) | حالت **Pure Mode** را در تنظیمات دستگاه خاموش کنید؛ همچنین مطمئن شوید بیلد شامل **ARM64** و Backend روی **IL2CPP** است. |

برای دیباگ بهتر، پیام‌های سمت یونیتی با پیشوند `[MagnetAd]` در Console و logcat قابل مشاهده‌اند.

---

## محدودیت‌ها

- فقط **بیلد اندروید** (iOS و Unity Editor پشتیبانی نمی‌شوند؛ در Editor و سایر پلتفرم‌ها همهٔ فراخوانی‌ها با هشدار، بی‌اثر و ناموفق برمی‌گردند).
- فقط آگهی **Interstitial** (تصویری و ویدیویی جایزه‌دار).
- بدون Banner و Native در این نسخه.
- کد شما نمی‌تواند تعیین کند محتوای آگهی تصویر باشد یا ویدیو.

---

*نسخهٔ پکیج: `1.1.0` — در زمان اجرا از ثابت `MagnetAd.Version` قابل خواندن است.*

</div>
