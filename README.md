# Lab_17_MobileSecurity
# Lab 17: Cracker OWASP Uncrackable Android Level 3

## Overview
This guide details the steps to bypass root/tampering checks, disable anti-debug/anti-Frida measures, and extract the hidden password from the native library of the OWASP Uncrackable‑Level3 APK.

## Steps

### 1. Static Analysis with Jadx‑GUI
- Open the APK in Jadx‑GUI.
- Navigate to `sg.vantagepoint.uncrackable3 → MainActivity`.
- Examine:
  - `verifyLibs()` – checks CRC of native `.so` and DEX via native function `baz`. Mismatch sets `tampered = 31337`.
  - `onCreate()` – calls `showDialog()` if root or tampering detected, causing the app to exit.
  - `System.loadLibrary("foo")` – loads `libfoo.so`, where the real password verification resides.
- Confirm the decompiled Java code is visible.

### 2. Decompile the APK with apktool
```bash
apktool d UnCrackable-Level3.apk -o uncrackable3
```
Result:
- `uncrackable3/` contains:
  - `smali/` – Android bytecode.
  - `lib/<arch>/libfoo.so` – native library matching the emulator/device architecture.
- This enables smali patching without a full rebuild.

### 3. Patch Smali – Remove Root/Tampering Dialog
**File**: `uncrackable3/smali/sg/vantagepoint/uncrackable3/MainActivity.smali`

Locate the second occurrence of `showDialog` inside `onCreate()` (around line 126):
```smali
.line 126
invoke-static {}, Lsg/vantagepoint/util/RootDetection;->checkRoot1()Z
move-result v0
if-nez v0, :cond_0
invoke-static {}, Lsg/vantagepoint/util/RootDetection;->checkRoot2()Z
move-result v0
if-nez v0, :cond_0
invoke-static {}, Lsg/vantagepoint/util/RootDetection;->checkRoot3()Z
move-result v0
if-nez v0, :cond_0
invoke-virtual {p0}, Lsg/vantagepoint/uncrackable3/MainActivity;->getApplicationContext()Landroid/content/Context;
move-result-object v0
invoke-static {v0}, Lsg/vantagepoint/util/IntegrityCheck;->isDebuggable(Landroid/content/Context;)Z
move-result v0
if-nez v0, :cond_0
sget v0, Lsg/vantagepoint/uncrackable3/MainActivity;->tampered:I
if-eqz v0, :cond_1

:cond_0
const-string v0, "Rooting or tampering detected."
invoke-direct {p0, v0}, Lsg/vantagepoint/uncrackable3/MainActivity;->showDialog(Ljava/lang/String;)V
.line 130

:cond_1
new-instance v0, Lsg/vantagepoint/uncrackable3/CodeCheck;
```

**Patch options**:
- **Method A (simplest)**: Replace the two lines under `:cond_0` with `return-void`.
  ```smali
  :cond_0
  return-void
  ```
- **Method B (clean jump)**: Replace the whole block with:
  ```smali
  :cond_0
  goto :cond_1
  ```

**Optional**: Neutralize `showDialog` entirely:
```smali
.method private showDialog(Ljava/lang/String;)V
    .locals 3
    return-void
.end method
```
Save the file.

### 4. Rebuild and Sign the APK
```bash
apktool b uncrackable3 -o UnCrackable-Level3-patched.apk
```
Sign using the debug keystore:
```bash
apksigner sign --ks "%USERPROFILE%\.android\debug.keystore" UnCrackable-Level3-patched.apk
```
Install:
```bash
adb install -r UnCrackable-Level3-patched.apk
```
Launch the app; it should open directly to the “Enter the secret code” screen without any tampering popup.

### 5. Patch the Native Library (anti‑debug / anti‑Frida)
1. Copy `libfoo.so` from `uncrackable3/lib/<arch>/`.
2. Load it into Ghidra, create a project, and run auto‑analysis.
3. Locate `sub_73D0` (executed from `.init_array`). This routine:
   - Scans `/proc/self/maps` for strings like “frida” or “exposed”.
   - Calls `ptrace` for anti‑debug.
   - On detection invokes `goodbye()` and crashes.
4. Patch the first instruction of `sub_73D0` to `RET` (Edit → Patch Instruction → `RET`).
5. Export the patched library, overwriting the original in `uncrackable3/lib/<arch>/libfoo.so`.
6. Rebuild, sign, and reinstall the APK (repeat step 4).

### 6. Extract the Password from Native Verification
#### Locate the JNI bridge
In Jadx or Android Studio, view `MainActivity.verify(View view)`:
```java
String string = ((EditText) findViewById(R.id.edit_text)).getText().toString();
if (this.check.check_code(string)) {
    // Success!
}
```
The decision is delegated to a native method.

#### Find the native function in Ghidra
- Symbol Tree → `Java_sg_vantagepoint_uncrackable3_Check_check_code`.
- Follow the call to the core routine `FUN_001012c0`.

#### Analyze `FUN_001012c0`
- The function contains a large repetitive obfuscation block (LCG calculations, repeated `malloc`, linked‑list construction) meant to hinder static analysis.
- Toward the end, three qwords (24 bytes) are written to the input buffer (`param_1`). These bytes constitute the encoded key.

#### Retrieve the constants (little‑endian)
From the memory view at the end of `FUN_001012c0`:
```
1d 08 11 13 0f 17 49 15  0d 00 03 19 5a 1d 13 15
08 0e 5a 00 17 08 13 14
```
These bytes represent the encoded key.

#### Decode the key
The simple XOR cipher uses the repeating string “pizzapizzapizzapizzapizzapizza” (24 bytes).  
Python script:
```python
encoded = bytes.fromhex("1d0811130f1749150d0003195a1d1315080e5a0017081314")
xor_key = b"pizzapizzapizzapizzapizzapizza"
secret = bytes(a ^ b for a, b in zip(encoded, xor_key))
print("Secret:", secret.decode())
```
Output:
```
making owasp great again
```
Enter this string in the app; the verification succeeds and a “Success!” message appears.

## Summary
- `check.check_code()` is a JNI wrapper forwarding the user‑supplied string to the native routine.
- `FUN_001012c0` performs extensive obfuscation but ultimately writes a 24‑byte buffer that, after XOR with a known constant, yields the password.
- Observed obfuscation markers: linear‑congruent generator loops, repeated heap allocations, and a dummy linked list.
- The final buffer comparison is the true verification logic; defeating the anti‑debug/anti‑Frida checks in `.init_array` allows the native code to run unhindered.

---  
*End of guide.*  
