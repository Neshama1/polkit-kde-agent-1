cmake_minimum_required(VERSION 3.0)

project(polkit-kde-agent-1)
set(PROJECT_VERSION "5.21.80")

set(QT_MIN_VERSION "5.15.0")
set(KF5_MIN_VERSION "5.78")

# we need some parts of the ECM CMake helpers
find_package(ECM ${KF5_MIN_VERSION} REQUIRED NO_MODULE)
set (CMAKE_MODULE_PATH ${ECM_MODULE_PATH})

include(KDEInstallDirs)
include(KDECMakeSettings)
include(KDECompilerSettings NO_POLICY_SCOPE)
include(KDEClangFormat)
include(FeatureSummary)
include(ECMConfiguredInstall)

if(ECM_VERSION VERSION_GREATER_EQUAL 5.79)
    include(KDEGitCommitHooks)
endif()

find_package(Qt5 ${QT_MIN_VERSION} CONFIG REQUIRED Core DBus Widgets)

find_package(Qt5 ${QT_MIN_VERSION} CONFIG REQUIRED Core Qml Quick)

find_package(KF5 ${KF5_MIN_VERSION} REQUIRED COMPONENTS
    I18n
    KIO
    Plasma
    PlasmaQuick
)
# Load the frameworks we need
find_package(KF5 ${KF5_MIN_VERSION} REQUIRED COMPONENTS
  I18n
  WindowSystem
  DBusAddons
  WidgetsAddons
  CoreAddons
  Crash
  IconThemes
)
include(CheckIncludeFile)
include(CheckSymbolExists)
check_include_file("sys/prctl.h" HAVE_SYS_PRCTL_H)
check_symbol_exists(PR_SET_DUMPABLE "sys/prctl.h" HAVE_PR_SET_DUMPABLE)
check_include_file("sys/procctl.h" HAVE_SYS_PROCCTL_H)
check_symbol_exists(PROC_TRACE_CTL "sys/procctl.h" HAVE_PROC_TRACE_CTL)
if (HAVE_PR_SET_DUMPABLE OR HAVE_PROC_TRACE_CTL)
  set(CAN_DISABLE_PTRACE TRUE)
endif ()
add_feature_info("prctl/procctl tracing control"
                 CAN_DISABLE_PTRACE
                 "Required for disallowing ptrace on polkit agent")

# create config file
configure_file (config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h)

# let our config.h be found first in any case
include_directories (BEFORE ${CMAKE_CURRENT_BINARY_DIR})

find_package(PolkitQt5-1 REQUIRED 0.103.0)
if (EXISTS "${CMAKE_SOURCE_DIR}/.git")
   add_definitions(-DQT_DISABLE_DEPRECATED_BEFORE=0x060000)
   add_definitions(-DKF_DISABLE_DEPRECATED_BEFORE_AND_AT=0x054200)
endif()

include_directories(
                    ${POLKITQT-1_INCLUDE_DIR}
                    ${CMAKE_CURRENT_SOURCE_DIR}
                    ${CMAKE_CURRENT_BINARY_DIR})

qt5_add_dbus_adaptor(policykit_SRCS org.kde.Polkit1AuthAgent.xml policykitlistener.h PolicyKitListener)


set(policykit_SRCS
    ${policykit_SRCS}
    policykitlistener.cpp
    main.cpp
)

add_executable(polkit-kde-authentication-agent-1 ${policykit_SRCS})

target_link_libraries(polkit-kde-authentication-agent-1
    KF5::DBusAddons
    KF5::WindowSystem
    KF5::WidgetsAddons
    KF5::CoreAddons
    KF5::I18n
    KF5::Crash
    KF5::IconThemes
    Qt5::Gui
    KF5::Plasma
    Qt5::Qml
    Qt5::Quick
    KF5::Service
    KF5::KIOGui
    KF5::Solid
    crypt
    ${POLKITQT-1_LIBRARIES}
)

configure_file(polkit-kde-authentication-agent-1.desktop.cmake ${CMAKE_BINARY_DIR}/polkit-kde-authentication-agent-1.desktop)

install(TARGETS polkit-kde-authentication-agent-1 DESTINATION ${KDE_INSTALL_LIBEXECDIR})

install(DIRECTORY DESTINATION /usr/share/jingostest/systemui/org.jingtest.polkit-kde-authentication-agent-1)
install(FILES ./authdialog.qml DESTINATION  /usr/share/jingostest/systemui/org.jingtest.polkit-kde-authentication-agent-1/ )

install(FILES ${CMAKE_BINARY_DIR}/polkit-kde-authentication-agent-1.desktop DESTINATION ${KDE_INSTALL_AUTOSTARTDIR})
install(FILES ${CMAKE_BINARY_DIR}/polkit-kde-authentication-agent-1.desktop DESTINATION ${XDG_APPS_INSTALL_DIR} RENAME org.kde.polkit-kde-authentication-agent-1.desktop)

ecm_install_configured_files(INPUT plasma-polkit-agent.service.in DESTINATION ${KDE_INSTALL_SYSTEMDUSERUNITDIR})

install(FILES policykit1-kde.notifyrc DESTINATION ${KDE_INSTALL_KNOTIFY5RCDIR})

# add clang-format target for all our real source files
file(GLOB_RECURSE ALL_CLANG_FORMAT_SOURCE_FILES *.cpp *.h)
kde_clang_format(${ALL_CLANG_FORMAT_SOURCE_FILES})

if(ECM_VERSION VERSION_GREATER_EQUAL 5.79)
    kde_configure_git_pre_commit_hook(CHECKS CLANG_FORMAT)
endif()
feature_summary(WHAT ALL FATAL_ON_MISSING_REQUIRED_PACKAGES)

find_package(KF5I18n CONFIG REQUIRED)
ki18n_install(po)