# Al-Madina General Store Management System (Flutter)

A complete offline-first store app: **POS billing, inventory, customer khata (udhaar), PDF invoices, WhatsApp reminders and real Google Drive backup / restore.**
Data is saved in a local **SQLite** database first, so everything works without internet.

> **Honest note:** this code was written without a Flutter SDK available, so it has **not been compiled or run yet**.
> Follow the steps below; if `flutter analyze` or `flutter run` shows any error, copy the message and send it to me - I will fix it.
> Google Drive needs *your* Google Cloud setup (Step 3) - no app can do that for you.

---
## 1. What you need (one time)
1. Install **Flutter** (https://docs.flutter.dev/get-started/install) and **Android Studio** (for the Android SDK).
2. Run `flutter doctor` and fix anything marked with an X.

## 2. Create the project files and run
Unzip this folder, open a terminal inside it and run:

```bash
flutter create --org com.almadina --project-name al_madina_store .
flutter pub get
```
`flutter create .` adds the missing `android/` folder. It will NOT overwrite `lib/` or `pubspec.yaml`.

### Android settings (edit 2 files)
**a) `android/app/src/main/AndroidManifest.xml`** - add these lines just inside `<manifest ...>` (above `<application>`):
```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.CAMERA"/>
<queries>
  <intent><action android:name="android.intent.action.VIEW"/><data android:scheme="https"/></intent>
  <intent><action android:name="android.intent.action.VIEW"/><data android:scheme="whatsapp"/></intent>
  <intent><action android:name="android.intent.action.DIAL"/><data android:scheme="tel"/></intent>
</queries>
```
**b) `android/app/build.gradle` (or `build.gradle.kts`)** - make sure `minSdk` is **at least 21** (the scanner needs it). Example: `minSdk = 21` (or `minSdkVersion 21`).

Run on a phone or emulator:
```bash
flutter run
```
On the very first start the app loads **sample data** (so you can test). Remove it in *Avatar icon -> Settings -> Remove ALL data*.

## 3. Google Drive setup (needed only for backup / restore)
1. Go to https://console.cloud.google.com and create a project.
2. **APIs & Services -> Library -> Google Drive API -> Enable.**
3. **OAuth consent screen:** choose *External*, fill the app name and your email. Add your Gmail address under **Test users**.
4. **Credentials -> Create credentials -> OAuth client ID -> Android.**
   * Package name: `com.almadina.al_madina_store`
   * SHA-1 for the APK built here: `D6:C7:42:F0:DD:74:86:1F:E3:0E:13:89:5A:CD:D6:A6:23:F0:FB:75`
   * This release APK currently uses the Android debug signing key. If you sign the APK with another key, register that key's SHA-1 instead (or add it as another Android OAuth client).
5. Run the app -> *Settings -> Connect Google Drive*. Sign in with a test user.
   * Add the Gmail account under **OAuth consent screen -> Test users** while the consent screen is in testing.
No API key or client secret is stored in the app. A `google-services.json` file is not required for this Android-only Google Sign-In setup. The app asks only for the `drive.file` permission, so it can see **only its own backup folder** ("AlMadina Store Backups").

## 4. Build the APK
```bash
flutter build apk --release                 # one big APK
flutter build apk --split-per-abi           # smaller APKs per phone type
```
File: `build/app/outputs/flutter-apk/app-release.apk` - copy it to the phone and install.
(For Google sign-in, make sure the OAuth Android client matches the package name and SHA-1 of the APK you install.)

---
## 5. Folder structure
```
lib/
  main.dart                     start of the app
  theme/app_theme.dart          colors + dark theme
  utils/                        money/date formatting, helpers, validators
  models/                       Product, Customer, Sale, LedgerEntry, settings ...
  database/
    db_helper.dart              creates the SQLite tables, backup file helpers
    repository.dart             ALL database queries + business rules
    sample_data.dart            optional test data
  services/
    pdf_service.dart            invoice / ledger / report PDFs, share, print
    drive_service.dart          real Google sign-in + Drive API
  providers/
    store_provider.dart         app data + POS cart (Provider)
    backup_provider.dart        backup / restore / auto-backup (Provider)
  widgets/                      reusable buttons, cards, nav bar, header
  screens/
    dashboard/ pos/ inventory/ customers/ ledger/ settings/
```
Screens never touch SQLite directly: **screen -> provider -> repository -> SQLite**.

## 6. How the important parts work (simple words)
* **Sale (POS):** pick products (search, barcode scanner) -> cart -> customer -> discount -> *Complete Sale*.
  Saving is ONE database transaction: sale + items + stock reduction + customer balance + invoice number. If anything fails, nothing is saved.
  Not enough stock = the sale is blocked.
* **Formulas:** `Grand Total = Subtotal - Discount` - `New Balance = Previous Balance + Credit Sales - Payments` -
  `Stock = Previous - Sold + Purchased` - `Profit = (Selling - Purchase) x Qty` (the discount is also subtracted from profit so it matches your real money).
* **Customer ledger:** shows bills, payments and credit adjustments in date order with a running balance. *You Got* reduces the balance, *You Gave* increases it.
* **Dashboard:** every number is calculated from SQLite and refreshes after every sale, payment or stock change. Pull down to refresh.
* **Backup:** "Backup Now" copies the database file, checks it is healthy, uploads it, and verifies the uploaded size (a half-uploaded file is deleted). Only the newest 5 backups are kept.
* **Auto backup:** after changes (every change / 15 min / hourly / daily - you choose). Offline? The app keeps working and retries later.
* **Conflict rule (several phones):** every change stamps a "last modified" time that is saved with each backup.
  Backup is **refused** if Drive already has a newer backup from another phone (restore first, or force). Restore **warns** if this phone has newer data. Older data never silently overwrites newer data.
* **Restore:** downloads the backup, checks it, then replaces the local database and reloads all screens.

## 7. Quick test checklist
1. Open the app -> Dashboard shows numbers from the sample data.
2. POS: search "Tapal", add items, discount, choose a customer, *Credit*, Complete Sale -> invoice dialog -> Share PDF.
3. Customers: Total Udhaar went up; open the customer -> new bill in the ledger -> *You Got (Cash)* -> balance goes down.
4. Inventory: add product, add/remove stock, open history, try removing more than available (blocked).
5. Ledger tab: Daily / Monthly report PDF.
6. Settings: connect Drive -> Backup Now -> Restore.
7. Settings: Remove ALL data -> start entering your real products and customers.
<img width="573" height="1280" alt="image" src="https://github.com/user-attachments/assets/98bcb42b-9dce-422e-90ff-86826aef4cc4" />
<img width="573" height="1280" alt="image" src="https://github.com/user-attachments/assets/519aece1-d4cf-404d-81e7-7ef139a20705" />
<img width="573" height="1280" alt="image" src="https://github.com/user-attachments/assets/1cafc433-b629-42ad-8d38-af4a5f92f81a" />
<img width="573" height="1280" alt="image" src="https://github.com/user-attachments/assets/a0e369de-e495-4def-a4d2-e55ec614981c" />
<img width="573" height="1280" alt="image" src="https://github.com/user-attachments/assets/51d47f5a-1813-41c1-a9d2-7beba143eec0" />


## 8. Known limits
* Barcode scanning needs a camera; a keyboard entry button is included.
* Sample data is test data only - remove it before real use.
* Single store database; merging two phones that edited at the same time is **not** automatic (the conflict rule protects you from overwriting, you choose which version wins).

