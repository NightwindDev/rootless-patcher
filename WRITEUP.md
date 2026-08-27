# Writeup
## How does `rootless-patcher` work?
- Takes the inputted rootful .deb.
- Repacks rootful .deb into a temporary directory, changing the jbroot structure of the .deb to be positioned in the `/var/jb` folder.
- Scans through all Mach-O, `.plist`, and script files (`postinst`, `extrainst`, regular script files, etc.).
- Uses the [`ConversionRuleset.json`](layout/Library/Application%20Support/rootless-patcher/ConversionRuleset.json) file to understand what strings should get converted to what.
- FAT Mach-O files are thinned into their single-slice counterparts.
- Mach-O files' entitlements get saved and they proceed to be unsigned in order to do modifications.
- The strings are scanned from `__TEXT,__cstring` and `__DATA,__data` sections of the Mach-O files.
- A new `__PATCH_ROOTLESS` segment is added to the Mach-O before the `__LINKEDIT` segment (and `__LLVM` segment if the Mach-O contains it). This segment contains its own `__cstring` section.
- The string xrefs in the Mach-O's are patched to point to the new strings in the `__PATCH_ROOTLESS,__cstring` section.
	- There are four different categories of strings which need to be handled:
		- `CFString`s (located in `__DATA,__cfstring`/`__DATA_CONST,__cfstring`)
		- Global C strings (located in `__DATA,__data`)
		- Regular C strings (located in `__TEXT,__cstring`)
		- Certain Swift strings (more detail on these later)
- If any binary is compiled with the old arm64e ABI, `rootless-patcher` takes note of that for later.
- Make sure to keep track of the original file permissions, and merge the patched thinned Mach-O's into a FAT file, overwriting the original.
- The load commands are fixed via the [`repack-rootless.sh`](layout/Library/Application%20Support/rootless-patcher/repack-rootless.sh) script to correspond to their rootless ones.
- The patched Mach-O is then resigned and its old entitlements are applied if needed.
- The `control` file gets modified, keeping its old permissions in memory.
	- The old ABI dependency gets added to the `control` file if the previous search for a binary compiled with the old arm64e ABI found one.
	- The `Icon:` path gets converted to the rootless one.
	- The `Architecture:` gets changed to `iphoneos-arm64` instead of the rootful one - `iphoneos-arm`.
	- The package ID and version get saved for later use.
	- The patched `control` file gets saved and its permissions are restored to the original ones.
- All of the relevant `.plist` files have their paths converted. Same steps as before, the file permissions are stored, the file's paths get converted, the file is saved and its permissions are restored.
- All of the non-Mach-O script files have their paths converted. Same steps as before, the file permissions are stored, the file's paths get converted, the file is saved and its permissions are restored.
- Some special handling is done for certain tweaks (e.g. XenHTML). These tweaks have an improper `DEBIAN` directory structure, which causes `dpkg-deb` to not be able to repack the .deb correctly. `rootless-patcher` fixes this structure to correspond to `dpkg-deb`'s format.
- `dpkg-deb` is used to repack the .deb file with the converted name.
- The temporary working directory is deleted.
## Development challenges
I wanted to list some of the challenges that came with developing `rootless-patcher`. There aren't many projects which edit Mach-O's to the level of `rootless-patcher`, so I discovered some things that people hadn't stumped onto before.
### The `__LLVM` segment
The [`__LLVM` segment](https://www.graalvm.org/latest/reference-manual/llvm/Compiling/) on certain Mach-O's proved to cause a challenge when attempting to add the `__PATCH_ROOTLESS` segment. Initially, the approach that I used to add the segment was to add it right before the `__LINKEDIT` segment. The original approach was based on the ["Adding a segment to an existing macOS Mach-O binary"](https://alexomara.com/blog/adding-a-segment-to-an-existing-macos-mach-o-binary/) writeup by Alexander O'Mara, which was an incredibly helpful starting point. Most Mach-O's do not contain the `__LLVM` bitcode segment. The issue when adding a custom segment is that the `__LLVM` segment needs to be placed directly before the `__LINKEDIT` segment. Therefore, it needed to be handled separately. You can see this special case in the [MachOModifier.m](src/RPMachOModifier.m) file. Big thanks to [Leptos](https://github.com/leptos-null) and [objc](https://github.com/EthanArbuckle) in figuring out how to handle this special case correctly.
### Rebasing string xrefs
This is the core of the entire patcher. First of all, huge thanks to [staturnz](https://github.com/staturnzz) for extraordinary amount of help in this area. This project would not be possible without him.
While changing the references for `CFString`s is relatively straightforward (see [`-[RPStringPatcher _patchCFString:replacementAddress:newLength:]`](src/RPStringPatcher.m#L88-L100)), other types of strings are more difficult to handle. Global C string references are somewhat simple to patch as well. Regular C string references are more complicated to rebase, however. In order to correctly rebase them, a patchfinder needs to be written which would keep track of the references to the C strings. These strings can be referenced through the `ADRP` + `ADD`, and `ADR` ARM64 assembly instructions. Each one of these needs to be correctly handled.
#### Max range of the `ADR` instruction
The `ADR` instruction has a range of +/- 1MB from the current program counter (`PC`). This is why `ADRP` + `ADD` is used, which allows a +/- 4GB range from the `PC`, plus the page offset. There is a special case when dealing with `ADR`, however. The original implementation may work for normal string xrefs, however since the `__PATCH_ROOTLESS,__cstring` section with the converted strings may physically be farther from the `ADR`, the patched xrefs may fail. There are two options to remedy this:
- The first option involves checking the next instruction. If it is a `NOP`, we can freely overwrite it. The `NOP` is really useful because `ADRP` + `ADD` takes two instructions, which can be put in the space of the old `ADR` and `NOP` instructions — [see code](src/RPStringPatcher.m#L163-L180).
- Another option involves more code and overhead. If we don't have space for `ADRP` + `ADD`, we may be able to use a trampoline. Essentially, the idea is this: we have a small function which we define, which can allow us to properly reference the converted string in `__PATCH_ROOTLESS,__cstring`. The idea is that in the space that was previously occupied by the `ADR`, we may be able to branch to the function we defined, run our code there, and then return back to regular execution.
#### Swift strings
For optimization purposes, Swift strings have some special behavior which makes it difficult to statically patch their xrefs. For short strings (<=30 characters?), the size is stored statically in `__TEXT,__text`, somewhere near an xref to the string. This causes an issue, let's say we have a string: `/Library/MobileSubstrate/` (25 characters long) and we want to patch it to `/var/jb/Library/MobileSubstrate/` (32 characters long). The old length of 25 would be stored in the binary, causing the patched string to be truncated to `/var/jb/Library/MobileSubs` when referenced in the binary. This behavior is not well-defined and it is not trivial to patch the length. The patcher implements a [`-[RPStringPatcher _patchSwiftInstructionForLengthAt:oldLength:newLength:]`](src/RPStringPatcher.m#L187-L202) method which aims to fix one of the special cases, however there are many more, some of which are extremely difficult to patch due to there not being enough free space in the assembly. In these cases, a solution which would convert the strings at runtime may cause less headaches and overhead.
### Improper FAT files
FAT files with multiple slices of the same `cpusubtype & ~CPU_SUBTYPE_MASK` are considered invalid. XenHTML has this structure for some of its Mach-O files in order to support both the old and new arm64e ABIs. However, this causes Mach-O tools to fail because these types of FAT files are considered improper and are not supported. `rootless-patcher` remedies this ([see code](main.m#L130-L134)) by adhering to the regular FAT format.
### Improper `DEBIAN` directory format
XenHTML also has an improper `DEBIAN` directory format, which was discovered during testing. `rootless-patcher` handles this by fixing the format ([see code](main.m#L293-L325)) for `dpkg-deb` to properly repack it.
### Limitations
- Binaries with obfuscated strings will not be patched properly. In order to convert the strings, `rootless-patcher` needs to have the unobfuscated strings so it can convert them and rebase the xrefs.
- There may be some false positives. With [ConversionRuleset.json](layout/Library/Application%20Support/rootless-patcher/ConversionRuleset.json), I've done my best to rule out most false positives, but some may slip past.
- Swift strings may be patched or interpreted incorrectly, as explained previously.
### Credits
- Big thanks to [staturnz](https://github.com/staturnzz) for help in the [RPStringPatcher.m](src/RPStringPatcher.m) part of the project. Without him, this project would not have been possible.
- Thanks to [opa334](https://github.com/opa334) for the initial idea on how to implement the patcher.
- Thanks to [Leptos](https://github.com/leptos-null) for reviewing the code to help make it as reliable as possible.
- Thanks to [tuancc](https://github.com/roothider) for help in regard to some of the technicalities.
- Thanks to the testers who tested the patcher with many tweaks, helping round out some of the edge cases!
