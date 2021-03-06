# Thanks

* Joedubs and SimosWiki: https://www.simoswiki.com/ for putting this all together.
* tinytuning - for all useful information contained in this document. We've gotten a long way by "making little changes, one byte at a time"
* the excellent https://github.com/pylessard/python-udsoncan project, which made this easy.

# Background

The Simos18 is a series of ECUs (Engine Control Units) utilized in a broad range of Volkswagen AG vehicles. The ECUs are manufactured by Continental-Siemens and are based on the Infineon Tricore CPU.

An Engine Control Unit accepts stimulus from various sensors attached to an engine, and is responsible for then running the control systems driving the engine itself. Such inputs include one or several cam angle sensors, crankshaft trigger sensors, air pressure sensors, and position actuators for the accelerator pedal, throttle plate, and turbocharger actuators. This document is not intended to explain engine control principles from a base level and assumes the reader is familiar enough with engine control systems to recognize the value in replacing the calibration data used in tuning the ECU's control systems.

# Embedded System Architecture

The Simos18 contains either the Infineon SAK-TC1791S-384 F200EP processor ("Simos18.1") or the SAK-TC1791S-512 F240EP processor ("Simos18.10").

These processors are Tricore processors from the TC179x series. The Tricore architecture is not really a parallel multi-core architecture as the name would suggest. Rather, "the real-time capability of a microcontroller, the computational power of a DSP, and the high performance/price features of a RISC load/store architecture" lead to the "Tricore" name.

Tricore is a moderately complex RISC-like 32-bit architecture. It has hybrid 32-bit and 16-bit instructions. The Tricore parts used in the Simos ECUs look like a "typical microcontroller" in that they support exection directly from memory-mapped program flash (PMEM) and do not have an MMU. The non-MMU models have several memory mapped peripherals including a programmable I/O coprocessor. With no MMU, address space is flat, although a degree of global remapping is possible.

However, there are a few interesting protections: memory regions be marked as non-executable, and the CPU provides hardware RTOS support via hardware store and restoration of "task contexts" which can have their own stack and can be given different permissions to read/write/execute memory. Flash areas and debugger attachment can also be protected by passwords which are stored in a purportedly-inaccessible area of flash and enforced by the CPU itself (although with no security co-processor, the software running on the CPU needs to be able to calculate and apply the password at runtime to unlock itself, so this is a rather weak trust model).

In general, the main protection Tricore possesses is a simple architectural one: the flash memory is physically internal to the processor module, so it cannot be dumped, sniffed, or altered without the permission of the CPU itself. This means the protections inherent in the software are actually much simpler and more exploitable than in other embedded systems, because Flash is generally considered as a "trusted" part of the system architecture.

At any rate, none of these protections actually play a substantial role in this particular exploit process due to a series of lucky breaks, coincedences, and humorous issues.

Several things about Tricore are helpful: the "nop" instruction is represented by 00 (this is critical later), and full programmer's manual is free and available with no paywall: https://www.infineon.com/dgdl/tc1_6__architecture_vol1.pdf?fileId=db3a3043372d5cc801373b0f374d5d67 . 

# Boot Process and Layout

The Tricore boot process consists of a mask ROM, which initializes the state of several debug co-processors and provides a debug-level programming backdoor which is protected by passwords stored in the flash. Following the mask ROM, execution begins at 0x80000000 in PMEM. 

Continental/Siemens, along with most other vendors using Tricore CPUs for engine control, have chosen to implement their application software in a series of blocks:

0. 0x80000000-0x8001C000: SBOOT (Supplier Bootloader). This bootloader is fixed across the delivered control units in a Supplier range (i.e., all Simos ECUs). It is never updated or flashed remotely once the ECU is in service.
1. 0x8001C000-0x80040000: CBOOT (Customer's Bootloader). This bootloader is updated and flashed remotely with each update. It contains the Customer (in this case, VW)'s manufacturer-specified flash memory update and verification routines.
2. 0x80040000-0x80140000: ASW1 (Application Software 1). These blocks consist of the Application Software. The Application Software for most ECUs is not hand-written compiled C code, but rather generated code from a model built using a tool like Simulink. When you read an article about "millions of lines of code in a control unit," it's worth rolling your eyes and closing it.
3. 0x80140000-0x80880000: ASW2 (Application Software 2)
4. 0x80880000-0x808FFC00: ASW3 (Application Software 3)
5. 0xA0800000: CAL (Calibration). This block consists of the vehicle-specific data which drives the generated models compiled into the Application Software. This is where the tables mapping things like airflow mass to fuel requirements, pedal request to torque request, and so on live. They're what changes to make each car model run correctly and meet emissions requirements from the factory, and therefore are what "tuners" seek to change to alter the performance of a vehicle.

# Boot procedure and trust chain

The boot-time process follows the list: Mask ROM -> SBOOT -> CBOOT -> ASW/CAL.

In the case of the VW-supplied application, when CBOOT starts up, it checks that each block has an "OK Flag" set before unlocking it for execution and jumping in. These "OK Flags" are outside of the remotely-writeable range of flash. They are set as part of the flash-memory update process in CBOOT, after a block has been measured against a [CRC32 checksum](https://github.com/bri3d/VW_Flash/blob/master/checksumsimos18.py) and an RSA signature check. Because flash is "trusted" in this model, CBOOT assumes that as long as the OK flag was set in flash, no additional re-measurement of the block is necessary and it is safe to unlock and jump into the block. If all necessary OK Flags aren't set, CBOOT simply does not continue the boot process, instead waiting for a manufacturer tool to supply a valid flash to resolve the issue.

Simple enough. Quite exploitable if flash can be written, but flash is inside the CPU and locked away, so we still need a way in. 

* The Mask ROM and debuggers are appealing, but the Tricore architecture implements a series of password protections around flash memory read, write, and debug, which have proven to be fairly robust in terms of backdoor access (the passwords, of course, need to be calculatable by the application software and therefore can be discovered, but first one must read flash...). 
* SBOOT is interesting as it must allow for the supplier to install the manufacturer's software in some way or another, but the non-standard communications required aren't compatible with the other control systems in the vehicle and only work reliably with the ECU removed. This part of the system comes in handy for getting a ROM dump, but we've solved our chicken-and-egg problem already by other means (maybe this will come in another document later...).
* So, let's look at CBOOT.

CBOOT is supplied by VW and therefore follows a VW Group standard for flash-memory updates. This standard is based on a wide variety of massively overcomplicated international standards such as ASAM ODX. The VW manufacturer diagnostics tool, called ODIS, loads encrypted ODX files from a manufacturer [ZIP-and-encrypt container called FRF](https://github.com/trick77/frfdumper), reads the standardized procedure documented in the ODX, and performs the flash routine. 

This means that with an ODX file, the flash routine is an open standard and becomes a matter of reading a horrifying amount of standards documents, which I'll save you the pain of doing by simply documenting the actual procedure here.

# OEM Update Process

Let's look at the factory update routine for this ECU. Updates for this ECU, as for most modern control units, happen over UDS over ISO-TP over CAN, which is exposed on the standard OBD-II diagnostics port. For the uninitiated, this is similar to layers in any other networking stack, from low to high:

* CAN (physical bus, device arbitration, addressing)
* ISO-TP (packet splitting/framing, checksums)
* UDS (application layer)

Thankfully, due to the highly standards-based nature of this system, constructing a flashing tool is relatively simple. The CAN and ISO-TP layers are outside the scope of this document, as they are handled for us by SocketCAN and CAN-ISOTP Linux kernel modules, leaving us with responsiblity for UDS, for which a [wealth of extremely high quality open-source implementations are available](https://github.com/pylessard/python-udsoncan).

# Simos18 Flash Procedure

The basic update layout for Simos18 is block based, rather than address based as in some previous ECUs. Each block is indexed beginning from "customer bootloader" above - so 

1. CBOOT
2. ASW1
3. ASW2
4. ASW3
5. Calibration

[The procedure to flash blocks looks like this](https://github.com/bri3d/VW_Flash/blob/master/flashsimos18.py):

* Clear Diagnostic Trouble Codes over OBD-II byte `04`, prior to upgrading to UDS. This is an essential "knock" which starts the diagnostic process in the Application Software.
* Enter UDS "extended diagnostic" session.
* Read a specific sub-set of UDS "local identifiers" corresponding to information the flashing tool needs to save off to the manufacturer, which also functions as a "knock" to signal the presence of a factory tool. 
* In Simos18, invoke a "remote routine" on the ECU to verify programming preconditions and set some flags.
* Enter UDS "programming" session, which "descends" into CBOOT by loading a section of CBOOT into memory, halting the Application Software tasks, and passing execution off to CBOOT itself (this comes in handy later!)
* Perform Seed/Key Security Access. [Seed/Key Security Access on VW relies on a clever bytecode virtual machine](https://github.com/bri3d/VW_Flash/blob/master/volkswagen_security.py). ODX update files are shipped with a small bytecode script, which the flashing tool runs against the Seed to create a Key.
* Write a "workshop log" to a specific "LocalIdentifier." This contains flash log information around the date, time, and workshop code. This data is eventually stored in the protected NVRAM container which also stores immobilizer, VIN, and trouble code data.
* Invoke a standard "remote routine" on the ECU to perform an Erase Block procedure, which will erase an _entire_ block, including its "OK flag."
* Send UDS "RequestDownload" with the block and length being downloaded... along with a "data format specifier" indicating "encryption A" and "compression A". Uh oh. We'll have to get to this later.
* Repeat UDS "TransferData" with block data. Remember, this ECU has nowhere near enough RAM to store a full block, and nowhere near enough free flash to store a second OS. So the data is immediately decrypted, decompressed, and written directly into flash in the area it will be executed from. 
* Send UDS Exit Transfer to signify the transfer is complete.
* Send a standard "remote routine" request to the ECU to perform a Checksum procedure on the block which was just written. The CBOOT checks a CRC32 checksum and an RSA signature. At this point, the "OK Flag" is written and the software is valid. If the checksumming process fails, the "OK Flag" is simply never written, and execution will not be passed off from CBOOT to the Application Software. 
* Exit the programming procedure.
* Reboot the ECU and clear codes again to present a clean working state to the customer.

# Compression and Encryption

That "[encryption A](https://github.com/bri3d/VW_Flash/blob/master/encryptsimos18.py)" and "[compression A](https://github.com/bri3d/VW_Flash/blob/master/lzss/lzss.c#L69)" is going to get to us. The good news is, we've magically been supplied with a decompressed, unencrypted ROM dump. After some reverse engineering, we find both the AES keys and a simple implementation of LZSS compression. So now we can compress and decompress raw binaries and send them to the ECU. Sweet. (Don't worry, I'll write a document about the process to reverse engineer this at a later date.)

# The Exploit

At this point we know how to flash factory software that passes the Checksum procedure. But we want to flash software that doesn't pass the Checksum procedure. We can send it to the ECU, and it will be written (remember, there's not enough storage or RAM to verify it before it's on the flash), but that pesky Okay flag won't let us boot into it...

Unless...

*So, here it is.* The exploit that allows arbitrary code execution on Simos18. It's... ... ... we send an Erase for one block, and then just write data to another block that's already been Checksummed and marked as Okay. That's it. That's the exploit. I know. To make this even more humorous, several Tricore-based VW ECUs seem to share this code.

It seems that when VW's engineers implemented the CBOOT flashing mechanism to send to the supplier, they followed a specification for a state machine where "Erase" was required before "Download" and "Checksum." This made sense, and would prevent arbitrary code execution as Erase would clear the Okay flag for a given block - except, they didn't check in their state machine _which_ block was Erased. Furthermore, they seem to have left support in the CBOOT for writing uncompressed data, which can be used to write over the top of un-erased flash.

Well, that was easy enough. The bad news is there's a little bit more to this...

First off, at this point we're trying to transfer data into a block of flash memory that hasn't been erased. We have no control over the flashing routine, which assumes the block has been zeroed and isn't telling the flash controller to erase a page or to ensure a written block has time to flip. This is fraught with peril - as we can only flip bits upwards, not downwards. The good news is that several nuances of Tricore come back to help us here: nop is 00, and there's empty space in the flash, so we have a lot of room to play with when it comes to writing a payload.

Second off, transferring data into already-written flash using this uncompressed data handler is a bit... wonky. We need to write our data very slowly to ensure the bits we want flipped actually physically end up flipped. And we need to do so on a physical block boundary, or we'll end up with all sorts of strange issues.

But, overall, not too bad. Find some nops, replace them with a jump to empty memory, write a little procedure, flash it slowly and off we go...

Er... well... what do we want the procedure to do? Where do we want to patch?

# The Patch

Let's just patch CBOOT and force it to mark every block as Okay... easy? Right??? Well... not so fast. Remember how CBOOT writes each block directly to flash, verifies each block, and writes an Okay flag? That's not how CBOOT writes itself. If CBOOT overwrote itself directly in flash, bricking an ECU would become really easy - a partial write of CBOOT would spell doom! 

But, it turns out CBOOT is small! So, CBOOT updates CBOOT by writing the new CBOOT_temp to free space, applying the Checksum and Signature (RSA) process, and then copying the new CBOOT payload back over the "real" CBOOT only if it passes verification.

And how does CBOOT overwrite itself, if it's running out of PFLASH? Well, remember from the "programming" section earlier - it's not! CBOOT is loaded into memory to perform all of this flashing - rather than running out of PFLASH. This comes in handy for us shortly.

So, we can't patch CBOOT directly. We can Transfer CBOOT data without Erasing it, in the same way we can application data, but we're Erasing and Transferring into the "temporary" CBOOT, which gets checksummed and signature-checked before it's written back. Unless we can make this CBOOT valid, it will never get copied back into its "home" and will never execute. So that's not very useful, unless we want to chase an RSA exploit around for a bit.

But, we do have unconstrained code execution in ASW at this point, as we can overwrite whatever part of THAT we'd like (constrained to the standard block boundaries, of course). So let's explore our options:

* Small brain: Implement a flash procedure in our ASW code which can update a block for us and mark it as Okay.  We have to write a whole flasher in our ASW code, and there are a lot of gymanastics we need to do to turn off write protection, set up the right permissions, and access the flash controller. But this will work, eventually. Wasn't it nice having CBOOT do that for us before?
* Expanding brain: Patch the real CBOOT in flash directly from our ASW code. We still need to write a flasher, which sucks, but we only need to run it once in order to install a compromised CBOOT. We'll need to find a patch in the CBOOT that will cause it to write unauthenticated data (it turns out this is one instruction, which won't surprise those of us familiar with bypassing most authentication systems). But, this seems viable. Unfortunately it's also extremely high risk - if we overwrite CBOOT in a way that doesn't run just right, we're bricked! (hopefully we saved off the Tricore bootmode passwords, which is out of scope for this document but will let us unbrick using debug tools).
* Galaxy brain: Let CBOOT patch itself! Remember, ASW loads CBOOT into RAM when it "descends" into Programming mode. What if we just copy the "descent" code, patch the CBOOT in RAM before we jump into it, and if we're really lucky, what if the same authentication bypass will cause CBOOT to copy our own CBOOT back over itself? Could we be so lucky? ... of course we could, it's Simos18!

[Without further ado, here's the patch!](patch.md)