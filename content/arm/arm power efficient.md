in a lot of chip what makes the power efficient is not the architecture but the [[ARM ASSEMBLY INTERNALS & REVERSE ENGINEERING NOTE#Diff between arch and micro-arch|micro-architecture]] that why you can see x86 cpu having a better power concemption than some arm chip.

but arm is more designed for low power this is manly due to :
- simple, fixed-length instructions ([[ARM ASSEMBLY INTERNALS & REVERSE ENGINEERING NOTE#Base of the ARCH|16/32]]) 
	- no 1-15 byte length
	- cheaper decoder (1–2 mm² vs 6–10 mm² on x86)
	- sve better power scaling than avx