# LLVMSharp

[![Join the chat at https://gitter.im/mjsabby/LLVMSharp](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/mjsabby/LLVMSharp?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

LLVMSharp are strongly-typed safe LLVM bindings written in C# for .NET and Mono, tested on Linux and Windows. They are auto-generated using [ClangSharp](http://www.clangsharp.org) parsing LLVM-C header files.

If you're on Windows, consider using the [**LLVMSharp 3.8 NuGet Package**](http://www.nuget.org/packages/LLVMSharp/3.8.0) - built from LLVM 3.8 Release.

## Building LLVMSharp

On Linux using Mono:

```bash
 $ git clone http://github.com/Microsoft/LLVMSharp
 $ cd LLVMSharp
 $ chmod +x build.sh
 $ ./build.sh /path/to/libLLVM.so /path/llvm/include
```

On Windows using Microsoft.NET:

**Note:** - you need to run from the Visual Studio Command Prompt of the architecture you want to target.

```bash
 :> cd c:\path\to\llvm_source\{Release|Debug}\lib
 :> git clone http://github.com/Microsoft/LLVMSharp
 :> powershell ./LLVMSharp/tools/GenLLVMDLL.ps1
 :> build.bat C:\path\llvm.dll C:\path\to\llvm\include
```

## Features

 * Auto-generated using LLVM C headers files, and supports all functionality exposed by them (more than enough to build a full compiler)
 * Type safe (LLVMValueRef and LLVMTypeRef are different types, despite being pointers internally)
 * Nearly identical to LLVM C APIs, e.g. LLVMModuleCreateWithName in C, vs. LLVM.ModuleCreateWithName (notice the . in the C# API)

## Kaleidoscope Tutorial

Much of the tutorial is already implemented here, and has some nice improvements like the Visitor pattern for code generation to make the LLVM code stand out and help you bootstrap your compiler.

The tutorials have been tested to run on Windows and Linux, however the build (using MSBuild) uses the Nuget packages, hence require some editing to run on Linux.

[Chapter 3](https://github.com/mjsabby/LLVMSharp/tree/master/KaleidoscopeTutorial/Chapter3)

[Chapter 4](https://github.com/mjsabby/LLVMSharp/tree/master/KaleidoscopeTutorial/Chapter4)

[Chapter 5](https://github.com/mjsabby/LLVMSharp/tree/master/KaleidoscopeTutorial/Chapter5)

## Conventions

* Types are exactly how they are defined in the C bindings, for example: LLVMTypeRef

* Functions are put in a C# class called LLVM and the LLVM prefix is removed from the functions, for example: LLVM.ModuleCreateWithName("LLVMSharpIntro");

* For certain functions requiring a pointer to an array, you must pass the array indexed into its first element. If you do not want to pass any element, you can either pass an array with a dummy element, or make a single type and pass it, otherwise you won't be able to compile, for example: LLVM.FunctionType(LLVM.Int32Type(), out typesArr[0], 0, False); is equivalent to LLVM.FunctionType(LLVM.Int32Type(), out type, 0, False);

## Example application

```csharp
using System;
using System.Runtime.InteropServices;
using LLVMSharp;

internal sealed class Program
{
    [UnmanagedFunctionPointer(CallingConvention.Cdecl)]
    public delegate int Add(int a, int b);

    private static void Main(string[] args)
    {
        LLVMBool False = new LLVMBool(0);
        LLVMModuleRef mod = LLVM.ModuleCreateWithName("LLVMSharpIntro");

        LLVMTypeRef[] param_types = {LLVM.Int32Type(), LLVM.Int32Type()};
        LLVMTypeRef ret_type = LLVM.FunctionType(LLVM.Int32Type(), out param_types[0], 2, False);
        LLVMValueRef sum = LLVM.AddFunction(mod, "sum", ret_type);

        LLVMBasicBlockRef entry = LLVM.AppendBasicBlock(sum, "entry");

        LLVMBuilderRef builder = LLVM.CreateBuilder();
        LLVM.PositionBuilderAtEnd(builder, entry);
        LLVMValueRef tmp = LLVM.BuildAdd(builder, LLVM.GetParam(sum, 0), LLVM.GetParam(sum, 1), "tmp");
        LLVM.BuildRet(builder, tmp);

        IntPtr error;
        LLVM.VerifyModule(mod, LLVMVerifierFailureAction.LLVMAbortProcessAction, out error);
        LLVM.DisposeMessage(error);

        LLVMExecutionEngineRef engine;

        LLVM.LinkInMCJIT();
        LLVM.InitializeNativeTarget();
        LLVM.InitializeNativeAsmPrinter();

        var options = new LLVMMCJITCompilerOptions();
        var optionsSize = (4*sizeof (int)) + IntPtr.Size; // LLVMMCJITCompilerOptions has 4 ints and a pointer

        LLVM.InitializeMCJITCompilerOptions(out options, optionsSize);
        LLVM.CreateMCJITCompilerForModule(out engine, mod, out options, optionsSize, out error);

        var addMethod = (Add) Marshal.GetDelegateForFunctionPointer(LLVM.GetPointerToGlobal(engine, sum), typeof (Add));
        int result = addMethod(10, 10);

        Console.WriteLine("Result of sum is: " + result);

        if (LLVM.WriteBitcodeToFile(mod, "sum.bc") != 0)
        {
            Console.WriteLine("error writing bitcode to file, skipping");
        }

        LLVM.DumpModule(mod);

        LLVM.DisposeBuilder(builder);
        LLVM.DisposeExecutionEngine(engine);
        Console.ReadKey();
    }
}
````

## Microsoft Open Source Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.
