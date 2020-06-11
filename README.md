# Files API in Android 11
So basically, _Files API works with scoped storage in Android 11._ They made it so to maintain compatibility with 3rd party libs and for our convenience. I will be performing broad file access test on Android 11 using the Files API on non-media files.

### The new `getDirectory()` method in StorageVolume
In Android 11, a new method called `getDirectory()` is introduced which returns the directory where this volume is currently mounted as `File`. However, according to documentation, "Direct filesystem access via this path has significant emulation overhead, and apps are instead strongly encouraged to interact with media on storage volumes via the MediaStore APIs.". But at the same time, they encourages us to try out both the APIs to see what works best. I will focus on `Files` API here.

### Clarification with storage names
Storage naming has always been confusing in Android. So to clarify this, I will be using 2 terms:
- __Internal storage__ - This is the non-removable internal shared storage, which comes with the device. Please don't confuse it with "app's internal storage" (which is meant for private app specific data). Simply put, when you download something from internet, the file usually gets saved in the Downloads folder present in this location.
- __SD card__ - This is the removable external SD card, something which users can buy and put in the external storage slot which a device may support.

### Test environment
I will be performing the test on the emulator which runs the latest Android R developer preview (not the beta, unfortunately, I will update the results as I will be performing the same on beta). I have `test.zip` in the `Downloads` folder in the internal storage (`/storage/emulated/0/Downloads/test.zip`). I also have a `test.zip` in root of internal storage (`/storage/emulated/0/test.zip`).

### Storage detection
In the upcoming sections, I will be performing tests using `READ_EXTERNAL_STORAGE` permission and `ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION`. Storage detection is done using `StorageVolume` API. And I will be using a recursive file tree walk implementation to search for zip files.
```kotlin
val storageManager = getSystemService(Context.STORAGE_SERVICE) as StorageManager
val storageVolumes = storageManager.storageVolumes

for (v in storageVolumes) {
    Log.d("Files_Test", "Volume = $v")

    val x = ArrayList<String>()

    // Recursive file tree walk impl
    // v.directory is introduced in Android 11
    // This also logs each folder it is traversing
    searchFiles(v.directory, x, ".zip")

    Log.d("Files_Test", x.toString())
}
```

### Using permission `READ_EXTERNAL_STORAGE`
It is able to list all the directories under both internal storage and the emulated sd card. However, __it is not able to detect the test files present in the internal storage (both the zip files, that is, one present in Downloads and the other present in root)__. Of course I have declared the permission in manifest and requested for it in runtime.
Interestingly, it is not able read directories inside `Android/data` in internal storage, which is an expected behavior, BUT, it is able to read directories inside `Android/data` in the sd card, which is strange as Google says that under any circumstances, apps will not be able to read this folder. Perhaps this is a bug in dev preview, I will update this again with beta test results.

### Using the `ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION`
This is the recommended way to perform broad file access in Android 11. For this we need the special `ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION` access. We need to declare `MANAGE_EXTERNAL_STORAGE` permission in manifest, and require access to all files access.
```kotlin
val intent = Intent(Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION)

// According to docs, we need to set the intent's data with the app package name in this format
// Please let me know if there is a better way to do this
intent.data = Uri.parse("package:$packageName")

startActivityForResult(intent, REQ_CODE)
```

After granting the access, __the test works as expected, as it is able to detect both of the files__.

### Bonus! Write operation in external SD card
If I am not wrong, performing write operations on external SD card starting with Android KitKat, needed the intervention of the Storage Access Framework, where users were required to select the root of sd card before things could proceed. Without these, write operations failed in SD card.
However, in Android 11, __I am able to programmatically create files in the external storage, which gets picked up by my search implementation, and even delete them!__, without the SAF intervention. Isn't this great?
I was also able to do the same on internal storage.

### Now back to testing with `READ_EXTERNAL_STORAGE`
As I have found previously that I was able to programmatically create files on internal storage and SD card using Files API, I created one, and then switched over to `READ_EXTERNAL_STORAGE` permissions.
It now no longer detects the files I created programmatically in internal storage, __but it is able to detect them in SD card__.

### Takeaways
- Files API works in Android 11, but access is limited by scoped storage.
- For broad files access, `ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION` is needed. Files API works with it.
- For broad files access, `MANAGE_EXTERNAL_STORAGE` permission needs to be declared in manifest. Without doing so, your app will be able to take the user to screen where he/she needs to enable "All files access", but the switch will be disabled, basically making the user unable to enable it.
- Google will manually review apps declaring `MANAGE_EXTERNAL_STORAGE`, and may reject if they feel if the reason for using broad files access is not strong enough.
- If the write operation on SD card without SAF intervention is an intended feature and not a bug, then this would result in a better UX.
