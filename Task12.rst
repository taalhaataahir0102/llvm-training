Pass Discription
=================

Write analysis pass similar to previous one. Here instead of Analysing the complete module, only analyze between two functions named start_analysis() and end_analysis().

Code
====

Here is the pass.cpp file:

.. code-block:: C

	#include "llvm/Transforms/Utils/NewStart.h"
	#include "llvm/IR/InstIterator.h"
	#include "llvm/IR/Instructions.h"
	#include "llvm/Pass.h"
	#include "llvm/IR/Function.h"
	#include "llvm/IR/IRBuilder.h"
	#include "llvm/IR/Value.h"

	using namespace llvm;

	PreservedAnalyses NewStartPass::run(Module &M, ModuleAnalysisManager &AM) {
	    LLVMContext &Context = M.getContext();

	    // Creating a prototype for the 'func' function
	    FunctionType *FuncType = FunctionType::get(Type::getVoidTy(Context), false);
	    Function *FuncDecl = Function::Create(FuncType, Function::ExternalLinkage, "func", &M);

	    Function *StartAnalysisFunc = M.getFunction("start_analysis");
	    Function *EndAnalysisFunc = M.getFunction("end_analysis");

	    bool do_analysis = false;
	    if (!StartAnalysisFunc || !EndAnalysisFunc) {
		errs() << "start_analysis() and/or end_analysis() not found. No transformation applied.\n";
		return PreservedAnalyses::all();
	    }

	    for (Function &F : M) {
		errs() << F.getName() << "\n";

		for (BasicBlock &BB : F) {
		    for (Instruction &I : BB) {
		        errs() << "Instruction: " << I << "  " << do_analysis << "\n";

		        if (CallInst *Call = dyn_cast<CallInst>(&I)) {
		            Function *Callee = Call->getCalledFunction();
		            if(!Callee)
		                continue;
		            if (Callee->getName() == "start_analysis") {
		                    do_analysis = true;
		            }
		            if (Callee->getName() == "end_analysis") {
		                    do_analysis = false;
		            }

		            if (do_analysis == true){
		                errs() << "Function called: " << Callee->getName() << "\n";

		                if (Callee->getName() == "printf") {
		                    errs() << "Yes, it's printf!\n";

		                    IRBuilder<> Builder(Call);
		                    Builder.CreateCall(FuncDecl);
		                }
		            }
		        }
		    }
		}
	    }

	    return PreservedAnalyses::none();
	}


This one is similar to the previous one. The only thing here is that we are looking for functions named start_analysis and end_analysis. Afterwards we initialize a bool which becomes true when we're between start_analysis and end_analysis.


Input IR
=========

.. code-block:: bash

	clang -S -O0 -Xclang -disable-O0-optnone -emit-llvm -fno-discard-value-names start.c -o newstart.ll
	
.. code-block:: C
        
        ; Function Attrs: noinline nounwind uwtable
	define dso_local signext i32 @here() #0 {
	entry:
	  %call = call signext i32 (ptr, ...) @printf(ptr noundef @.str)
	  ret i32 1
	}

	declare dso_local signext i32 @printf(ptr noundef, ...) #1

	; Function Attrs: noinline nounwind uwtable
	define dso_local void @start_analysis() #0 {
	entry:
	  ret void
	}

	; Function Attrs: noinline nounwind uwtable
	define dso_local void @end_analysis() #0 {
	entry:
	  ret void
	}

	; Function Attrs: noinline nounwind uwtable
	define dso_local signext i32 @here2() #0 {
	entry:
	  %call = call signext i32 (ptr, ...) @printf(ptr noundef @.str.1)
	  ret i32 2
	}

	; Function Attrs: noinline nounwind uwtable
	define dso_local signext i32 @main() #0 {
	entry:
	  %retval = alloca i32, align 4
	  %i = alloca i32, align 4
	  %j = alloca i32, align 4
	  %o = alloca i32, align 4
	  %b = alloca i32, align 4
	  store i32 0, ptr %retval, align 4
	  store i32 8, ptr %i, align 4
	  store i32 64, ptr %j, align 4
	  %0 = load i32, ptr %i, align 4
	  %1 = load i32, ptr %j, align 4
	  %cmp = icmp sgt i32 %0, %1
	  br i1 %cmp, label %if.then, label %if.end

	if.then:                                          ; preds = %entry
	  %call = call signext i32 (ptr, ...) @printf(ptr noundef @.str.2)
	  br label %if.end

	if.end:                                           ; preds = %if.then, %entry
	  call void @start_analysis()
	  %2 = load i32, ptr %i, align 4
	  %3 = load i32, ptr %j, align 4
	  %mul = mul nsw i32 %2, %3
	  store i32 %mul, ptr %o, align 4
	  %4 = load i32, ptr %o, align 4
	  %cmp1 = icmp slt i32 %4, 0
	  br i1 %cmp1, label %if.then2, label %if.else

	if.then2:                                         ; preds = %if.end
	  store i32 1, ptr %o, align 4
	  %call3 = call signext i32 (ptr, ...) @printf(ptr noundef @.str.3)
	  br label %if.end5

	if.else:                                          ; preds = %if.end
	  %call4 = call signext i32 (ptr, ...) @printf(ptr noundef @.str.4)
	  br label %if.end5

	if.end5:                                          ; preds = %if.else, %if.then2
	  %call6 = call signext i32 @here()
	  store i32 %call6, ptr %b, align 4
	  %5 = load i32, ptr %i, align 4
	  %6 = load i32, ptr %j, align 4
	  %call7 = call signext i32 (ptr, ...) @printf(ptr noundef @.str.5, i32 noundef signext %5, i32 noundef signext %6)
	  %7 = load i32, ptr %j, align 4
	  %add = add nsw i32 %7, 1
	  store i32 %add, ptr %j, align 4
	  call void @end_analysis()
	  %call8 = call signext i32 (ptr, ...) @printf(ptr noundef @.str.6)
	  ret i32 0
	}
	
Output
======

Running the pass

.. code-block:: bash

	./opt -passes=newcall start.ll -S -o newstart.ll


New IR generated:

.. code-block:: C

        ; Function Attrs: noinline nounwind uwtable
	define dso_local signext i32 @here() #0 {
	entry:
	  %call = call signext i32 (ptr, ...) @printf(ptr noundef @.str)
	  ret i32 1
	}

	declare dso_local signext i32 @printf(ptr noundef, ...) #1

	; Function Attrs: noinline nounwind uwtable
	define dso_local void @start_analysis() #0 {
	entry:
	  ret void
	}

	; Function Attrs: noinline nounwind uwtable
	define dso_local void @end_analysis() #0 {
	entry:
	  ret void
	}

	; Function Attrs: noinline nounwind uwtable
	define dso_local signext i32 @here2() #0 {
	entry:
	  %call = call signext i32 (ptr, ...) @printf(ptr noundef @.str.1)
	  ret i32 2
	}

	; Function Attrs: noinline nounwind uwtable
	define dso_local signext i32 @main() #0 {
	entry:
	  %retval = alloca i32, align 4
	  %i = alloca i32, align 4
	  %j = alloca i32, align 4
	  %o = alloca i32, align 4
	  %b = alloca i32, align 4
	  store i32 0, ptr %retval, align 4
	  store i32 8, ptr %i, align 4
	  store i32 64, ptr %j, align 4
	  %0 = load i32, ptr %i, align 4
	  %1 = load i32, ptr %j, align 4
	  %cmp = icmp sgt i32 %0, %1
	  br i1 %cmp, label %if.then, label %if.end

	if.then:                                          ; preds = %entry
	  %call = call signext i32 (ptr, ...) @printf(ptr noundef @.str.2)
	  br label %if.end

	if.end:                                           ; preds = %if.then, %entry
	  call void @start_analysis()
	  %2 = load i32, ptr %i, align 4
	  %3 = load i32, ptr %j, align 4
	  %mul = mul nsw i32 %2, %3
	  store i32 %mul, ptr %o, align 4
	  %4 = load i32, ptr %o, align 4
	  %cmp1 = icmp slt i32 %4, 0
	  br i1 %cmp1, label %if.then2, label %if.else

	if.then2:                                         ; preds = %if.end
	  store i32 1, ptr %o, align 4
	  call void @func()
	  %call3 = call signext i32 (ptr, ...) @printf(ptr noundef @.str.3)
	  br label %if.end5

	if.else:                                          ; preds = %if.end
	  call void @func()
	  %call4 = call signext i32 (ptr, ...) @printf(ptr noundef @.str.4)
	  br label %if.end5

	if.end5:                                          ; preds = %if.else, %if.then2
	  %call6 = call signext i32 @here()
	  store i32 %call6, ptr %b, align 4
	  %5 = load i32, ptr %i, align 4
	  %6 = load i32, ptr %j, align 4
	  call void @func()
	  %call7 = call signext i32 (ptr, ...) @printf(ptr noundef @.str.5, i32 noundef signext %5, i32 noundef signext %6)
	  %7 = load i32, ptr %j, align 4
	  %add = add nsw i32 %7, 1
	  store i32 %add, ptr %j, align 4
	  call void @end_analysis()
	  %call8 = call signext i32 (ptr, ...) @printf(ptr noundef @.str.6)
	  ret i32 0
	}

	declare void @func()
	
Here we cann se that a new function named func is created and inserted at the top of each printf statement which are between start and end analysis functions.

Now we create a new C file having the declation of this function and a counter which will update whenever this function is called.

.. code-block:: C

	#include<stdio.h>
	int Numprintf = 0;
	void fun(){
		printf("Number of printf function executed during runtime until now:%d\n",++Numprintf);
	}
	
Create object files for both and link them:

.. code-block:: bash

	./clang -c wow.c
	./llvm-as newstart.ll -o newstart.bc
	./clang -c newstart.bc
	./clang wow.o newstart.o -o output
	
Now run the output executable:

.. code-block:: bash
	
	./qemu-riscv64 output

.. code-block:: C

	Number of printf function executed during runtime until now:1
	Number of printf function executed during runtime until now:2


