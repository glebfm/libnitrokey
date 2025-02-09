# libnitrokey
libnitrokey is a project to communicate with Nitrokey Pro and Storage devices in a clean and easy manner. Written in C++14, testable with `py.test` and `Catch` frameworks, with C API, Python access (through CFFI and C API, in future with Pybind11).

The development of this project is aimed to make it itself a living documentation of communication protocol between host and the Nitrokey stick devices. The command packets' format is described here: [Pro v0.7](libnitrokey/stick10_commands.h), [Pro v0.8](libnitrokey/stick10_commands_0.8.h), [Storage](libnitrokey/stick20_commands.h). Handling and additional operations are described here: [NitrokeyManager.cc](NitrokeyManager.cc).

A C++14 complying compiler is required due to heavy use of variable templates. For feature support tables please check [table 1](https://gcc.gnu.org/projects/cxx-status.html#cxx14) or [table 2](http://en.cppreference.com/w/cpp/compiler_support).

libnitrokey is developed and tested with a variety of compilers, starting from g++ 6.2 and clang 3.8. We use Travis CI to test builds also on g++ 5.4 and under OSX compilers starting up from xcode 9 environment. 

## Getting sources
This repository uses `git submodules`.
To clone please use git's `--recursive` option like in:
```bash
git clone --recursive https://github.com/Nitrokey/libnitrokey.git
```
or for already cloned repository:
```bash
git clone https://github.com/Nitrokey/libnitrokey.git
cd libnitrokey
git submodule update --init --recursive
```

## Dependencies
Following libraries are needed to use libnitrokey on Linux (names of the packages on Ubuntu):
- libhidapi-dev [(HID API)](http://www.signal11.us/oss/hidapi/)
- libusb-1.0-0-dev 


## Build
libnitrokey uses CMake as its main build system. As a secondary option it offers building through Qt's qMake.

### Docker Isolated Build

To run a docker isolated build it suffices to run the helper command:

```bash
# build docker image, and execute the build
$ make docker-build-all
```

Currently, Ubuntu 22.04 is used as a build base. Results will be placed in the `./build/` directory.

Additionally, it is possible to check if the Debian package of selected version is able to build itself with the following command:
```text
# for the default 3.7 URL
$ make docker-package

# for the customized DGET url
$ make docker-package DOCKERCMD="make ci-package DGET_URL=https://people.debian.org/~patryk/tmp/libnitrokey/libnitrokey_3.7-1.dsc"
```

### Qt
A Qt's .pro project file is provided for direct compilation and for inclusion to other projects.
Using it directly is not recommended due to lack of dependencies check and not implemented library versioning.
Compilation is tested with Qt 5.6 and greater.

Quick start example:
```bash
mkdir -p build
cd build
qmake ..
make -j2
```

### Windows and Visual Studio 2017
Lately Visual Studio has started handling CMake files directly. After opening the project's directory it should recognize it and initialize build system. Afterwards please run:
1. `CMake -> Cache -> View Cache CMakeLists.txt -> CMakeLists.txt` to edit settings
2. `CMake -> Build All` to build

It is possible too to use CMake GUI directly with its settings editor.

### CMake
To compile please run following sequence of commands:
```bash
# assuming current dir is ./libnitrokey/
mkdir -p build
cd build
cmake .. <OPTIONS>
make -j2
```

By default (with empty `<OPTIONS>` string) this will create in `build/` directory a shared library (.so, .dll or .dynlib). If you wish to build static version you can use as `<OPTIONS>` string `-DBUILD_SHARED_LIBS=OFF`. 

All options could be listed with `cmake .. -L` or instead `cmake` a `ccmake ..` tool could be used for configuration (where `..` is the path to directory with `CMakeLists.txt` file). `ccmake` shows also description of the build parameters.

If you have trouble compiling or running the library you can check [.travis.yml](.travis.yml) file for configuration details. This file is used by Travis CI service to make test builds on OSX and Ubuntu 14.04.

Other build options (all take either `ON` or `OFF`):
* ADD_ASAN - add tests for memory leaks and out-of-bounds access
* ADD_TSAN - add tests for threads race, needs USE_CLANG
* COMPILE_TESTS - compile C++ tests
* COMPILE_OFFLINE_TESTS - compile C++ tests, that do not require any device to be connected
* LOG_VOLATILE_DATA (default: OFF) - include secrets in log (PWS passwords, PINs etc)
* NO_LOG (default: OFF) - do not compile LOG statements - will make library smaller, but without any diagnostic messages


### Meson
Note: Meson build is currently not tested. Please file a ticket in case it would not work for you.

It is possible to use Meson and Ninja to build the project as well.
Please run:
```
meson builddir <OPTIONS>
meson configure builddir # to show available build flags
ninja -C builddir
```

# Using libnitrokey with Python
To use libnitrokey with Python a [CFFI](http://cffi.readthedocs.io/en/latest/overview.html) library is required (either 2.7+ or 3.0+). It can be installed with:
```bash
pip install --user cffi # for python 2.x
pip3 install cffi # for python 3.x
```
## Python2
Note: Python 2 is not supported anymore.

Just import it, read the C API header and it is done! You have access to the library. Here is an example (in Python 2) printing HOTP code for Pro or Storage device, assuming it is run in root directory [(full example)](python_bindings_example.py):

<details>
    <summary>Code snippet (click to show)</summary>


```python
#!/usr/bin/env python2
import cffi

ffi = cffi.FFI()
get_string = ffi.string

def get_library():
    fp = 'NK_C_API.h'  # path to C API header

    declarations = []
    with open(fp, 'r') as f:
        declarations = f.readlines()

    cnt = 0
    a = iter(declarations)
    for declaration in a:
        if declaration.strip().startswith('NK_C_API'):
            declaration = declaration.replace('NK_C_API', '').strip()
            while ';' not in declaration:
                declaration += (next(a)).strip()
            # print(declaration)
            ffi.cdef(declaration, override=True)
            cnt +=1
    print('Imported {} declarations'.format(cnt))


    C = None
    import os, sys
    path_build = os.path.join(".", "build")
    paths = [
            os.environ.get('LIBNK_PATH', None),
            os.path.join(path_build,"libnitrokey.so"),
            os.path.join(path_build,"libnitrokey.dylib"),
            os.path.join(path_build,"libnitrokey.dll"),
            os.path.join(path_build,"nitrokey.dll"),
    ]
    for p in paths:
        if not p: continue
        print("Trying " +p)
        p = os.path.abspath(p)
        if os.path.exists(p):
            print("Found: "+p)
            C = ffi.dlopen(p)
            break
        else:
            print("File does not exist: " + p)
    if not C:
        print("No library file found")
        sys.exit(1)

    return C


def get_hotp_code(lib, i):
    return lib.NK_get_hotp_code(i)


libnitrokey = get_library()
libnitrokey.NK_set_debug(False)  # do not show debug messages (log library only)

hotp_slot_code = get_hotp_code(libnitrokey, 1)
print('Getting HOTP code from Nitrokey device: ')
print(hotp_slot_code)
libnitrokey.NK_logout()  # disconnect device
```

</details>

In case  no devices are connected, a friendly message will be printed.
All available functions for C and Python are listed in [NK_C_API.h](NK_C_API.h). Please check `Documentation` section below.

## Python3
Just import it, read the C API header and it is done! You have access to the library. Here is an example (in Python 3) printing HOTP code for Pro or Storage device, assuming it is run in root directory [(full example)](python3_bindings_example.py):

<details>
    <summary>Code snippet (click to show)</summary>


```python
#!/usr/bin/env python3
import cffi

ffi = cffi.FFI()
get_string = ffi.string

def get_library():
    fp = 'NK_C_API.h'  # path to C API header

    declarations = []
    with open(fp, 'r') as f:
        declarations = f.readlines()

    cnt = 0
    a = iter(declarations)
    for declaration in a:
        if declaration.strip().startswith('NK_C_API'):
            declaration = declaration.replace('NK_C_API', '').strip()
            while ';' not in declaration:
                declaration += (next(a)).strip()
            # print(declaration)
            ffi.cdef(declaration, override=True)
            cnt +=1
    print('Imported {} declarations'.format(cnt))


    C = None
    import os, sys
    path_build = os.path.join(".", "build")
    paths = [
            os.environ.get('LIBNK_PATH', None),
            os.path.join(path_build,"libnitrokey.so"),
            os.path.join(path_build,"libnitrokey.dylib"),
            os.path.join(path_build,"libnitrokey.dll"),
            os.path.join(path_build,"nitrokey.dll"),
    ]
    for p in paths:
        if not p: continue
        print("Trying " +p)
        p = os.path.abspath(p)
        if os.path.exists(p):
            print("Found: "+p)
            C = ffi.dlopen(p)
            break
        else:
            print("File does not exist: " + p)
    if not C:
        print("No library file found")
        sys.exit(1)

    return C


def get_hotp_code(lib, i):
    return lib.NK_get_hotp_code(i)

def connect_device(lib):
	# lib.NK_login('S'.encode('ascii'))  # connect only to Nitrokey Storage device
	# lib.NK_login('P'.encode('ascii'))  # connect only to Nitrokey Pro device
	device_connected = lib.NK_login_auto()  # connect to any Nitrokey Stick
	if device_connected:
		print('Connected to Nitrokey device!')
	else:
	    print('Could not connect to Nitrokey device!')
	    exit()

libnitrokey = get_library()
libnitrokey.NK_set_debug(False)  # do not show debug messages (log library only)

connect_device(libnitrokey)

hotp_slot_code = get_hotp_code(libnitrokey, 1)
print('Getting HOTP code from Nitrokey device: ')
print(ffi.string(hotp_slot_code).decode('ascii'))
libnitrokey.NK_logout()  # disconnect device
```

</details>

In case  no devices are connected, a friendly message will be printed.
All available functions for C and Python are listed in [NK_C_API.h](NK_C_API.h). Please check `Documentation` section below.

## Documentation
The documentation of C API is included in the sources (can be generated with `make doc` if Doxygen is installed).
Please check [NK_C_API.h](NK_C_API.h) (C API) for high level commands and [libnitrokey/NitrokeyManager.h](libnitrokey/NitrokeyManager.h) (C++ API). All devices' commands are listed along with packet format in [libnitrokey/stick10_commands.h](libnitrokey/stick10_commands.h) and [libnitrokey/stick20_commands.h](libnitrokey/stick20_commands.h) respectively for Nitrokey Pro and Nitrokey Storage products.

# Tests
**Warning!** Most of the tests will overwrite user data. The only user-data safe tests are specified in `unittest/test_safe.cpp` (see *C++ tests* chapter).

**Warning!** Before you run unittests please change both your Admin and User PINs on your Nitrostick to defaults (`12345678` and `123456` respectively), or change the values in tests source code. If you do not change them, the tests might lock your device temporarily. If it's too late already, you can reset your Nitrokey using instructions from [homepage](https://www.nitrokey.com/de/documentation/how-reset-nitrokey).

## Helper

Here are Python tests helper calls:

```bash
# setup, requires pipenv installed
$ make tests-setup
# For Nitrokey Pro
$ make tests-pro
# For Nitrokey Storage
$ make tests-storage
```

See below for the further information.

## Python tests
libnitrokey has a great suite of tests written in Python 3 under the path: [unittest/test_*.py](https://github.com/Nitrokey/libnitrokey/tree/master/unittest): 
* `test_pro.py` - contains tests of OTP, Password Safe and PIN control functionality. Could be run on both Pro and Storage devices.
* `test_storage.py` - contains tests of Encrypted Volumes functionality. Could be run only on Storage.

The tests themselves show how to handle common requests to device.
Before running please install all required libraries with:
```bash
cd unittest
pip install --user -r requirements.txt
```
or use Python's environment managing tool like [pipenv](https://pipenv.readthedocs.io/en/latest/) or `virtualenv`.


To run them please execute: 
```bash
# substitute <dev> with either 'pro' or 'storage'
py.test -v test_<dev>.py
# more specific use - run tests containing in name <test_name> 5 times:
py.test -v test_<dev>.py -k <test_name> --count 5

```
For additional documentation please check the following for [py.test installation](http://doc.pytest.org/en/latest/getting-started.html). For better coverage [randomly plugin](https://pypi.python.org/pypi/pytest-randomly) is installed - it randomizes the test order allowing to detect unseen dependencies between the tests.

## C++ tests
There are also some unit tests implemented in C++, placed in unittest directory. The only user-data safe online test set here is [test_safe.cpp](https://github.com/Nitrokey/libnitrokey/blob/master/unittest/test_safe.cpp), which tries to connect to the device, and collect its status data. Example run for Storage:

<details>
    <summary>Log (click to show)</summary>

```text
# Storage device inserted, firmware version v0.53
$ ./test_safe
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    => GET_DEVICE_STATUS
..
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    <= GET_DEVICE_STATUS 0 1
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    => GET_PASSWORD_RETRY_COUNT
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    <= GET_PASSWORD_RETRY_COUNT 0 0
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    => GET_DEVICE_STATUS
..
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    <= GET_DEVICE_STATUS 0 1
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    => GET_USER_PASSWORD_RETRY_COUNT
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    <= GET_USER_PASSWORD_RETRY_COUNT 0 0
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    => GET_DEVICE_STATUS
...
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    <= GET_DEVICE_STATUS 0 1
 transmission_data.dissect():   _padding:
0000    00 00 00 00 00 00 00 00 00 00 00 00 00 05 2e 01   ................
0010    00 00 -- -- -- -- -- -- -- -- -- -- -- -- -- --   ..
 (int) SendCounter_u8:  0
 (int) SendDataType_u8: 3
 (int) FollowBytesFlag_u8:      0
 (int) SendSize_u8:     28

 MagicNumber_StickConfig_u16:   13080
 (int) ReadWriteFlagUncryptedVolume_u8: 1
 (int) ReadWriteFlagCryptedVolume_u8:   0
 (int) ReadWriteFlagHiddenVolume_u8:    0
 (int) versionInfo.major:       0
 (int) versionInfo.minor:       53
 (int) versionInfo.build_iteration:     0
 (int) FirmwareLocked_u8:       0
 (int) NewSDCardFound_u8:       1
 (int) NewSDCardFound_st.NewCard:       1
 (int) NewSDCardFound_st.Counter:       0
 (int) SDFillWithRandomChars_u8:        1
 ActiveSD_CardID_u32:   3670817656
 (int) VolumeActiceFlag_u8:     1
 (int) VolumeActiceFlag_st.unencrypted: 1
 (int) VolumeActiceFlag_st.encrypted:   0
 (int) VolumeActiceFlag_st.hidden:      0
 (int) NewSmartCardFound_u8:    0
 (int) UserPwRetryCount:        3
 (int) AdminPwRetryCount:       3
 ActiveSmartCardID_u32: 24122
 (int) StickKeysNotInitiated:   0

[Wed Jan  2 13:31:17 2019][DEBUG_L1]    => GET_DEVICE_STATUS
..
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    <= GET_DEVICE_STATUS 0 1
00005e3a
[Wed Jan  2 13:31:17 2019][DEBUG_L1]    => GET_DEVICE_STATUS
....
[Wed Jan  2 13:31:18 2019][DEBUG_L1]    <= GET_DEVICE_STATUS 0 1
[Wed Jan  2 13:31:18 2019][DEBUG_L1]    => GET_DEVICE_STATUS
...
[Wed Jan  2 13:31:18 2019][DEBUG_L1]    <= GET_DEVICE_STATUS 0 1
===============================================================================
All tests passed (18 assertions in 6 test cases)
```

</details>


Test's execution configuration and verbosity could be manipulated - please see `./test_safe --help` for details.

The other tests sets are not written as extensively as Python tests and are rather more a C++ low level interface check used during the library development, using either low-level components, C API from `NK_C_API.cc`, or C++ API from `NitrokeyManager.cc`. Some of them are: [test_HOTP.cc](https://github.com/Nitrokey/libnitrokey/blob/master/unittest/test_HOTP.cc),
[test1.cc](https://github.com/Nitrokey/libnitrokey/blob/master/unittest/test1.cc). See more in [unittest](https://github.com/Nitrokey/libnitrokey/tree/master/unittest) directory.

**Note: these are not device model agnostic, and will most probably destroy your data on the device.**


Unit tests were checked on Ubuntu 16.04/16.10/17.04. To run them just execute binaries built in `./libnitrokey/build` dir, after enabling them by passing `-DCOMPILE_TESTS=ON` option to `cmake` - e.g.: `cmake .. -DCOMPILE_TESTS=ON && make`. 


The documentation of how it works could be found in nitrokey-app project's README on Github:
[Nitrokey-app - internals](https://github.com/Nitrokey/nitrokey-app/blob/master/README.md#internals).

To peek/debug communication with device running nitrokey-app (0.x branch) in debug mode (`-d` switch) and checking the logs
(right click on tray icon and then 'Debug') might be helpful. Latest Nitrokey App (1.x branch) uses libnitrokey to communicate with device. Once run with `--dl 3` (3 or higher; range 0-5) it will print all communication to the console. Additionally crosschecking with
firmware code should show how things works:
[report_protocol.c](https://github.com/Nitrokey/nitrokey-pro-firmware/blob/master/src/keyboard/report_protocol.c)
(for Nitrokey Pro, for Storage similarly).

# Known issues / tasks
* C++ API needs some reorganization to C++ objects (instead of pointers to byte arrays). This would be also preparation for Pybind11 integration;
* Fix compilation warnings.

Other tasks might be listed either in [TODO](TODO) file or on project's issues page.

# License
This project is licensed under LGPL version 3. License text can be found under [LICENSE](LICENSE) file.

# Roadmap
To check what issues will be fixed and when please check [milestones](https://github.com/Nitrokey/libnitrokey/milestones) page.

# Nitrokey USB IDs

Currently used USB identifiers for Nitrokey products are listed below. See [./data](./data) for the further details.

| Name                        |  USB ID   |
|-----------------------------|:---------:|
| Crypto Stick 1.2            | 20a0:4107 |
| Nitrokey 3                  | 20a0:42b2 |
| Nitrokey 3 Bootloader       | 20a0:42dd |
| Nitrokey 3 Bootloader NRF   | 20a0:42e8 |
| Nitrokey FIDO U2F           | 20a0:4287 |
| Nitrokey FIDO2              | 20a0:42b1 |
| Nitrokey HSM                | 20a0:4230 |
| Nitrokey Pro                | 20a0:4108 |
| Nitrokey Pro Bootloader     | 20a0:42b4 |
| Nitrokey Start              | 20a0:4211 |
| Nitrokey Storage            | 20a0:4109 |
| Nitrokey Storage Bootloader | 03eb:2ff1 |
| Nitrokey U2F                | 2581:f1d0 |