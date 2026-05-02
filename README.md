# Лабораторная работа №6

Данная лабораторная работа посвещена изучению средств пакетирования на примере **CPack**

## Homework

Для создания релиза проекта нужны две вещи: файлы CMake и файлы CI.

### CMake

Рассмотрим главный `CMakeLists.txt` всего проекта:
```cmake
cmake_minimum_required(VERSION 3.10)
project(examples)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(PRINT_VERSION_MAJOR 1)
set(PRINT_VERSION_MINOR 0)
set(PRINT_VERSION_PATCH 0)
set(PRINT_VERSION_TWEAK 0)
set(PRINT_VERSION ${PRINT_VERSION_MAJOR}.${PRINT_VERSION_MINOR}.${PRINT_VERSION_PATCH}.${PRINT_VERSION_TWEAK})
set(PRINT_VERSION_STRING "v${PRINT_VERSION}")

add_subdirectory(formatter_lib)
add_subdirectory(formatter_ex_lib)
add_subdirectory(solver_lib)
add_subdirectory(hello_world_application)
add_subdirectory(solver_application)

install(TARGETS solver
        DESTINATION bin
        COMPONENT applications)

include(CPackConfig.cmake)
```

По сравнению с ЛР3 тут добавляется инициализация набора переменных, отвечающих текущей версии проекта, команда install, которая скопирует приложение `solver` в папку `bin` (это надо для CPack). И также нужно подключить файл конфигурации для пакетов, собранных CPack. Посмотрим на этот файл:
```cmake
include(InstallRequiredSystemLibraries)

set(CPACK_PACKAGE_CONTACT rchekr@yandex.ru)
set(CPACK_PACKAGE_VERSION_MAJOR ${PRINT_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PRINT_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PRINT_VERSION_PATCH})
set(CPACK_PACKAGE_VERSION_TWEAK ${PRINT_VERSION_TWEAK})
set(CPACK_PACKAGE_VERSION ${PRINT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "solver library")

set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)
set(CPACK_RESOURCE_FILE_README ${CMAKE_CURRENT_SOURCE_DIR}/README.md)

set(CPACK_RPM_PACKAGE_NAME "solver")
set(CPACK_RPM_PACKAGE_LICENSE "MIT")
set(CPACK_RPM_PACKAGE_GROUP "solver")
set(CPACK_RPM_PACKAGE_RELEASE 1)
set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")

set(CPACK_DEBIAN_PACKAGE_NAME "libsolver-dev")
set(CPACK_DEBIAN_PACKAGE_PREDEPENDS "cmake >= 3.0")
set(CPACK_DEBIAN_PACKAGE_RELEASE 1)

if(WIN32)
    set(CPACK_GENERATOR "WIX")
    set(CPACK_WIX_ARCHITECTURE "x64")
    set(CPACK_WIX_PRODUCT_GUID "05b23be7-c85f-40fb-942c-7a678101c21a")
    set(CPACK_WIX_LICENSE_RTF "${CMAKE_SOURCE_DIR}/LICENSE.rtf")
    set(CPACK_WIX_PROGRAM_MENU_FOLDER "out")
    set(CPACK_WIX_VERSION "4")
    set(CPACK_COMPONENTS_ALL applications)
elseif(APPLE)
    set(CPACK_GENERATOR "DragNDrop")
    set(CPACK_DMG_VOLUME_NAME "solver")
    set(CPACK_DMG_FORMAT "UDBZ")
else()
    set(CPACK_GENERATOR "DEB;RPM;TGZ;ZIP")
endif()

include(CPack)
```

Здесь мы сначала задаем общие переменные Cpack, такие как версия проекта, название, различные пути и т.д., а потом специализированно для различных ОС задаем настройки конкретных форматов создаваемых архивов.

### Continuous integration

А теперь рассмотрим файл CI:
```yaml
name: build and release
on:
  push:
    tags: [ "v*" ]
  workflow_dispatch:


permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Get repo files
        uses: actions/checkout@v6
      - name: Configure build directory
        run: cmake -H. -B build
      - name: Build project
        run: cmake --build build
      - name: Upload artifacts
        uses: actions/upload-artifact@v6
        with:
          name: Applications
          path: |
            ./build/hello_world_application/hello_world
            ./build/solver_application/solver

  test:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v6
        with:
          name: Applications
          path: ./apps
      - name: Fix permissions
        run: |
          chmod +x ./apps/hello_world_application/hello_world
          chmod +x ./apps/solver_application/solver
      - name: Test hello_world
        run: ./apps/hello_world_application/hello_world
      - name: Test solver
        run: echo "1 5 -6" | ./apps/solver_application/solver

  create-binary-packages:
    needs: test
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]

    steps:
      - name: Get repo files
        uses: actions/checkout@v6
      - name: Install required dependencies Ubuntu
        if: matrix.os == 'ubuntu-latest'
        run: |
          sudo apt-get update
          sudo apt-get install -y rpm
      - name: Set up DotNet Windows
        if: matrix.os == 'windows-latest'
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.x'
      - name: Install required dependencies Windows
        if: matrix.os == 'windows-latest'
        run: |
          dotnet tool install --global wix --version 4.0.0
          wix extension add --global WixToolset.UI.wixext/4.0.4
      - name: Configure build directory Windows
        if: matrix.os == 'windows-latest'
        run: |
          mkdir build
          cmake -B build
      - name: Configure build directory Ubuntu and MacOS
        if: matrix.os != 'windows-latest'
        run: cmake -H. -B build
      - name: Build project
        run: cmake --build build --config Release
      - name: Create packages DEB RPM
        if: matrix.os == 'ubuntu-latest'
        run: |
          cd build
          cpack -G DEB -C Release
          cpack -G RPM -C Release
      - name: Create packages DMG
        if: matrix.os == 'macos-latest'
        run: |
          cd build
          cpack -G DragNDrop -C Release
      - name: Create packages MSI
        if: matrix.os == 'windows-latest'
        run: |
          cd build
          cpack -G WIX -C Release
      - name: Upload binary packages
        uses: softprops/action-gh-release@v1
        with:
          files: |
            build/*.deb
            build/*.rpm
            build/*.dmg
            build/*.msi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Этот скрипт можно разделить на 2 части: предварительная сборка и микро-тестирование и сборка бинарных пакетов. Первая часть уже была показана в ЛР4, а вторую мы сейчас разберем. Вначале затребуем успешное выполнение джоба `test` и сделаем матрицу с тремя ОС, так как часть кода там повторяется. Дальше устанавливаем различные зависимости индивидуально для каждой ОС и собираем программу, также учитывая особенности операционок. И наконец собираем резилы программы разных форматов (для `.deb` и `.rpm` - Linux, для `.dmg` - MacOS, для `.msi` - Windows). После чего с помощью действия `action-gh-release` формируем релиз из архивов. В этот момент также автоматически соберутся `.zip` и `.tar.gz` всего исходного кода, поэтому для них ничего дополнительно писать не надо.