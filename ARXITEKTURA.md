# O'zbegim 0.9.1: arxitektura qaydlari

Bu hujjat uch muammoning sababini, kiritilgan tuzatishni va telefonda sinash tartibini bayon qiladi. Kod namunalari
haqiqiy fayllardan olingan.

## Holat (halol ma'lumot)

- 0.9.1 tuzatishlari (5-bo'lim) ham Gradle bilan qurilmadi (kompilyator yo'q) va telefonda sinalmadi. Qavslar balansi, nomlarning
  bir marta e'lon qilinishi va chaqiruvlar qo'lda tekshirildi.
- 5-bo'limdagi Telegram sabablarining bir qismi kod o'qishdan aniq, bir qismi taxmin: yuborilgan jurnalda Telegram yozuvi yo'q edi.
  Qaysi biri taxmin ekani 5.1'da alohida belgilangan.
- Tuzatishlar v12 kodiga kiritildi va qo'lda tekshirildi (qavslar balansi, nomlar, chaqiruvlar).
- Gradle bilan qurilmadi (tahrir muhitida tarmoq yo'q) va telefonda sinalmadi. Qurishda xato chiqsa, xato matnini yuboring.
- Ishora haqidagi bir taxmin (pastda, 3-bo'lim) tekshirilmagan; shu sababli ilovalar menyusi bir necha yo'l bilan ochiladi va
  qaysi yo'l ishlagani Xatolar jurnaliga yoziladi.

## 1. Fokus olish va elementni aniqlash

Teginib o'rganishda tizim barmoq ostidagi tugmani o'zi aniqlaydi va `TYPE_VIEW_HOVER_ENTER` hodisasiga manba qilib beradi
(`event.source`). Hodisada koordinata yo'q, shuning uchun "eng yaqin koordinata" qidiruvi uchun ma'lumot yetishmaydi.
`onMotionEvent` yordam bermaydi: Android hujjatiga ko'ra teginib o'rganish yoqiq bo'lsa, u sensor hodisalarini olmaydi.
Koordinata kerak bo'lsa, keyingi bosqichda `TouchInteractionController` (Android 11+) sinab ko'riladi; telefonsiz buni xavfsiz
qo'shib bo'lmaydi. `findAccessibilityNodeInfosByViewId` ham mos emas: id lar ilovaga xos va ko'pincha yo'q.

Shuning uchun aniqlash yorliq va ahamiyat qoidasiga tayanadi (`core/Navigator.kt`):

```kotlin
fun isActionable(n: AccessibilityNodeInfo): Boolean =
    n.isClickable || n.isLongClickable || n.isCheckable || n.isEditable || (n.isFocusable && n.childCount == 0)

fun isMeaningful(n: AccessibilityNodeInfo): Boolean {
    if (!n.isVisibleToUser) return false
    return hasOwnLabel(n) || isActionable(n) || n.isFocusable || n.isScreenReaderFocusable
}
```

Yangi: teginilgan tugunda ham, ota-onalarida ham yorliq bo'lmasa (jim qolardi), tugun kichik konteyner bo'lsa, uning ichidagi
birinchi ahamiyatli tugun olinadi:

```kotlin
fun unitFor(start: AccessibilityNodeInfo, descendantAreaLimit: Long = 0L): AccessibilityNodeInfo? {
    // ... bosiladigan ota-ona, tugunning o'zi, yorliqli ota-ona ...
    if (descendantAreaLimit > 0L && start.childCount > 0) {
        val r = Rect()
        start.getBoundsInScreen(r)
        if (r.width().toLong() * r.height().toLong() <= descendantAreaLimit) return firstMeaningfulDescendant(start)
    }
    return null
}
```

Chegara ekran maydonining 30 foizi (`ReaderService.processHover`): katta bo'sh joyga tegilganda tasodifiy bola o'qilmasligi uchun.

## 2. Xabarlarning o'tkazib yuborilishi (Telegram kanallari, ro'yxatlar)

Sabab: postning matni konteynerning o'zida turadi, ichida esa tugmalar va ba'zan oddiy matnli bolalar (ko'rishlar soni, vaqt)
bo'ladi. Eski qoida ichida oddiy matnli bola bo'lsa konteynerni fokus nuqtasi deb hisoblamasdi: asosiy matn tushib qolar,
tugmalar va bolalar o'qilardi.

Tuzatish: konteynerning o'z yorlig'i hech qachon tashlab ketilmaydi. Takror o'qilmasligi uchun matni ota-onasining
yorlig'ida to'liq bor bola alohida nuqta bo'lmaydi.

```kotlin
fun isLabeledUnit(n: AccessibilityNodeInfo): Boolean {
    if (!hasOwnLabel(n)) return false
    val count = n.childCount
    if (count == 0) return true
    return !(n.isScrollable || n.collectionInfo != null || count > 12)
}

fun coveredByLabeledParent(n: AccessibilityNodeInfo): Boolean {
    if (n.isScreenReaderFocusable || n.childCount != 0) return false
    val own = labelText(n)
    if (own.isEmpty()) return false
    var p: AccessibilityNodeInfo? = n.parent
    var depth = 0
    while (p != null && depth++ < 2) {
        if (p.isScrollable || p.collectionInfo != null || isActionable(p)) return false
        if (isLabeledUnit(p)) {
            val pl = labelText(p)
            if (pl.length >= own.length && pl.contains(own, ignoreCase = true)) return true
        }
        p = p.parent
    }
    return false
}
```

Tartib: `collect()` va `nextUnit()` daraxtni chuqurlik bo'yicha (DFS) bosib o'tadi, shuning uchun xabar matni, keyin vaqt, keyin
tugmalar ketma-ket keladi. Ro'yxat qatorlarini bitta obyektga birlashtirish (TalkBack "top-level scroll item") qilinmadi:
bu hamma ro'yxatning o'qilishini o'zgartiradi va telefonsiz tekshirib bo'lmaydi. Hozirgi ketma-ketlik hech narsani
tashlab ketmaydi.

Chuqur joylashgan matn: yorlig'siz konteynerning matnini yig'uvchi `NodeDescriber.rollUp` chuqurligi 6 gacha, soni 8 gacha
(tezkor rejimda 3 va 4), tugunlar byudjeti 90 (tezkor rejimda 40).

Aylantirish (`TYPE_VIEW_SCROLLED`, `ReaderService.onScrolled/handleScrolled`): hodisa soniyasiga o'nlab marta keladi, shuning
uchun har hodisada faqat kutish vaqti suriladi, ish aylantirish tinchigach (160 ms) bir marta bajariladi:

```kotlin
private fun onScrolled(event: AccessibilityEvent) {
    if (SystemClock.uptimeMillis() < ownScrollUntil) return      // xizmatning o'z aylantirishi: fokusni o'zi boshqaradi
    if (scrollSource == null) scrollSource = event.source
    main.removeCallbacks(scrollTask)
    main.postDelayed(scrollTask, SCROLL_SETTLE_MS)
}

private fun handleScrolled() {
    // ... navigator.clearCaches(); foydalanuvchi so'nggi 4 soniyada tekkan bo'lsa ...
    val focusGone = last == null || !last.refresh() || !last.isVisibleToUser
    if (!focusGone) return                                       // fokus ko'rinib tursa, tegmaymiz
    val first = navigator.collect(source).firstOrNull { navigator.matches(it, navigator.mode) } ?: return
    focus(first)
}
```

## 3. Ishoralar va global amallar

### Joriy arxitektura

`accessibility_service_config.xml` teginib o'rganish va ko'p barmoqli ishoralarni so'raydi:

```xml
android:accessibilityFlags="flagDefault|flagReportViewIds|flagRequestTouchExplorationMode|flagRequestFilterKeyEvents|flagRetrieveInteractiveWindows|flagEnableAccessibilityVolume|flagRequestMultiFingerGestures"
android:canPerformGestures="true"
android:canRequestTouchExplorationMode="true"
android:canRetrieveWindowContent="true"
```

Teginib o'rganish yoqiq bo'lganda tizim barcha barmoq harakatini o'zi ushlab oladi va faqat tayyor ishora sifatida
`onGesture(gestureId)` ga beradi. Bosh ekran (launcher) xom barmoq harakatini ko'rmaydi: bu "to'liq intercept"ning o'zi,
alohida kod kerak emas. `onGesture` -> `GestureMap` (ishora -> amal) -> `perform(amal)`.

`onMotionEvent` bu yerda ishora ushlash uchun yaramaydi: u faqat kuzatish uchun (Android 14+) va teginib o'rganish yoqiq
bo'lsa sensor hodisalarini bermaydi.

Ikki barmoqli surishni ilovaning o'ziga o'tkazish kerak bo'lsa (qulf ekrani), `FLAG_REQUEST_2_FINGER_PASSTHROUGH` ishlatiladi
(`ReaderService.updatePassthrough`, sozlama `two_finger_pass`).

### Ilovalar menyusi muammosi

Eski kod bosh ekranda ikki barmoq bilan tepaga surishni `dispatchGesture` bilan yuborilgan barmoq surishi orqali bajarardi.
Taxmin (tekshirilmagan): teginib o'rganish yoqiq paytda tizim xizmatning o'zi yuborgan surishni ham o'z ushlagichidan
o'tkazadi va u launcherga yetmaydi. Taxmin to'g'ri bo'lsa, eski yo'l hech qachon ishlamasdi.

Yangi yo'l (`core/SystemSurfaces.kt`): uch qadam ketma-ket, har biridan keyin ekran o'zgarganmi tekshiriladi:

```kotlin
fun openApps(onDone: (Boolean) -> Unit = {}) {
    // ...
    val before = screenSignature()
    val steps = ArrayList<Step>(3)
    if (Build.VERSION.SDK_INT >= 31) {
        steps.add(Step("global") { svc.performGlobalAction(AccessibilityService.GLOBAL_ACTION_ACCESSIBILITY_ALL_APPS) })
    }
    steps.add(Step("button") { clickAppsButton() })
    steps.add(Step("swipe") { injectSwipe(0.5f, 0.74f, 0.5f, 0.26f, 220L) })
    runStep(steps, 0, before, onDone)
}
```

Oxirgi qadamda teginib o'rganish 250 ms o'chiriladi va surish tugagach (yoki 2 soniyalik xavfsizlik muddatidan keyin) qayta
yoqiladi:

```kotlin
val teWasOn = touchExplorationOn()
if (teWasOn) { teSuspendedByUs = true; setTouchExploration(false) }
main.postDelayed(restoreTask, settle + durationMs + RESTORE_MARGIN_MS)
main.postDelayed({ svc.dispatchGesture(desc, callback /* onCompleted/onCancelled -> restore */, main) }, settle)
```

Boshqa o'zgarishlar: ikki barmoq bilan pastga tortish bosh ekranda to'g'ridan-to'g'ri `GLOBAL_ACTION_NOTIFICATIONS`;
soyani yopish Android 12+ da `GLOBAL_ACTION_DISMISS_NOTIFICATION_SHADE`; qulf ekranini ochish va sahifa almashtirish
surishlari ham xavfsiz yuborish yo'lidan o'tadi (`ReaderService.fingerSwipe`). Yangi ishora amali `OPEN_APPS`
("Ilovalar menyusini ochish") istalgan ishoraga biriktirilishi mumkin.

## 4. Telefonda sinash ro'yxati

1. Bosh ekranda ikki barmoq bilan tepaga surish: ilovalar menyusi ochiladimi.
2. Bosh ekranda ikki barmoq bilan pastga tortish: xabarnoma soyasi tushadimi; soya ochiq paytda tepaga surish uni yopadimi.
3. Qulf ekranida ikki barmoq bilan tepaga surish: qulf ochiladimi.
4. Teginib o'rganish tugagandan keyin ishlayaptimi (surishdan keyin ekranga tegib, o'qilishini tekshiring).
5. Telegram kanalida xabarlar bo'ylab o'ngga suring: avval post matni, keyin tugmalar o'qiladimi.
6. Telegram kanalida ikki barmoq bilan aylantirish: fokus yangi ko'ringan birinchi elementga o'tadimi.
7. Ro'yxatlarda (Sozlamalar, Telegram chatlar) yurish avvalgidek tez va takrorsiz ishlayaptimi.

Ishlamagan joy bo'lsa, Xatolar jurnalini yuboring: ilovalar menyusi bo'yicha `GESTURE_PATH` yozuvlari (`apps_ok`,
`apps_step_rejected`, `apps_step_no_effect`, `apps_failed`) qaysi yo'l nima uchun yiqilganini ko'rsatadi.

## 5. 0.9.1: Telegram'da yurish, tanlanganlik va yorliq tartibi

### 5.1 Sabablar

Kod o'qishdan aniq (tekshirilgan):

- `NEEDED_EVENTS` ichida `TYPE_VIEW_SELECTED` va `TYPE_VIEW_LONG_CLICKED` yo'q edi, `onClicked` esa faqat `isCheckable`
  elementlarda ishlardi. Tanlash yoki tanlovni olib tashlash hech qachon aytilmasdi; faqat "belgilangan" tomoni ba'zan
  (element belgilanadigan bo'lib qolganda) eshitilardi.
- `stateOf` belgilanadigan elementda `isSelected`ni, ro'yxat elementining tanlanganligini (`collectionItemInfo.isSelected`)
  hisobga olmasdi.
- `NodeDescriber.labelOf` matnni contentDescription'dan oldin olardi. TalkBack aksincha: ilova ekran o'quvchi uchun yozgan
  contentDescription birinchi.
- `labelOf` ichida `val hint = node.hintText...` parametr `hint`ni yopib qo'ygan edi, shuning uchun hodisaning o'z matni
  (4-qadam) hech qachon aytilmasdi.
- `onFocused` manbasiz hodisada (`event.source == null`) hech narsani eslab qolmasdi va `currentFocus()` tizim topmasa eskirgan
  `lastFocusedSource`ni qaytarardi. Natija: keyingi surish eski elementdan hisoblanadi.
- `focusFrom` fokus bermagan elementni jimgina tashlab, keyingisiga o'tardi.
- `handleScrolled` virtual elementda `refresh()` false qaytarsa, fokusni "yo'qolgan" deb ro'yxat boshiga ko'chirardi.
- `isLabeledUnit` bolalari 12 tadan ko'p konteynerni, uzun matni bo'lsa ham, nuqta deb hisoblamasdi.

Taxmin (jurnalda Telegram yozuvi yo'qligi uchun tasdiqlanmagan):

- Javob tariqasida yozilgan xabardagi javob havolasi virtual element; uning fokus hodisasi manbasiz kelganda joriy element
  eskirgan bo'lib, yurish shu yerda qotadi. Tuzatish shu mexanizmga qaratilgan.
- Ovozli xabar tugmasi ikkita: bosiladigan konteyner va ichidagi bosiladigan belgi, ikkalasi bir xil nom bilan alohida nuqta.
- YouTube'dagi "Video": tugmada ham matn, ham contentDescription bor. Aniq daraxt noma'lum.

### 5.2 Kod

Fokus manbasi (ReaderService.onFocused): manbasiz hodisada o'zimiz so'ragan fokus joriy bo'ladi.

```kotlin
val req = focusRequest
if (req != null && nowMs - focusRequestAt < 1500L) {
    lastFocusedSource = req
    lastFocusedAt = nowMs
    rememberState(req)
}
```

Joriy element (ReaderService.currentFocus): tizim ota-onani, biz esa uning ichidagi elementni bildirsak, aniqrog'i olinadi.

```kotlin
if (sys == null) return tracked
if (tracked == null) return sys
if (sys == tracked) return sys
if (isInside(tracked, sys)) return tracked
return sys
```

Fokus bermagan element o'tkazilmaydi (ReaderService.focusOrRead) va qotib qolishdan himoya (moveFocus):

```kotlin
val ok = focus(node) || readWithoutFocus(node)
...
if (fast != null && stuckTarget != null && fast == stuckTarget) fast = navigator.nextUnit(fast, forward, mode)
```

Ikkilangan nuqta (Navigator.isShadow): yorliq bir xil va maydonning 80 foizi ustma-ust.

```kotlin
val smaller = minOf(areaA, areaB)
return smaller > 0L && areaInter * 10L >= smaller * 8L
```

Yorliq tartibi (NodeDescriber.labelOf): contentDescription, matn, hint, tooltip, maxsus yorliq, labelFor, ichidagi matnlar,
hodisa matni, ichidagi bosiladigan bolalar.

Holat o'zgarishi (ReaderService.checkStateChange): bosish, uzoq bosish, tanlash yoki holat tavsifi o'zgargach fokusdagi element
qayta o'qilmaydi, faqat yangi holat aytiladi; tanlov olib tashlansa "tanlanmagan" yoki "belgilanmagan".

### 5.3 Telefonda sinash ro'yxati

1. Telegram kanalida o'ngga surib to'liq yuring: har post matni, keyin "fikrlar" va "ulashish" tugmalari tushib qolmasdan o'qiladimi.
2. Xuddi shunday chapga surib tepaga yuring.
3. Telegram guruhida o'zingiz javob tariqasida yozgan xabarga yetganda yurish davom etadimi, pastdagi yozish maydoniga o'tiladimi.
4. Yozish maydonida ovozli xabar tugmasi bir marta o'qiladimi.
5. Telegram'da bir nechta audio yoki faylni tanlang: har tanlashda "tanlangan" yoki "belgilangan" aytiladimi; olib tashlaganda
   "tanlanmagan" yoki "belgilanmagan".
6. Samsung "Mening fayllarim"da bir nechta faylni tanlang: shunday tekshiring; keyin ro'yxat bo'ylab yurib, tanlanganlar
   "tanlangan" deb o'qiladimi.
7. YouTube'da video pleyer tugmasini toping: "Video" o'rniga tugmaning nomi aytiladimi.
8. Sozlamalar va chatlar ro'yxatida yurish avvalgidek tez, takrorsiz va to'liq ishlayaptimi (yorliq tartibi o'zgargani uchun).

Ishlamagan joy bo'lsa, Xatolar jurnalini yuboring. Yangi yozuvlar: `FOCUS_SOFT_READ` (tizim fokus bermagan element: sinf, ilova,
ko'rinish identifikatori), `FOCUS_STUCK_FIXED` (qotib qolishdan himoya ishga tushgan joy), `FOCUS_NODE_FAILED`.
