# Native Interops in TinyCLR
---
Interops allow you to write a class in managed code that is partially or entirely implemented in native code. This is useful for time critical tasks, things that would take too long in managed code, or interacting with native functionality not exposed through managed code. Keep in mind that while native code executes, all managed threads are blocked and if you crash in native code, managed code also crashes.

To get started, create a TinyCLR project called `InteropTest`. In the project properties window, go to the `TinyCLR OS` tab. Check both the `Generate native stubs for internal methods` and the `Generate bare native stubs` checkboxes. Next, define your native API. Any method that you plan to implement in native code must be declared extern and be decorated with the `System.Runtime.CompilerServices.MethodImpl` attribute that is constructed with `MethodImplOptions.InternalCall`. Static and instance functions, static and instance constructors, finalizers, and property set and get bodies can all be implemented native. They can have any visibility, can take any number or types of parameters, and can return any type. For example:

```csharp
class MyNativeClass {
    private int field = 5;

    [MethodImpl(MethodImplOptions.InternalCall)]
    public extern string MyNativeFunc(uint param1);

    public extern int MyNativeProperty {
        [MethodImpl(MethodImplOptions.InternalCall)]
        get;
    }
}
```

Once you have your native API defined, build your project. In the output folder, find and open `pe` and then `Interop`. In there are three files that let TinyCLR connect the managed methods to the native methods. There are two main files that have the same name as your project. These define the entire API. Importantly, there is an object that has the assembly name, its checksum, and an array of its methods. The remaining file contains function stubs for each native method you need to implement from the `MyNativeClass` class. Each function has a single parameter of type `TinyCLR_Interop_MethodData` that can be found in the `TinyCLR.h` file. This type has two memebers: an opaque stack type that you pass to other interop functions and the [API provider](native_apis.md) that gives you access to the runtime. You can use this API provider to find the interop provider. The interop provider allows you to read and write object fields, read arguments passed to the function, write to reference arguments, set the return value, raise other events, and create new managed objects. The following code shows reading from a field and setting it as the return value of the property:

```cpp
TinyCLR_Result InteropTest_InteropTest_MyNativeClass::MyNativeProperty___I4(const TinyCLR_Interop_MethodData md) {
    auto ip = reinterpret_cast<const TinyCLR_Interop_Provider*>(md.ApiProvider.FindDefault(&md.ApiProvider, TinyCLR_Api_Type::InteropProvider));

    const TinyCLR_Interop_ClrObject* self;
    TinyCLR_Interop_ClrValue ret;
    TinyCLR_Interop_ClrValue field;

    ip->GetThisObject(ip, md.Stack, self);
    ip->GetField(ip, self, InteropTest_InteropTest_MyNativeClass::FIELD___field___I4, field);
    ip->GetReturn(ip, md.Stack, ret);

    ret.Data.Numeric->I4 = field.Data.Numeric->I4;

    return TinyCLR_Result::Success;
}
```

Now you need to compile these files. If you don't have GCC yet, see the [porting guide](intro.md) to find out how to install GCC. To compile using GCC, the easiest way is to use a makefile and a scatterfile. We've provided samples of each below.

The makefile is setup to compile all cpp in the same directory it is and to do using for a Cortex M4 architecture. If you're not on CortexM4, change the `MCU_FLAGS` parameter accordingly. The output file is `InteropTest.bin`. You can change that with the `OUTPUT_NAME` property.

[!code-makefile[makefile](samples/makefile)]

Because TinyCLR can't currently dynamically relocate your code, you'll need to provide the proper base and length values for the RLI region in the scatterfile by changing the `RLI_BASE` and `RLI_LENGTH` placeholders. You can find the RLI region for your device, if it has one, in the device's specifications.

[!code[scatterfile](samples/scatterfile)]

Lastly, make sure that you place `TinyCLR.h` in the folder so that the interop files can see it. You need to use the file that corresponds to the release of the firmware you are running.

To execute the makefile, you'll need to have make installed. You can get it from a toolkit like [MinGW](http://mingw.org/) or, if you're on Windows 10, the Windows Subsystem for Linux. Once you have make installed, just navigate to the folder with the makefile and interop files in a shell and execute `make build`.

> [!Tip]
> If you use the Windows Subsystem for Linux, you'll need to change `del` in the makefile to `rm`.

Once you have a compiled image, look in the map file to find out where the interop definition variable `Interop_InteropTest` (if you're using the default names) got placed. You'll need to pass this address to the managed function that registers the interop. In managed code, add the compiled binary image as a resource and use the `Marshal` class to copy it into the correct location in memory. Then call `System.Runtime.InteropServices.Interop.Add` and pass it the address of the `Interop_InteropTest` object from the map file. You need to do this every time your program runs and before you call any of the native methods in your interop class.

```csharp
var interop = Resources.GetBytes(Resources.BinaryResources.InteropTest);

Marshal.Copy(interop, 0, new IntPtr(0x20016000), interop.Length);

Interop.Add(new IntPtr(0x2001607C));

var cls = new MyNativeClass();
var val = cls.MyNativeProperty;
```
