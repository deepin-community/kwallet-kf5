project(kwalletd5)

find_package(Qt5 ${REQUIRED_QT_VERSION} CONFIG REQUIRED Gui)

find_package(KF5Config ${KF_DEP_VERSION} REQUIRED)
find_package(KF5ConfigWidgets ${KF_DEP_VERSION} REQUIRED)
find_package(KF5CoreAddons ${KF_DEP_VERSION} REQUIRED)
find_package(KF5DBusAddons ${KF_DEP_VERSION} REQUIRED)
find_package(KF5Notifications ${KF_DEP_VERSION} REQUIRED)
find_package(KF5Service ${KF_DEP_VERSION} REQUIRED)
find_package(KF5WidgetsAddons ${KF_DEP_VERSION} REQUIRED)
find_package(KF5WindowSystem ${KF_DEP_VERSION} REQUIRED)

########### find needed packages ######
find_package(Gpgmepp 1.7.0) # provided by GpgME

if (Gpgmepp_FOUND)
    message("GPG support enabled")
    add_definitions(-DHAVE_GPGMEPP)
    add_definitions(-DBOOST_NO_EXCEPTIONS)
    include_directories(${GPGME_INCLUDES})
endif(Gpgmepp_FOUND)


include_directories(${CMAKE_CURRENT_BINARY_DIR})
########### build backends #########
add_subdirectory(backend)

########### kwalletd ###############

add_executable(kwalletd5)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/backend)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/../../api/KWallet)
include_directories(${CMAKE_CURRENT_BINARY_DIR}/../../api/KWallet)

remove_definitions(-DQT_NO_CAST_FROM_ASCII)

ecm_setup_version(${KF_VERSION} VARIABLE_PREFIX KWALLETD
                        VERSION_HEADER "${CMAKE_CURRENT_BINARY_DIR}/kwalletd_version.h"
                        PACKAGE_VERSION_FILE "${CMAKE_CURRENT_BINARY_DIR}/KF5WalletConfigVersion.cmake"
                        SOVERSION 5)

target_sources(kwalletd5 PRIVATE
   main.cpp
   kbetterthankdialog.cpp
   kwalletd.cpp
   kwalletwizard.cpp
   ktimeout.cpp
   kwalletsessionstore.cpp
)
ecm_qt_declare_logging_category(kwalletd5
    HEADER kwalletd_debug.h
    IDENTIFIER KWALLETD_LOG
    CATEGORY_NAME kf.wallet.kwalletd
    OLD_CATEGORY_NAMES kf5.kwallet.kwalletd
    DESCRIPTION "kwalletd"
    EXPORT KWALLET
)

ki18n_wrap_ui(kwalletd5
   kbetterthankdialogbase.ui
   kwalletwizardpageexplanation.ui
   kwalletwizardpageintro.ui
   kwalletwizardpageoptions.ui
   kwalletwizardpagepassword.ui
)

if (Gpgmepp_FOUND)
    target_sources(kwalletd5 PRIVATE
        knewwalletdialog.cpp
    )
    ki18n_wrap_ui(kwalletd5
        kwalletwizardpagepasswordgpg.ui
        kwalletwizardpagegpgkey.ui
        knewwalletdialogintro.ui
        knewwalletdialoggpg.ui
    )
endif(Gpgmepp_FOUND)

if (NOT EXCLUDE_DEPRECATED_BEFORE_AND_AT STREQUAL "CURRENT" AND
    EXCLUDE_DEPRECATED_BEFORE_AND_AT VERSION_LESS 5.72.0)
    set(kwallet_xml ${CMAKE_SOURCE_DIR}/src/api/KWallet/org.kde.KWallet.xml)
else()
    # copy of org.kde.KWallet.xml, but with all deprecated API removed
    set(kwallet_xml ${CMAKE_SOURCE_DIR}/src/api/KWallet/org.kde.KWallet.nodeprecated.xml)
endif()

set(kwalletd_dbus_SRCS)
qt_add_dbus_adaptor(kwalletd_dbus_SRCS ${kwallet_xml} kwalletd.h KWalletD kwalletadaptor KWalletAdaptor)
target_sources(kwalletd5 PRIVATE
    ${kwalletd_dbus_SRCS}
)

if(WIN32)
    configure_file(org.kde.kwalletd5.service.win.in
                   ${CMAKE_CURRENT_BINARY_DIR}/org.kde.kwalletd5.service)
else()
    configure_file(org.kde.kwalletd5.service.in
                   ${CMAKE_CURRENT_BINARY_DIR}/org.kde.kwalletd5.service)
endif()

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/org.kde.kwalletd5.service
        DESTINATION ${KDE_INSTALL_DBUSSERVICEDIR})

target_link_libraries(kwalletd5
    kwalletbackend5
    KF5Wallet
    Qt5::Widgets
    KF5::I18n
    KF5::CoreAddons
    KF5::ConfigCore
    KF5::ConfigWidgets
    KF5::Service
    KF5::DBusAddons
    KF5::WidgetsAddons
    KF5::WindowSystem
    KF5::Notifications)
if (Gpgmepp_FOUND)
    target_link_libraries(kwalletd5 Gpgmepp)
    kde_target_enable_exceptions(kwalletd5 PRIVATE)
endif(Gpgmepp_FOUND)

install(TARGETS kwalletd5  ${KF5_INSTALL_TARGETS_DEFAULT_ARGS})

########### install files ###############

install( FILES kwalletd5.desktop  DESTINATION  ${KDE_INSTALL_KSERVICES5DIR} )
install( FILES kwalletd5.notifyrc DESTINATION  ${KDE_INSTALL_KNOTIFY5RCDIR} )
install( FILES org.kde.kwalletd5.desktop DESTINATION ${KDE_INSTALL_APPDIR} )

