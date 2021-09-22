# llvm-machine-instruction-pass
an LLVM Machine Instruction Pass for Binary Instrumentation on RISCV
Full Video available on https://youtu.be/mmBfaMA3DUg

## Clone an LLVM Repository
```sh
git clone https://github.com/llvm/llvm-project.git riscv-llvm
cd riscv-llvm/llvm
```

## Add line on llvm/lib/Target/RISCV/CMakeList
```c
add_llvm_target(RISCVCodeGen
  RISCVAsmPrinter.cpp
  RISCVCallLowering.cpp
  RISCVExpandAtomicPseudoInsts.cpp
  RISCVExpandPseudoInsts.cpp
  RISCVFrameLowering.cpp
  RISCVInsertVSETVLI.cpp
  RISCVInstrInfo.cpp
  RISCVInstructionSelector.cpp
  RISCVISelDAGToDAG.cpp
  RISCVISelLowering.cpp
  RISCVLegalizerInfo.cpp
  RISCVMCInstLower.cpp
  RISCVMergeBaseOffset.cpp
  RISCVRegisterBankInfo.cpp
  RISCVRegisterInfo.cpp
  RISCVSubtarget.cpp
  RISCVTargetMachine.cpp
  RISCVTargetObjectFile.cpp
  RISCVTargetTransformInfo.cpp
  RISCVMachineInstrPrinter.cpp //add this line
```

## Add line on llvm/lib/Target/RISCV/RISCV.h
```c
FunctionPass *createRISCVISelDag(RISCVTargetMachine &TM);

FunctionPass *createRISCVMachineInstrPrinterPass(); //add this line
void initializeRISCVMachineInstrPrinterPass(PassRegistry &); //and this line

FunctionPass *createRISCVMergeBaseOffsetOptPass();
void initializeRISCVMergeBaseOffsetOptPass(PassRegistry &);

```

## Create a new file: llvm/lib/Target/RISCV/RISCVMachineInstrPrinter.cpp
```c
#include "RISCV.h"
#include "RISCVInstrInfo.h"
#include "llvm/CodeGen/MachineFunctionPass.h"
#include "llvm/CodeGen/MachineInstrBuilder.h"
#include "llvm/CodeGen/TargetRegisterInfo.h"

using namespace llvm;

#define RISCV_MACHINEINSTR_PRINTER_PASS_NAME "Dummy RISCV machineinstr printer pass"

namespace {

class RISCVMachineInstrPrinter : public MachineFunctionPass {
public:
    static char ID;

    RISCVMachineInstrPrinter() : MachineFunctionPass(ID) {
        initializeRISCVMachineInstrPrinterPass(*PassRegistry::getPassRegistry());
    }

    bool runOnMachineFunction(MachineFunction &MF) override;

    StringRef getPassName() const override { return RISCV_MACHINEINSTR_PRINTER_PASS_NAME; }
};

char RISCVMachineInstrPrinter::ID = 0;

bool RISCVMachineInstrPrinter::runOnMachineFunction(MachineFunction &MF) {

    outs() << "In " << MF.getFunction().getName() << " Function\n";
    for (auto &MBB : MF) {
        for (auto &MI : MBB) {
            
            const TargetInstrInfo *XII = MF.getSubtarget().getInstrInfo();
            DebugLoc DL;

            if (MI.mayStore())
                outs() << "Found Store\n";

            if (MI.mayLoad())
                outs() << "Found Load\n";
            
            if (MI.isReturn() && (MF.getName() == "count")) 
            {    
                outs() << "Found Return\n";
                // addi a0, a0, 2
                BuildMI(MBB, MI, DL, XII->get(RISCV::ADDI), RISCV::X10)
                .addReg(RISCV::X10)
                .addImm(2);
            }          
    } 

    return false;
}
}

} // end of anonymous namespace

INITIALIZE_PASS(RISCVMachineInstrPrinter, "RISCV-machineinstr-printer",
    RISCV_MACHINEINSTR_PRINTER_PASS_NAME,
    true, // is CFG only?
    true  // is analysis?
)

namespace llvm {

FunctionPass *createRISCVMachineInstrPrinterPass() { return new RISCVMachineInstrPrinter(); }

}
```

## Add line on llvm/lib/Target/RISCV/RISCVTargetMachine.cpp
```c
extern "C" LLVM_EXTERNAL_VISIBILITY void LLVMInitializeRISCVTarget() {
  RegisterTargetMachine<RISCVTargetMachine> X(getTheRISCV32Target());
  RegisterTargetMachine<RISCVTargetMachine> Y(getTheRISCV64Target());
  auto *PR = PassRegistry::getPassRegistry();
  initializeGlobalISel(*PR);
  initializeRISCVMergeBaseOffsetOptPass(*PR);
  initializeRISCVExpandPseudoPass(*PR);
  initializeRISCVInsertVSETVLIPass(*PR);
  initializeRISCVMachineInstrPrinterPass(*PR); //add this line
}
...
void RISCVPassConfig::addPreEmitPass2() {
  addPass(createRISCVExpandPseudoPass());
  // Schedule the expansion of AMOs at the last possible moment, avoiding the
  // possibility for other passes to break the requirements for forward
  // progress in the LR/SC block.
  addPass(createRISCVExpandAtomicPseudoPass());
  addPass(createRISCVMachineInstrPrinterPass()); //and this line
}
```

## Build the clang
```sh
RISCV_GNU_TOOLCHAIN_PATH = "/your-riscv-toolchain-directory"
mkdir build
cd build
cmake -G Ninja -DCMAKE_BUILD_TYPE="Release"  -DLLVM_ENABLE_PROJECTS=clang -DBUILD_SHARED_LIBS=False -DLLVM_USE_SPLIT_DWARF=True  -DCMAKE_INSTALL_PREFIX="$RISCV_GNU_TOOLCHAIN_PATH"  -DLLVM_OPTIMIZED_TABLEGEN=True -DLLVM_BUILD_TESTS=False -DLLVM_PARALLEL_LINK_JOBS=False -DDEFAULT_SYSROOT="$RISCV_GNU_TOOLCHAIN_PATH/riscv64-unknown-elf"  -DLLVM_DEFAULT_TARGET_TRIPLE="riscv64-unknown-elf"  -DLLVM_TARGETS_TO_BUILD="RISCV"  ../llvm
cmake --build . --target install
```

## Create a C program, math.c
```c
#include <stdio.h>

int count(int a, int b)
{
    return a + b;
}

int main()
{
    int a = 0;
    int b = 1;
    int c = 2;

    a = count(b,c);
    printf("Result: %d\n", a);
    return a;
}
```

## Compile and test
```sh
riscv64-unknown-elf-gcc math.c -o math
./math
clang math.c -o math-edit
./math-edit
```
Math will returns 3, while the math-edit will return 5.

## Check Assembly Difference
```sh
riscv64-unknown-elf-objdump -d math > math.txt
riscv64-unknown-elf-objdump -d math-edit > math-edit.txt
```
Compare math.txt and math-edit.txt
