+++
date = '2026-08-06T13:15:00+08:00'
draft = false
title = 'Practical Frida : Fixing Android Emulator Push Token Checks'
categories = ['Mobile Security']
tags = ['Android', 'Frida', 'Mobile Security', 'React Native']
+++

During a pentest, I found a login flow that worked on a physical phone but failed on an Android 11 emulator.

The application opened normally, but pressing the login button showed this error:

```text
Cannot Login
Missing Notification Token.

Please allow Notifications in the application settings.
```

At first, it looked like a notification permission issue. I checked the emulator settings, but there was no notification permission option for this app, so I had to look at what the application was doing instead.

## The actual problem

Android 11 does not use the Android 13 `POST_NOTIFICATIONS` runtime permission, which explained why I could not find a notification permission switch in the app settings.

Logcat showed a more useful clue:

```text
FirebaseInstanceId: Token retrieval failed: SERVICE_NOT_AVAILABLE
```

After checking the app behavior, I found that the application treated emulator and physical device differently.

The app was using Expo Constants:

```javascript
if (Constants.isDevice) {
  registerForPushNotifications();
}
```

On a real phone, `Constants.isDevice` returned `true`.

On the emulator, it returned `false`, so the app skipped the normal push-token registration path. Later, the login page expected a notification token and failed.

## The Frida idea

Instead of patching the APK or loading a huge bypass script, I changed the one value that controlled this branch.

The goal was simple:

```text
Make the emulator follow the physical-device path.
```

## Making a hook for this case

The important part is not copying a random hook from the internet. First, find the class and method that control the behavior you want to test. I started with the error message, then followed the login flow in the decompiled code. Since the app was checking whether it was running on a real device, `isDevice` was a useful search term: it is a common name for a boolean that distinguishes a physical phone from an emulator. I searched for `isDevice` in JADX, checked the nearby code for the push-token registration call, and found the Expo Constants check. After that, I searched for the class and method names in the decompiled code to confirm the exact Java implementation that Frida needed.

For a quick check, I also listed the running processes and attached Frida to the app:

```bash
frida-ps -Uai
```

The app was using Expo Constants, and the relevant Java class was `expo.modules.constants.ConstantsService`. Its `getIsDevice()` method returned the value used by the JavaScript code. That gave me a small, targeted hook instead of a script that changed lots of unrelated methods.

Create a file called `is-device.js`:

```javascript
'use strict';

Java.perform(function () {
  const ConstantsService = Java.use(
    'expo.modules.constants.ConstantsService'
  );

  ConstantsService.getIsDevice.implementation = function () {
    console.log('[+] Constants.isDevice forced to true');
    return true;
  };

  console.log('[+] Hook loaded');
});
```

The hook replaces the original method implementation and always returns `true`. The `console.log` messages make it easy to confirm that the hook loaded and that the method was actually called. It does not create a fake token; it only makes the app take the same push-registration path as a physical device.

If the class or method is different in another app, change `ConstantsService` and `getIsDevice` to match what you found during that pentest. If the method has overloads, select the right one explicitly with Frida's `.overload(...)` API. The hook should match the app you are testing, not assume every Expo app has the same internals.

## Running it

First, make sure the Frida client and server versions match:

```bash
python -c 'import frida; print(frida.__version__)'
adb -s <DEVICE_SERIAL> shell '/data/local/tmp/frida-server --version'
```

Start Frida server on the rooted emulator:

```bash
adb -s <DEVICE_SERIAL> shell 'su -c "chmod 755 /data/local/tmp/frida-server"'
adb -s <DEVICE_SERIAL> shell 'su -c "nohup /data/local/tmp/frida-server >/data/local/tmp/frida.log 2>&1 &"'
```

Then create `run-hook.py`. This starts the app in a paused state, attaches Frida before the app initializes, loads the hook, and only then resumes the process. Spawning it this way matters because attaching after tapping the login button may be too late; the value could already have been read.

```python
#!/usr/bin/env python3

import pathlib
import frida

DEVICE_ID = '<DEVICE_SERIAL>'
PACKAGE = '<AUTHORIZED_PACKAGE_NAME>'
SCRIPT = pathlib.Path('is-device.js').read_text()

device = frida.get_device(DEVICE_ID, timeout=10)
pid = device.spawn([PACKAGE])
session = device.attach(pid)

script = session.create_script(SCRIPT)
script.load()

device.resume(pid)
print(f'Application resumed as PID {pid}')

input('Press Enter to quit...')
```

Run it with:

```bash
python run-hook.py
```

Expected output:

```text
[+] Hook loaded
[+] Constants.isDevice forced to true
```

I hope this helps :)
