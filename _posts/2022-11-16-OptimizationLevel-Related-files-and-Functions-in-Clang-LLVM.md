---
layout: post
title: OptimizationLevel-Related-files-and-Functions-in-Clang-LLVM
date: 2022-11-16
updated: 2022-11-16
categories: Tech
tags: [Essay，Tech]
description: 最近一直在调试LLVM的优化选项和相关的PassPipeline，就对Clang和LLVM中跟OptimizationLevel相关的文件和函数进行了梳理，在此总结一下，希望为以后的工作提供一些借鉴。
---

## 1.2.1 Clang Related Files & Functions

CodeGenOpts.OptimizationLevel和CodeGenOpts.OptimizeSize的取值范围分别是0到3和0到2，定义可以参考1.2.1.1节介绍。

而CodeGenOpt::Level，则是函数getCGOptLevel()的返回类型，函数getCGOptLevel()在runThinLTOBackend()和CreateTargetMachine()中被调用。

而OptimizationLevel，又是函数mapToLevel()的返回类型（与CodeGenOpts.OptimizationLevel不同），而是将CodeGenOpts.OptimizationLevel和CodeGenOpts.OptimizeSize进行映射，分别对应OptimizationLevel::O0、OptimizationLevel::O1、OptimizationLevel::O2、OptimizationLevel::O3、OptimizationLevel::Oz、OptimizationLevel::Os，定义可以参考1.2.1.1节介绍。

在函数addSanitizers()、EmitAssemblyHelper::RunOptimizationPipeline()中，均使用的是OptimizationLevel::Ox，调用函数mapToLevel()。

EmitAssemblyHelper::RunOptimizationPipeline()在EmitAssemblyHelper::EmitAssembly()中被调用。

### 1.2.1.1 CodeGenOptions和CodeGenOpt
/// CodeGenOptions - Track various options which control how the code is optimized and passed to the backend.
两个子类，OptimizationLevel和OptimizeSize，定义在llvm-project-llvmorg-13.0.1\clang\include\clang\Basic\CodeGenOptions.def文件中。

```
VALUE_CODEGENOPT(Name, Bits, Default)

VALUE_CODEGENOPT(OptimizationLevel, 2, 0) ///< The -O[0-3] option specified.
VALUE_CODEGENOPT(OptimizeSize, 2, 0) ///< If -Os (==1) or -Oz (==2) is specified.
```

```
static CodeGenOpt::Level getCGOptLevel(const CodeGenOptions &CodeGenOpts) {
  switch (CodeGenOpts.OptimizationLevel) {
  default:
    llvm_unreachable("Invalid optimization level!");
  case 0:
    return CodeGenOpt::None;
  case 1:
    return CodeGenOpt::Less;
  case 2:
    return CodeGenOpt::Default; // O2/Os/Oz
  case 3:
    return CodeGenOpt::Aggressive;
  }
}
```

### 1.2.1.2 mapToLevel()
函数定义在llvm-project-llvmorg-13.0.1\clang\lib\CodeGen\BackendUtil.cpp中。

```
static OptimizationLevel mapToLevel(const CodeGenOptions &Opts) {
593  switch (Opts.OptimizationLevel) {
594  default:
595    llvm_unreachable("Invalid optimization level!");
596
597  case 0:
598    return OptimizationLevel::O0;
599
600  case 1:
601    return OptimizationLevel::O1;
602
603  case 2:
604    switch (Opts.OptimizeSize) {
605    default:
606      llvm_unreachable("Invalid optimization level for size!");
607
608    case 0:
609      return OptimizationLevel::O2;
610
611    case 1:
612      return OptimizationLevel::Os;
613
614    case 2:
615      return OptimizationLevel::Oz;
616    }
617
618  case 3:
619    return OptimizationLevel::O3;
620  }
621}
```

### 1.2.1.3 Clang相关的文件

```
./clang/unittests/CodeGen/TBAAMetadataTest.cpp:    CGOpts.OptimizationLevel = 1;
./clang/lib/CodeGen/CGStmt.cpp:  if (!Count && CGM.getCodeGenOpts().OptimizationLevel)
./clang/lib/CodeGen/CGStmt.cpp:    if (!Weights && CGM.getCodeGenOpts().OptimizationLevel)
./clang/lib/CodeGen/CGStmt.cpp:    if (!Weights && CGM.getCodeGenOpts().OptimizationLevel)
./clang/lib/CodeGen/CGStmt.cpp:  if (!Weights && CGM.getCodeGenOpts().OptimizationLevel)
./clang/lib/CodeGen/CGStmt.cpp:      CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/CodeGen/CGStmt.cpp:  } else if (CGM.getCodeGenOpts().OptimizationLevel) {
./clang/lib/CodeGen/CGStmt.cpp:  if (Call && CGM.getCodeGenOpts().OptimizationLevel != 0) {
./clang/lib/CodeGen/BackendUtil.cpp:  switch (CodeGenOpts.OptimizationLevel) {
./clang/lib/CodeGen/BackendUtil.cpp:  if (CodeGenOpts.OptimizationLevel > 0)
./clang/lib/CodeGen/BackendUtil.cpp:static OptimizationLevel mapToLevel(const CodeGenOptions &Opts) {
./clang/lib/CodeGen/BackendUtil.cpp:  switch (Opts.OptimizationLevel) {
./clang/lib/CodeGen/BackendUtil.cpp:    return OptimizationLevel::O0;
./clang/lib/CodeGen/BackendUtil.cpp:    return OptimizationLevel::O1;
./clang/lib/CodeGen/BackendUtil.cpp:      return OptimizationLevel::O2;
./clang/lib/CodeGen/BackendUtil.cpp:      return OptimizationLevel::Os;
./clang/lib/CodeGen/BackendUtil.cpp:      return OptimizationLevel::Oz;
./clang/lib/CodeGen/BackendUtil.cpp:    return OptimizationLevel::O3;
./clang/lib/CodeGen/BackendUtil.cpp:                                         OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:        if (Level != OptimizationLevel::O0) {
./clang/lib/CodeGen/BackendUtil.cpp:             /*DisableOptimization=*/CodeGenOpts.OptimizationLevel == 0}));
./clang/lib/CodeGen/BackendUtil.cpp:    OptimizationLevel Level = mapToLevel(CodeGenOpts);
./clang/lib/CodeGen/BackendUtil.cpp:          [](ModulePassManager &MPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:            if (Level != OptimizationLevel::O0)
./clang/lib/CodeGen/BackendUtil.cpp:          [](ModulePassManager &MPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:            if (Level != OptimizationLevel::O0)
./clang/lib/CodeGen/BackendUtil.cpp:          [](FunctionPassManager &FPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:            if (Level != OptimizationLevel::O0)
./clang/lib/CodeGen/BackendUtil.cpp:          [](ModulePassManager &MPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:          [](ModulePassManager &MPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:          [](ModulePassManager &MPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:          [](FunctionPassManager &FPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:          [Options](ModulePassManager &MPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:          [Options](ModulePassManager &MPM, OptimizationLevel Level) {
./clang/lib/CodeGen/BackendUtil.cpp:    if (CodeGenOpts.OptimizationLevel == 0) {
./clang/lib/CodeGen/BackendUtil.cpp:  Conf.OptLevel = CGOpts.OptimizationLevel;
./clang/lib/CodeGen/CGStmtOpenMP.cpp:  if (CGM.getCodeGenOpts().OptimizationLevel != 0) {
./clang/lib/CodeGen/CGObjC.cpp:    CGM.getCodeGenOpts().OptimizationLevel != 0);
./clang/lib/CodeGen/CGObjC.cpp:    } else if (CGF.CGM.getCodeGenOpts().OptimizationLevel == 0) {
./clang/lib/CodeGen/CGObjC.cpp:  if (CGF.CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/CodeGen/CGObjC.cpp:  if (CGM.getCodeGenOpts().OptimizationLevel == 0) {
./clang/lib/CodeGen/CGObjC.cpp:      CGM.getCodeGenOpts().OptimizationLevel == 0) {
./clang/lib/CodeGen/CGLoopInfo.cpp:  if (CGOpts.OptimizationLevel > 0)
./clang/lib/CodeGen/CodeGenFunction.cpp:  return CGOpts.OptimizationLevel != 0;
./clang/lib/CodeGen/CodeGenFunction.cpp:      if (CGM.getCodeGenOpts().OptimizationLevel == 0)
./clang/lib/CodeGen/CodeGenFunction.cpp:  if (Call && CGM.getCodeGenOpts().OptimizationLevel != 0) {
./clang/lib/CodeGen/CodeGenFunction.cpp:    if (CGM.getCodeGenOpts().OptimizationLevel == 0)
./clang/lib/CodeGen/CodeGenTBAA.cpp:  if (CodeGenOpts.OptimizationLevel == 0 || CodeGenOpts.RelaxedAliasing)
./clang/lib/CodeGen/CGVTables.cpp:    return CGM.getCodeGenOpts().OptimizationLevel && !IsUnprototyped;
./clang/lib/CodeGen/CGVTables.cpp:  return CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/CodeGen/CGVTables.cpp:        assert((def || CodeGenOpts.OptimizationLevel > 0 ||
./clang/lib/CodeGen/CGVTables.cpp:        if (!def && CodeGenOpts.OptimizationLevel > 0)
./clang/lib/CodeGen/CGClass.cpp:        CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/CodeGen/CGClass.cpp:        CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/CodeGen/CGClass.cpp:          CGM.getCodeGenOpts().OptimizationLevel > 0)
./clang/lib/CodeGen/CGClass.cpp:  if (CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/CodeGen/CGClass.cpp:  if (CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/CodeGen/CGClass.cpp:  if (CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/CodeGen/CGCall.cpp:    if (CGF.CGM.getCodeGenOpts().OptimizationLevel != 0 &&
./clang/lib/CodeGen/CGCall.cpp:  if (CGM.getCodeGenOpts().OptimizationLevel != 0 &&
./clang/lib/CodeGen/CodeGenFunction.h:    return CGM.getCodeGenOpts().OptimizationLevel == 0;
./clang/lib/CodeGen/CGDecl.cpp:  if (CGM.getCodeGenOpts().OptimizationLevel == 0)
./clang/lib/CodeGen/CGDecl.cpp:          if (CGM.getCodeGenOpts().OptimizationLevel == 0) {
./clang/lib/CodeGen/CGExpr.cpp:  } else if (CGM.getCodeGenOpts().OptimizationLevel > 0)
./clang/lib/CodeGen/CGExpr.cpp:    if (CGM.getCodeGenOpts().OptimizationLevel > 0) {
./clang/lib/CodeGen/CGExpr.cpp:      if (CGM.getCodeGenOpts().OptimizationLevel > 0) {
./clang/lib/CodeGen/CGExpr.cpp:  if (!CGM.getCodeGenOpts().OptimizationLevel || !TrapBB) {
./clang/lib/CodeGen/CGCXX.cpp:  if (getCodeGenOpts().OptimizationLevel == 0)
./clang/lib/CodeGen/CGExprScalar.cpp:  if (CGF.CGM.getCodeGenOpts().OptimizationLevel > 0)
./clang/lib/CodeGen/CodeGenModule.cpp:      (!CodeGenOpts.RelaxedAliasing && CodeGenOpts.OptimizationLevel > 0))
./clang/lib/CodeGen/CodeGenModule.cpp:  if (CodeGenOpts.OptimizationLevel > 0 && CodeGenOpts.StrictVTablePointers) {
./clang/lib/CodeGen/CodeGenModule.cpp:      !CodeGenOpts.DisableO0ImplyOptNone && CodeGenOpts.OptimizationLevel == 0;
./clang/lib/CodeGen/CodeGenModule.cpp:  if (CodeGenOpts.OptimizationLevel == 0 && !F->hasAttr<AlwaysInlineAttr>())
./clang/lib/CodeGen/CodeGenModule.cpp:  return CodeGenOpts.OptimizationLevel > 0;
./clang/lib/CodeGen/CGBuiltin.cpp:          if (CGM.getCodeGenOpts().OptimizationLevel != 0)
./clang/lib/CodeGen/CGBuiltin.cpp:    if (CGM.getCodeGenOpts().OptimizationLevel == 0)
./clang/lib/CodeGen/CGBuiltin.cpp:    if (CGM.getCodeGenOpts().OptimizationLevel == 0)
./clang/lib/CodeGen/CGBlocks.cpp:               CGM.getCodeGenOpts().OptimizationLevel != 0) {
./clang/lib/CodeGen/CGBlocks.cpp:        CGM.getCodeGenOpts().OptimizationLevel != 0) {
./clang/lib/CodeGen/CGBlocks.cpp:  if (CGM.getCodeGenOpts().OptimizationLevel == 0) {
./clang/lib/CodeGen/CGBlocks.cpp:      if (CGM.getCodeGenOpts().OptimizationLevel == 0) {
./clang/lib/CodeGen/CGBlocks.cpp:    if (CGF.CGM.getCodeGenOpts().OptimizationLevel == 0) {
./clang/lib/CodeGen/CGDeclCXX.cpp:  if (!CGM.getCodeGenOpts().OptimizationLevel)
./clang/lib/CodeGen/ItaniumCXXABI.cpp:    if (CGM.getCodeGenOpts().OptimizationLevel > 0 &&
./clang/lib/Frontend/CompilerInvocation.cpp:static unsigned getOptimizationLevel(ArgList &Args, InputKind IK,
./clang/lib/Frontend/CompilerInvocation.cpp:static unsigned getOptimizationLevelSize(ArgList &Args) {
./clang/lib/Frontend/CompilerInvocation.cpp:  if (Opts.OptimizationLevel == 0)
./clang/lib/Frontend/CompilerInvocation.cpp:    GenerateArg(Args, OPT_O, Twine(Opts.OptimizationLevel), SA);
./clang/lib/Frontend/CompilerInvocation.cpp:  if (Opts.OptimizationLevel > 0) {
./clang/lib/Frontend/CompilerInvocation.cpp:  if (Opts.UnrollLoops && Opts.OptimizationLevel <= 1)
./clang/lib/Frontend/CompilerInvocation.cpp:  else if (!Opts.UnrollLoops && Opts.OptimizationLevel > 1)
./clang/lib/Frontend/CompilerInvocation.cpp:  unsigned OptimizationLevel = getOptimizationLevel(Args, IK, Diags);
./clang/lib/Frontend/CompilerInvocation.cpp:  if (OptimizationLevel > MaxOptLevel) {
./clang/lib/Frontend/CompilerInvocation.cpp:    OptimizationLevel = MaxOptLevel;
./clang/lib/Frontend/CompilerInvocation.cpp:  Opts.OptimizationLevel = OptimizationLevel;
./clang/lib/Frontend/CompilerInvocation.cpp:  if (Opts.OptimizationLevel == 0) {
./clang/lib/Frontend/CompilerInvocation.cpp:  if (Opts.OptimizationLevel > 0 && Opts.hasReducedDebugInfo() &&
./clang/lib/Frontend/CompilerInvocation.cpp:  Opts.OptimizeSize = getOptimizationLevelSize(Args);
./clang/lib/Frontend/CompilerInvocation.cpp:                   (Opts.OptimizationLevel > 1));
./clang/lib/Frontend/CompilerInvocation.cpp:  unsigned Opt = getOptimizationLevel(Args, IK, Diags),
./clang/lib/Frontend/CompilerInvocation.cpp:       OptSize = getOptimizationLevelSize(Args);
./clang/lib/Driver/ToolChain.cpp:  if (!isOptimizationLevelFast(Args)) {
./clang/lib/Driver/ToolChains/Hexagon.cpp:unsigned HexagonToolChain::getOptimizationLevel(
./clang/lib/Driver/ToolChains/CommonArgs.cpp:    // CompilerInvocation.cpp:getOptimizationLevel().
./clang/lib/Driver/ToolChains/Clang.cpp:    RenderFloatingPointOptions(TC, D, isOptimizationLevelFast(Args), Args,
./clang/lib/Driver/ToolChains/Clang.cpp:  bool OFastEnabled = isOptimizationLevelFast(Args);
./clang/lib/Driver/ToolChains/Hexagon.h:  unsigned getOptimizationLevel(const llvm::opt::ArgList &DriverArgs) const;
./clang/lib/Driver/Driver.cpp:bool clang::driver::isOptimizationLevelFast(const ArgList &Args) {
./clang/docs/tools/clang-formatted-files.txt:llvm/include/llvm/Passes/OptimizationLevel.h
./clang/docs/tools/clang-formatted-files.txt:llvm/lib/Passes/OptimizationLevel.cpp
./clang/include/clang/Basic/CodeGenOptions.def:VALUE_CODEGENOPT(OptimizationLevel, 2, 0) ///< The -O[0-3] option specified.
./clang/include/clang/Driver/Driver.h:bool isOptimizationLevelFast(const llvm::opt::ArgList &Args);
```

## 1.2.2 LLVM Related Files & Functions

LLVM中也定义了一个OptimizationLevel类，在llvm/include/llvm/Passes/OptimizationLevel.h:37中。

对优化等级也是定义了五类，OptimizationLevel::O0、OptimizationLevel::O1、OptimizationLevel::O2、OptimizationLevel::O3、OptimizationLevel::Oz、OptimizationLevel::Os，且说明了具体的含义，可以参考1.2.2.1小节介绍（在llvm 13.0.1中该定义在PassBuilder.h中）。而在llvm-project-llvmorg-15.0.4\llvm\lib\Passes\OptimizationLevel.cpp文件中，定义不同定义五类优化等级对应的Level和Size选项的取值。

在函数PassBuilder::parseModulePass()中，根据传递的优化选项，选择不同的Passpipeline，如buildO0DefaultPipeline、buildPerModuleDefaultPipeline、buildThinLTOPreLinkDefaultPipeline、buildThinLTODefaultPipeline、buildLTOPreLinkDefaultPipeline、buildLTODefaultPipeline等，可以参考1.2.2.2小节介绍。

在1.2.2.3小节中，介绍的PassBuilderPipelines.cpp文件中，定义一系列内置的PassPipeline，很多的PassPipeline都需要在优化等级为非O0的时候才执行，如buildFunctionSimplificationPipeline()，如果检查到优化等级为O0，则直接退出当前Pipeline的执行。

**在LLVM 13.0.1 V20221105版中，也是采用这种暴力手段不让PassPipeline退出执行，但是仅有个别的pass执行和生效，大部分的O1选项的pass都未能达到代码优化的效果，这是因为O0选项下编译器前端会添加optnone的函数属性，这就阻止了几乎所有的优化pass执行，包括分析pass和转换pass。这也能解释为什么当打开-disable-O0-optnone选项后，LLVM 13.0.1 V20221105版有了11.96%的代码量下降。**

### 1.2.2.1PassBuilder::OptimizationLevel
在文件llvm-project-llvmorg-13.0.1\llvm\include\llvm\Passes\PassBuilder.h中，（或者llvm-project-llvmorg-15.0.4\llvm\include\llvm\Passes\OptimizationLevel.h中）
```
  /// LLVM-provided high-level optimization levels.
  ///
  /// This enumerates the LLVM-provided high-level optimization levels. Each
  /// level has a specific goal and rationale.
  class OptimizationLevel final {
    unsigned SpeedLevel = 2;
    unsigned SizeLevel = 0;
    OptimizationLevel(unsigned SpeedLevel, unsigned SizeLevel)
        : SpeedLevel(SpeedLevel), SizeLevel(SizeLevel) {
      // Check that only valid combinations are passed.
      assert(SpeedLevel <= 3 &&
             "Optimization level for speed should be 0, 1, 2, or 3");
      assert(SizeLevel <= 2 &&
             "Optimization level for size should be 0, 1, or 2");
      assert((SizeLevel == 0 || SpeedLevel == 2) &&
             "Optimize for size should be encoded with speedup level == 2");
    }
    
    
    OptimizationLevel() = default;
    /// Disable as many optimizations as possible. This doesn't completely
    /// disable the optimizer in all cases, for example always_inline functions
    /// can be required to be inlined for correctness.
    static const OptimizationLevel O0;

    /// Optimize quickly without destroying debuggability.
    ///
    /// This level is tuned to produce a result from the optimizer as quickly
    /// as possible and to avoid destroying debuggability. This tends to result
    /// in a very good development mode where the compiled code will be
    /// immediately executed as part of testing. As a consequence, where
    /// possible, we would like to produce efficient-to-execute code, but not
    /// if it significantly slows down compilation or would prevent even basic
    /// debugging of the resulting binary.
    ///
    /// As an example, complex loop transformations such as versioning,
    /// vectorization, or fusion don't make sense here due to the degree to
    /// which the executed code differs from the source code, and the compile
    /// time cost.
    static const OptimizationLevel O1;
    /// Optimize for fast execution as much as possible without triggering
    /// significant incremental compile time or code size growth.
    ///
    /// The key idea is that optimizations at this level should "pay for
    /// themselves". So if an optimization increases compile time by 5% or
    /// increases code size by 5% for a particular benchmark, that benchmark
    /// should also be one which sees a 5% runtime improvement. If the compile
    /// time or code size penalties happen on average across a diverse range of
    /// LLVM users' benchmarks, then the improvements should as well.
    ///
    /// And no matter what, the compile time needs to not grow superlinearly
    /// with the size of input to LLVM so that users can control the runtime of
    /// the optimizer in this mode.
    ///
    /// This is expected to be a good default optimization level for the vast
    /// majority of users.
    static const OptimizationLevel O2;
    /// Optimize for fast execution as much as possible.
    ///
    /// This mode is significantly more aggressive in trading off compile time
    /// and code size to get execution time improvements. The core idea is that
    /// this mode should include any optimization that helps execution time on
    /// balance across a diverse collection of benchmarks, even if it increases
    /// code size or compile time for some benchmarks without corresponding
    /// improvements to execution time.
    ///
    /// Despite being willing to trade more compile time off to get improved
    /// execution time, this mode still tries to avoid superlinear growth in
    /// order to make even significantly slower compile times at least scale
    /// reasonably. This does not preclude very substantial constant factor
    /// costs though.
    static const OptimizationLevel O3;
    /// Similar to \c O2 but tries to optimize for small code size instead of
    /// fast execution without triggering significant incremental execution
    /// time slowdowns.
    ///
    /// The logic here is exactly the same as \c O2, but with code size and
    /// execution time metrics swapped.
    ///
    /// A consequence of the different core goal is that this should in general
    /// produce substantially smaller executables that still run in
    /// a reasonable amount of time.
    static const OptimizationLevel Os;
    /// A very specialized mode that will optimize for code size at any and all
    /// costs.
    ///
    /// This is useful primarily when there are absolute size limitations and
    /// any effort taken to reduce the size is worth it regardless of the
    /// execution time impact. You should expect this level to produce rather
    /// slow, but very small, code.
    static const OptimizationLevel Oz;
```
在llvm-project-llvmorg-15.0.4\llvm\lib\Passes\OptimizationLevel.cpp文件中
```
const OptimizationLevel OptimizationLevel::O0 = {
    /*SpeedLevel*/ 0,
    /*SizeLevel*/ 0};
const OptimizationLevel OptimizationLevel::O1 = {
    /*SpeedLevel*/ 1,
    /*SizeLevel*/ 0};
const OptimizationLevel OptimizationLevel::O2 = {
    /*SpeedLevel*/ 2,
    /*SizeLevel*/ 0};
const OptimizationLevel OptimizationLevel::O3 = {
    /*SpeedLevel*/ 3,
    /*SizeLevel*/ 0};
const OptimizationLevel OptimizationLevel::Os = {
    /*SpeedLevel*/ 2,
    /*SizeLevel*/ 1};
const OptimizationLevel OptimizationLevel::Oz = {
    /*SpeedLevel*/ 2,
    /*SizeLevel*/ 2};
```

### 1.2.2.2PassBuilder::parseModulePass()
llvm-project-llvmorg-15.0.4\llvm\lib\Passes\PassBuilder.cpp

```
    OptimizationLevel L = StringSwitch<OptimizationLevel>(Matches[2])
                              .Case("O0", OptimizationLevel::O0)
                              .Case("O1", OptimizationLevel::O1)
                              .Case("O2", OptimizationLevel::O2)
                              .Case("O3", OptimizationLevel::O3)
                              .Case("Os", OptimizationLevel::Os)
                              .Case("Oz", OptimizationLevel::Oz);
    if (L == OptimizationLevel::O0 && Matches[1] != "thinlto" &&
        Matches[1] != "lto") {
      MPM.addPass(buildO0DefaultPipeline(L, Matches[1] == "thinlto-pre-link" ||
                                                Matches[1] == "lto-pre-link"));
      return Error::success();
    }

    // This is consistent with old pass manager invoked via opt, but
    // inconsistent with clang. Clang doesn't enable loop vectorization
    // but does enable slp vectorization at Oz.
    PTO.LoopVectorization =
        L.getSpeedupLevel() > 1 && L != OptimizationLevel::Oz;
    PTO.SLPVectorization =
        L.getSpeedupLevel() > 1 && L != OptimizationLevel::Oz;
        
    if (Matches[1] == "default") {
      MPM.addPass(buildPerModuleDefaultPipeline(L));
    } else if (Matches[1] == "thinlto-pre-link") {
      MPM.addPass(buildThinLTOPreLinkDefaultPipeline(L));
    } else if (Matches[1] == "thinlto") {
      MPM.addPass(buildThinLTODefaultPipeline(L, nullptr));
    } else if (Matches[1] == "lto-pre-link") {
      MPM.addPass(buildLTOPreLinkDefaultPipeline(L));
    } else {
      assert(Matches[1] == "lto" && "Not one of the matched options!");
      MPM.addPass(buildLTODefaultPipeline(L, nullptr));
    }
    return Error::success();
  }
```

### 1.2.2.3PassBuilderPipelines
llvm-project-llvmorg-15.0.4\llvm\lib\Passes\PassBuilderPipelines.cpp
```
void PassBuilder::invokePeepholeEPCallbacks(FunctionPassManager &FPM,
                                            OptimizationLevel Level)
FunctionPassManager
PassBuilder::buildO1FunctionSimplificationPipeline(OptimizationLevel Level,
                                                   ThinOrFullLTOPhase Phase)
FunctionPassManager
PassBuilder::buildFunctionSimplificationPipeline(OptimizationLevel Level,
                                                 ThinOrFullLTOPhase Phase)
void PassBuilder::addPGOInstrPasses(ModulePassManager &MPM,
                                    OptimizationLevel Level, bool RunProfileGen,
                                    bool IsCS, std::string ProfileFile,
                                    std::string ProfileRemappingFile,
                                    ThinOrFullLTOPhase LTOPhase)
ModuleInlinerWrapperPass
PassBuilder::buildInlinerPipeline(OptimizationLevel Level,
                                  ThinOrFullLTOPhase Phase)
ModulePassManager
PassBuilder::buildModuleInlinerPipeline(OptimizationLevel Level,
                                        ThinOrFullLTOPhase Phase)
ModulePassManager
PassBuilder::buildModuleSimplificationPipeline(OptimizationLevel Level,
                                               ThinOrFullLTOPhase Phase)
void PassBuilder::addVectorPasses(OptimizationLevel Level,
                                  FunctionPassManager &FPM, bool IsFullLTO)
ModulePassManager
PassBuilder::buildModuleOptimizationPipeline(OptimizationLevel Level,
                                             ThinOrFullLTOPhase LTOPhase)
ModulePassManager
PassBuilder::buildPerModuleDefaultPipeline(OptimizationLevel Level,
                                           bool LTOPreLink)
ModulePassManager
PassBuilder::buildThinLTOPreLinkDefaultPipeline(OptimizationLevel Level)                                                                                       
ModulePassManager PassBuilder::buildThinLTODefaultPipeline(
    OptimizationLevel Level, const ModuleSummaryIndex *ImportSummary)
ModulePassManager
PassBuilder::buildLTOPreLinkDefaultPipeline(OptimizationLevel Level)
ModulePassManager PassBuilder::buildO0DefaultPipeline(OptimizationLevel Level,
                                                      bool LTOPreLink)

```
### 1.2.2.4 LLVM Related Files

```
./llvm/utils/gn/secondary/llvm/lib/Passes/BUILD.gn:    "OptimizationLevel.cpp",
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](FunctionPassManager &PM, OptimizationLevel Level) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](LoopPassManager &PM, OptimizationLevel Level) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](LoopPassManager &PM, OptimizationLevel Level) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](FunctionPassManager &PM, OptimizationLevel Level) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](CGSCCPassManager &PM, OptimizationLevel Level) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](FunctionPassManager &PM, OptimizationLevel Level) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](ModulePassManager &PM, OptimizationLevel) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](ModulePassManager &PM, OptimizationLevel) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](ModulePassManager &PM, OptimizationLevel) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](ModulePassManager &PM, OptimizationLevel) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](ModulePassManager &PM, OptimizationLevel) {
./llvm/tools/opt/NewPMDriver.cpp:        [&PB](ModulePassManager &PM, OptimizationLevel) {
./llvm/bindings/go/llvm/executionengine_test.go:        options.SetMCJITOptimizationLevel(2)
./llvm/bindings/go/llvm/executionengine.go:func (options *MCJITCompilerOptions) SetMCJITOptimizationLevel(level uint) {
./llvm/examples/Bye/Bye.cpp:                [](llvm::FunctionPassManager &PM, OptimizationLevel Level) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:#include "llvm/Passes/OptimizationLevel.h"
./llvm/lib/Passes/PassBuilderPipelines.cpp:                                            OptimizationLevel Level) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildO1FunctionSimplificationPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildFunctionSimplificationPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:  assert(Level != OptimizationLevel::O0 && "Must request optimizations!");
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level == OptimizationLevel::O3)
./llvm/lib/Passes/PassBuilderPipelines.cpp:      LoopRotatePass(Level != OptimizationLevel::Oz, isLTOPreLink(Phase)));
./llvm/lib/Passes/PassBuilderPipelines.cpp:      SimpleLoopUnswitchPass(/* NonTrivial */ Level == OptimizationLevel::O3 &&
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (EnableCHR && Level == OptimizationLevel::O3 && PGOOpt &&
./llvm/lib/Passes/PassBuilderPipelines.cpp:                                    OptimizationLevel Level, bool RunProfileGen,
./llvm/lib/Passes/PassBuilderPipelines.cpp:  assert(Level != OptimizationLevel::O0 && "Not expecting O0 here!");
./llvm/lib/Passes/PassBuilderPipelines.cpp:          LoopRotatePass(Level != OptimizationLevel::Oz),
./llvm/lib/Passes/PassBuilderPipelines.cpp:static InlineParams getInlineParamsFromOptLevel(OptimizationLevel Level) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildInlinerPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level == OptimizationLevel::O3)
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level == OptimizationLevel::O2 || Level == OptimizationLevel::O3)
./llvm/lib/Passes/PassBuilderPipelines.cpp:  MainCGPipeline.addPass(CoroSplitPass(Level != OptimizationLevel::O0));
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildModuleInlinerPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:      CoroSplitPass(Level != OptimizationLevel::O0)));
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildModuleSimplificationPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level == OptimizationLevel::O3)
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level != OptimizationLevel::O0)
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (EnableFunctionSpecialization && Level == OptimizationLevel::O3)
./llvm/lib/Passes/PassBuilderPipelines.cpp:void PassBuilder::addVectorPasses(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:                                       OptimizationLevel::O3));
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildModuleOptimizationPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:  LPM.addPass(LoopRotatePass(Level != OptimizationLevel::Oz, LTOPreLink));
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildPerModuleDefaultPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:  assert(Level != OptimizationLevel::O0 &&
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildThinLTOPreLinkDefaultPipeline(OptimizationLevel Level) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:  assert(Level != OptimizationLevel::O0 &&
./llvm/lib/Passes/PassBuilderPipelines.cpp:    OptimizationLevel Level, const ModuleSummaryIndex *ImportSummary) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level == OptimizationLevel::O0) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildLTOPreLinkDefaultPipeline(OptimizationLevel Level) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:  assert(Level != OptimizationLevel::O0 &&
./llvm/lib/Passes/PassBuilderPipelines.cpp:PassBuilder::buildLTODefaultPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level == OptimizationLevel::O0) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:    if (EnableFunctionSpecialization && Level == OptimizationLevel::O3)
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level == OptimizationLevel::O1) {
./llvm/lib/Passes/PassBuilderPipelines.cpp:  if (Level == OptimizationLevel::O3)
./llvm/lib/Passes/PassBuilderPipelines.cpp:ModulePassManager PassBuilder::buildO0DefaultPipeline(OptimizationLevel Level,
./llvm/lib/Passes/PassBuilderPipelines.cpp:  assert(Level == OptimizationLevel::O0 &&
./llvm/lib/Passes/CMakeLists.txt:  OptimizationLevel.cpp
./llvm/lib/Passes/PassBuilder.cpp:    OptimizationLevel L = StringSwitch<OptimizationLevel>(Matches[2])
./llvm/lib/Passes/PassBuilder.cpp:                              .Case("O0", OptimizationLevel::O0)
./llvm/lib/Passes/PassBuilder.cpp:                              .Case("O1", OptimizationLevel::O1)
./llvm/lib/Passes/PassBuilder.cpp:                              .Case("O2", OptimizationLevel::O2)
./llvm/lib/Passes/PassBuilder.cpp:                              .Case("O3", OptimizationLevel::O3)
./llvm/lib/Passes/PassBuilder.cpp:                              .Case("Os", OptimizationLevel::Os)
./llvm/lib/Passes/PassBuilder.cpp:                              .Case("Oz", OptimizationLevel::Oz);
./llvm/lib/Passes/PassBuilder.cpp:    if (L == OptimizationLevel::O0 && Matches[1] != "thinlto" &&
./llvm/lib/Passes/PassBuilder.cpp:        L.getSpeedupLevel() > 1 && L != OptimizationLevel::Oz;
./llvm/lib/Passes/PassBuilder.cpp:        L.getSpeedupLevel() > 1 && L != OptimizationLevel::Oz;
./llvm/lib/Passes/PassRegistry.def:  buildInlinerPipeline(OptimizationLevel::Oz, ThinOrFullLTOPhase::None))
./llvm/lib/Passes/OptimizationLevel.cpp://===- OptimizationLevel.cpp ----------------------------------------------===//
./llvm/lib/Passes/OptimizationLevel.cpp:#include "llvm/Passes/OptimizationLevel.h"
./llvm/lib/Passes/OptimizationLevel.cpp:const OptimizationLevel OptimizationLevel::O0 = {
./llvm/lib/Passes/OptimizationLevel.cpp:const OptimizationLevel OptimizationLevel::O1 = {
./llvm/lib/Passes/OptimizationLevel.cpp:const OptimizationLevel OptimizationLevel::O2 = {
./llvm/lib/Passes/OptimizationLevel.cpp:const OptimizationLevel OptimizationLevel::O3 = {
./llvm/lib/Passes/OptimizationLevel.cpp:const OptimizationLevel OptimizationLevel::Os = {
./llvm/lib/Passes/OptimizationLevel.cpp:const OptimizationLevel OptimizationLevel::Oz = {
./llvm/lib/LTO/ThinLTOCodeGenerator.cpp:  OptimizationLevel OL;
./llvm/lib/LTO/ThinLTOCodeGenerator.cpp:    OL = OptimizationLevel::O0;
./llvm/lib/LTO/ThinLTOCodeGenerator.cpp:    OL = OptimizationLevel::O1;
./llvm/lib/LTO/ThinLTOCodeGenerator.cpp:    OL = OptimizationLevel::O2;
./llvm/lib/LTO/ThinLTOCodeGenerator.cpp:    OL = OptimizationLevel::O3;
./llvm/lib/LTO/LTOBackend.cpp:  OptimizationLevel OL;
./llvm/lib/LTO/LTOBackend.cpp:    OL = OptimizationLevel::O0;
./llvm/lib/LTO/LTOBackend.cpp:    OL = OptimizationLevel::O1;
./llvm/lib/LTO/LTOBackend.cpp:    OL = OptimizationLevel::O2;
./llvm/lib/LTO/LTOBackend.cpp:    OL = OptimizationLevel::O3;
./llvm/lib/Target/NVPTX/NVPTXTargetMachine.cpp:      [this](ModulePassManager &PM, OptimizationLevel Level) {
./llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp:      [this](ModulePassManager &PM, OptimizationLevel Level) {
./llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp:        if (EnableLibCallSimplify && Level != OptimizationLevel::O0)
./llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp:      [this](ModulePassManager &PM, OptimizationLevel Level) {
./llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp:        if (Level == OptimizationLevel::O0)
./llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp:      [this](CGSCCPassManager &PM, OptimizationLevel Level) {
./llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp:        if (Level == OptimizationLevel::O0)
./llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp:        if (Level.getSpeedupLevel() > OptimizationLevel::O1.getSpeedupLevel() &&
./llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp:        if (Level != OptimizationLevel::O0) {
./llvm/lib/Target/Hexagon/HexagonTargetMachine.cpp:      [=](LoopPassManager &LPM, OptimizationLevel Level) {
./llvm/lib/Target/Hexagon/HexagonTargetMachine.cpp:      [=](LoopPassManager &LPM, OptimizationLevel Level) {
./llvm/lib/Target/BPF/BPFTargetMachine.cpp:      [=](ModulePassManager &MPM, OptimizationLevel) {
./llvm/lib/Target/BPF/BPFTargetMachine.cpp:                                    OptimizationLevel Level) {
./llvm/lib/Target/BPF/BPFTargetMachine.cpp:      [=](ModulePassManager &MPM, OptimizationLevel) {
./llvm/docs/CommandLine.rst:  cl::opt<OptLevel> OptimizationLevel(cl::desc("Choose optimization level:"),
./llvm/docs/CommandLine.rst:    if (OptimizationLevel >= O2) doPartialRedundancyElimination(...);
./llvm/docs/CommandLine.rst:This declaration defines a variable "``OptimizationLevel``" of the
./llvm/docs/CommandLine.rst:  cl::opt<OptLevel> OptimizationLevel(cl::desc("Choose optimization level:"),
./llvm/docs/CommandLine.rst:    if (OptimizationLevel == Debug) outputDebugInfo(...);
./llvm/docs/NewPassManager.rst:  ModulePassManager MPM = PB.buildPerModuleDefaultPipeline(OptimizationLevel::O2);
./llvm/docs/NewPassManager.rst:                                         PassBuilder::OptimizationLevel Level) {
./llvm/include/llvm/Passes/PassBuilder.h:#include "llvm/Passes/OptimizationLevel.h"
./llvm/include/llvm/Passes/PassBuilder.h:  buildFunctionSimplificationPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  ModulePassManager buildModuleSimplificationPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  ModuleInlinerWrapperPass buildInlinerPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  ModulePassManager buildModuleInlinerPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  buildModuleOptimizationPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  ModulePassManager buildPerModuleDefaultPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  ModulePassManager buildThinLTOPreLinkDefaultPipeline(OptimizationLevel Level);
./llvm/include/llvm/Passes/PassBuilder.h:  buildThinLTODefaultPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  ModulePassManager buildLTOPreLinkDefaultPipeline(OptimizationLevel Level);
./llvm/include/llvm/Passes/PassBuilder.h:  ModulePassManager buildLTODefaultPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  ModulePassManager buildO0DefaultPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(FunctionPassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(LoopPassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(LoopPassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(FunctionPassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(CGSCCPassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(FunctionPassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(ModulePassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(ModulePassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(ModulePassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(ModulePassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(ModulePassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:      const std::function<void(ModulePassManager &, OptimizationLevel)> &C) {
./llvm/include/llvm/Passes/PassBuilder.h:  buildO1FunctionSimplificationPipeline(OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  void addVectorPasses(OptimizationLevel Level, FunctionPassManager &FPM,
./llvm/include/llvm/Passes/PassBuilder.h:  void addPGOInstrPasses(ModulePassManager &MPM, OptimizationLevel Level,
./llvm/include/llvm/Passes/PassBuilder.h:  void invokePeepholeEPCallbacks(FunctionPassManager &, OptimizationLevel);
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(FunctionPassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(LoopPassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(LoopPassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(FunctionPassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(CGSCCPassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(FunctionPassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(ModulePassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(ModulePassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(ModulePassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(ModulePassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(ModulePassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/PassBuilder.h:  SmallVector<std::function<void(ModulePassManager &, OptimizationLevel)>, 2>
./llvm/include/llvm/Passes/OptimizationLevel.h:class OptimizationLevel final {
./llvm/include/llvm/Passes/OptimizationLevel.h:  OptimizationLevel(unsigned SpeedLevel, unsigned SizeLevel)
./llvm/include/llvm/Passes/OptimizationLevel.h:  OptimizationLevel() = default;
./llvm/include/llvm/Passes/OptimizationLevel.h:  static const OptimizationLevel O0;
./llvm/include/llvm/Passes/OptimizationLevel.h:  static const OptimizationLevel O1;
./llvm/include/llvm/Passes/OptimizationLevel.h:  static const OptimizationLevel O2;
./llvm/include/llvm/Passes/OptimizationLevel.h:  static const OptimizationLevel O3;
./llvm/include/llvm/Passes/OptimizationLevel.h:  static const OptimizationLevel Os;
./llvm/include/llvm/Passes/OptimizationLevel.h:  static const OptimizationLevel Oz;
./llvm/include/llvm/Passes/OptimizationLevel.h:  bool operator==(const OptimizationLevel &Other) const {
./llvm/include/llvm/Passes/OptimizationLevel.h:  bool operator!=(const OptimizationLevel &Other) const {
```


（2022-11-16，财智国际大厦，北京）
