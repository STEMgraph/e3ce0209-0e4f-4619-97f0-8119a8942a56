<!---
{
  "id": "e3ce0209-0e4f-4619-97f0-8119a8942a56",
  "depends_on": ["a9c9305e-826b-41e2-8235-bb1315b07db9", "ba108c71-19c6-4063-b813-fdbbe0b5d775"],
  "author": "Stephan Bökelmann",
  "first_used": "2025-12-04",
  "keywords": ["C++", "CMake", "Installation", "find_package", "Packaging", "Config files"]
}
--->

# Installing a C++ Library with CMake and Using It in a Separate Project via `find_package`

> In this exercise you will learn how to build and install a self-written C++ library using CMake’s `install()` commands. Furthermore you will learn how to configure a separate application project so that it can automatically locate and use the installed library via `find_package()`.

## Introduction

Most professional C++ libraries are **installed** on the system (or in a custom prefix such as `/usr/local`, `/opt`, or a dev workspace). After installation, other CMake projects can discover and use them without needing to reference their source directory.

This works by providing **CMake package configuration files**, typically installed as:

```
<install-prefix>/lib/cmake/<ProjectName>/<ProjectName>Config.cmake
```

These package files define imported targets (e.g., `mylib::mylib`) that can be consumed via:

```cmake
find_package(mylib REQUIRED)
target_link_libraries(app PRIVATE mylib::mylib)
```

In this exercise, you will:

1. Create a **compiled library** with its own `CMakeLists.txt`.
2. Add **installation rules** for headers, libraries, and CMake config files.
3. Run `cmake --install` to place the library into an installation prefix.
4. Create a **separate main project** that uses `find_package()` to locate and link the installed library **without hardcoded paths**.

This exercise introduces foundational concepts for packaging C++ libraries in CMake, preparing you for reusable modules, system installs, and dependency management.

### Further Readings and Other Sources

* Installing libraries with CMake: [https://cmake.org/cmake/help/latest/guide/importing-exporting/index.html](https://cmake.org/cmake/help/latest/guide/importing-exporting/index.html)
* Modern CMake package files: [https://cliutils.gitlab.io/modern-cmake/chapters/install/installing.html](https://cliutils.gitlab.io/modern-cmake/chapters/install/installing.html)
* YouTube: *CMake Packaging and Installing Explained*
* C++ Core Guidelines: [https://isocpp.github.io/CppCoreGuidelines](https://isocpp.github.io/CppCoreGuidelines)

---

# Tasks

---

# **Task 1 — Create and Install the Library**

## 1. Create the library directory structure

```bash
mkdir myinstalllib
cd myinstalllib
mkdir -p include/myinstalllib src
```

Directory roles:

* `include/myinstalllib/` → public headers
* `src/` → implementation files
* `build/` → out-of-source build directory

---

## 2. Write the library code

### `include/myinstalllib/greeter.h`

```cpp
#pragma once
#include <string>

namespace myinstalllib {
    std::string greet(const std::string& name);
}
```

### `src/greeter.cpp`

```cpp
#include "myinstalllib/greeter.h"

namespace myinstalllib {
    std::string greet(const std::string& name) {
        return "Hello, " + name + "!";
    }
}
```

---

## 3. Create the library `CMakeLists.txt`

**`myinstalllib/CMakeLists.txt`:**

```cmake
cmake_minimum_required(VERSION 3.12)
project(myinstalllib VERSION 1.0 LANGUAGES CXX)

add_library(myinstalllib
    src/greeter.cpp
)

target_include_directories(myinstalllib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

# Export target under namespace "myinstalllib::"
add_library(myinstalllib::myinstalllib ALIAS myinstalllib)

include(GNUInstallDirs)

# Install the library and headers
install(TARGETS myinstalllib
    EXPORT myinstalllibTargets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# Install the export file
install(EXPORT myinstalllibTargets
    FILE myinstalllibTargets.cmake
    NAMESPACE myinstalllib::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myinstalllib
)

# Create a Config file so find_package(myinstalllib) works
include(CMakePackageConfigHelpers)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/myinstalllibConfig.cmake"
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myinstalllib
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/myinstalllibConfig.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myinstalllib
)
```

### Create the config template

Make a directory:

```bash
mkdir cmake
```

Then create `cmake/Config.cmake.in`:

```cmake
@PACKAGE_INIT@
include("${CMAKE_CURRENT_LIST_DIR}/myinstalllibTargets.cmake")
```

---

## 4. Build and install the library

```bash
cmake -B build -DCMAKE_INSTALL_PREFIX=./install
cmake --build build
cmake --install build
```

This installs:

```
install/
 ├─ include/myinstalllib/greeter.h
 ├─ lib/libmyinstalllib.a (or .so)
 └─ lib/cmake/myinstalllib/
        myinstalllibConfig.cmake
        myinstalllibTargets.cmake
```

Your library is now discoverable with `find_package(myinstalllib)`.

---

# **Task 2 — Create a Separate Project That Uses the Installed Library**

## 1. Create a new project folder *outside* the library directory

```bash
mkdir ../myapp
cd ../myapp
mkdir src build
```

---

## 2. Write the main application

**`src/main.cpp`:**

```cpp
#include <iostream>
#include <myinstalllib/greeter.h>

int main() {
    std::cout << myinstalllib::greet("World") << std::endl;
    return 0;
}
```

---

## 3. Create the application `CMakeLists.txt`

**Important:** You do **not** specify include paths or library paths manually.

```cmake
cmake_minimum_required(VERSION 3.12)
project(myapp LANGUAGES CXX)

# Tell CMake where to look for installed packages
list(APPEND CMAKE_PREFIX_PATH "../myinstalllib/install")

find_package(myinstalllib REQUIRED)

add_executable(myapp src/main.cpp)

target_link_libraries(myapp PRIVATE myinstalllib::myinstalllib)
```

---

## 4. Build the application

```bash
cmake -B build
cmake --build build
./build/myapp
```

Expected output:

```
Hello, World!
```

Your application successfully found and linked the library **without** using any manual include or library path settings.

---

# Questions

### **1. Why does `find_package(myinstalllib)` work only after running `cmake --install`?**

<details>
<summary>Click to reveal answer</summary>

Because the installation step generates and places the required CMake package files (`myinstalllibConfig.cmake` and target exports) into a structure that CMake can search. Before installation, these files do not exist.

</details>

---

### **2. Why do we use `INSTALL_INTERFACE` and `BUILD_INTERFACE` include directories?**

<details>
<summary>Click to reveal answer</summary>

`BUILD_INTERFACE` is used when building directly from the source tree;
`INSTALL_INTERFACE` is used after installation when the library is used by other projects.
This separation ensures the library works in both contexts.

</details>

---

### **3. Why is a namespace (`myinstalllib::`) used for the CMake target?**

<details>
<summary>Click to reveal answer</summary>

Namespaces prevent name collisions between different libraries and follow modern CMake best practices. Installed packages almost always provide namespaced targets.

</details>

---

# Advice

Mastering installation and package configuration is one of the biggest steps toward writing reusable, professional C++ libraries. Once you understand how `install()`, export files, and configuration files work together, you can begin distributing your libraries across systems, CI pipelines, or team repositories. Modern CMake encourages this style, because it allows consumers to depend on your library with a single, elegant line:

```cmake
find_package(myinstalllib REQUIRED)
```
