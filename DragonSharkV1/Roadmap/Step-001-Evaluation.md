# Evaluation process
The idea of this step is to evaluate the current state and create a checkpoint of the full development. The steps will be defined as follows:

1. Document the involved hardware and Operative System.
2. Document the installed software services. So far, they're:
	1. The well-known `emulationstation` launcher with `retroarch` interface to `libretro` emulators.
	2. The brand-new `Hawa` games launcher.
	3. The gamepad server.
3. Save a snapshot image (an `ISO` file) and define the policies to properly store it (backup).
4. Test the launcher for WebGL games, again.
5. Test the launcher for Native games.
6. Test the launcher for Emulated games.

The deliverables from this task involve completing certain specifications / development documentation's entries / files. The details of the nature of these tasks will be defined here, along the proper documentation files that must be populated (or consulted, when ready).
## Hardware and Operative System
The idea is to traverse the technical specifications of the console, in terms of hardware and software. This includes:

1. [x] Hardware specifications.
2. [x] OS specifications and configuration (e.g. default users' credentials). If the credentials cannot be retrieved, they should be forced to default values.
3. [x] Installed drivers (e.g. graphics, sound).

- [x] Everything must be noted in [the Specifications file](HardwareSpecs.md).
## Installed Software Services
The idea is to tell about the software services that are installed in the console. These services are technical in nature, and the end user does not need to know about them except for arising problems. They're:

1. [x] The `libretro` and `emulationstation` packages.
2. [x] The Hawa Launcher service.
3. [x] The Hawa VirtualPad service.

The only required thing is to document everything there.

No extra services are installed _at this stage of the development_ but that does not mean that no more services will definitely be present in a later stage.
## Saving a Snapshot
- [x] The idea here is to take a snapshot using the `dd` tool. https://wiki.banana-pi.org/Getting_Started_with_M5/M2Pro#Install_Image_to_EMMC