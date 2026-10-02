# LearkPeN

**Developed by learkcompany.**

LearkPeN is a learkcompany project. The LearkPeN application and its original project code are attributed to learkcompany. Third-party components retain their own copyrights and license terms.

Use an iPad and Apple Pencil to control and write in a Windows workspace over USB.

## Release v0.0.10

This repository publishes the iPad source code. The Windows installer and portable ZIP belong in GitHub Releases.

The iPad app remains v0.0.9; Windows v0.0.10 accepts that protocol. This release updates the Windows companion. The application version is unchanged; source headers identify learkcompany.

## Run on iPad

1. Create an App project in Swift Playgrounds named LearkPeN.
2. Replace ContentView with the contents of [iPad/ContentView.swift](iPad/ContentView.swift). [ContentView.txt](iPad/ContentView.txt) contains the same code for copying.
3. The supplied file includes an @main App entry point. Remove the template MyApp.swift entry point, so the project contains exactly one @main.
4. Run the app, choose a language, and start the USB connection.
5. Connect a USB data cable, unlock the iPad, and accept Trust if requested.
6. Open the Windows companion and enter the two-digit code shown on iPad.

## Features

- Native desktop video over USB.
- Apple Pencil control, stroke smoothing, and separate touch gestures.
- English / Persian, light / dark / system appearance.
- Pencil double tap toggles pen control on supported hardware.
- Automatically hiding toolbar.

## Requirements and limitations

The Windows companion requires compatible NVIDIA NVENC/Vulkan hardware and Apple USB support (Apple Devices or supported iTunes installation). It uses a bundled Node.js/FFmpeg runtime. Core binaries are bundled; missing Apple support may require an online installation.

In mouse mode, Pencil pressure does not change the Windows application's brush thickness. This app does not install a virtual pressure-sensitive tablet driver.

Liquid Glass requires a compatible SDK/compiler and iPadOS version; older toolchains use the material fallback. The publication copy has not been compiled or reviewed on iPad in this environment. Desktop rendering, Windows package checks, and device tests cover different parts of the system; a stable 60 FPS end-to-end result is not guaranteed on every machine.

## Source availability

The iPad source is publicly viewable. No open-source license has been selected yet; publication alone does not grant an open-source license.

## فارسی

**LearkPeN محصول learkcompany است.** نام سازنده در معرفی پروژه و سربرگ کد اصلی درج شده است. حقوق و مجوزهای کتابخانه‌ها و اجزای شخص ثالث متعلق به صاحبان همان اجزا است.

کد آیپد عمومی است. نصب‌کننده و ZIP ویندوز در بخش Releases منتشر می‌شوند. نسخهٔ آیپد ۰.۰.۹ است و با نسخهٔ ویندوز ۰.۰.۱۰ کار می‌کند. فایل Swift شامل @main است؛ ورودی MyApp پروژهٔ پیش‌فرض را حذف کن تا فقط یک @main باقی بماند.
