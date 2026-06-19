cmake_minimum_required(VERSION 3.20)
project(SkylerPlasmaTaskManager VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)

# ── KDE ECM ──────────────────────────────────────────
find_package(ECM REQUIRED CONFIG)
set(CMAKE_MODULE_PATH ${ECM_MODULE_PATH} ${ECM_KDE_MODULE_DIR})

# Set KDE install dirs before any find_package that pulls in GNUInstallDirs
if(NOT DEFINED KDE_INSTALL_PLUGINDIR)
    set(KDE_INSTALL_PLUGINDIR "${CMAKE_INSTALL_PREFIX}/lib/qt6/plugins" CACHE PATH "KDE plugin install dir")
endif()
if(NOT DEFINED KDE_INSTALL_QMLDIR)
    set(KDE_INSTALL_QMLDIR "${CMAKE_INSTALL_PREFIX}/lib/qt6/qml" CACHE PATH "QML install dir")
endif()

# Plasma macros set LIBRARY_OUTPUT_DIRECTORY based on CMAKE_LIBRARY_OUTPUT_DIRECTORY
if(NOT DEFINED CMAKE_LIBRARY_OUTPUT_DIRECTORY)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
endif()

# ── Qt ───────────────────────────────────────────────
find_package(Qt6 REQUIRED COMPONENTS Core Qml Quick)

# ── KF6 (separate packages for cleaner dependency management) ──
find_package(KF6 REQUIRED COMPONENTS I18n Notifications Service WindowSystem)
find_package(KF6Config REQUIRED)    # → KF6::ConfigGui
find_package(KF6KIO REQUIRED)       # → KF6::KIOCore KF6::KIOGui KF6::KIOFileWidgets
find_package(Plasma REQUIRED)       # → Plasma::Plasma (standalone, not under KF6)

# ── Plasma extras ────────────────────────────────────
find_package(PlasmaActivities REQUIRED)       # → Plasma::Activities
find_package(PlasmaActivitiesStats REQUIRED)   # → Plasma::ActivitiesStats
find_package(KSysGuard REQUIRED)               # → KSysGuard::ProcessCore
find_package(LibNotificationManager REQUIRED)  # → PW::LibNotificationManager

# ── Build ────────────────────────────────────────────
add_subdirectory(taskmanager-skyler)
add_subdirectory(kickoff-skyler)

# ── Install wrapper packages ─────────────────────────
plasma_install_package(icontasks-skyler org.kde.plasma.icontasks.skyler)
plasma_install_package(kickoff-skyler org.kde.plasma.kickoff.skyler)

# ── Compile extra translations ────────────────────────
set(EXTRA_PO_DIR "${CMAKE_SOURCE_DIR}/po")
set(EXTRA_MO_DIR "${CMAKE_BINARY_DIR}/po")
file(GLOB EXTRA_PO_FILES "${EXTRA_PO_DIR}/*/*.po")
set(EXTRA_MO_FILES "")
foreach(PO_FILE ${EXTRA_PO_FILES})
    get_filename_component(LOCALE_NAME "${PO_FILE}" DIRECTORY)
    get_filename_component(LOCALE_NAME "${LOCALE_NAME}" NAME)
    set(MO_FILE "${EXTRA_MO_DIR}/${LOCALE_NAME}/extra.mo")
    add_custom_command(
        OUTPUT ${MO_FILE}
        COMMAND ${CMAKE_COMMAND} -E make_directory "${EXTRA_MO_DIR}/${LOCALE_NAME}"
        COMMAND msgfmt -o ${MO_FILE} ${PO_FILE}
        DEPENDS ${PO_FILE}
        COMMENT "Compiling extra translations (${LOCALE_NAME})"
    )
    list(APPEND EXTRA_MO_FILES ${MO_FILE})
endforeach()
add_custom_target(extra_translations ALL DEPENDS ${EXTRA_MO_FILES})

# ── Install taskmanager translations (merge extra) ────
# msgcat cannot merge binary .mo; convert orig→po, merge, recompile
install(CODE "
    set(ORIG_DOMAIN plasma_applet_org.kde.plasma.taskmanager)
    set(SKYLER_DOMAINS
        plasma_applet_org.kde.plasma.icontasks.skyler
        plasma_applet_org.kde.plasma.taskmanager.skyler
    )
    file(GLOB LOCALE_DIRS /usr/share/locale/*/LC_MESSAGES)
    foreach(LOCALE_DIR \${LOCALE_DIRS})
        get_filename_component(LOC \${LOCALE_DIR} DIRECTORY)
        get_filename_component(LOC \${LOC} NAME)
        set(ORIG_MO \"\${LOCALE_DIR}/\${ORIG_DOMAIN}.mo\")
        set(EXTRA_PO \"${EXTRA_PO_DIR}/\${LOC}/plasma_applet_org.kde.plasma.taskmanager_extra.po\")
        if(EXISTS \"\${ORIG_MO}\")
            foreach(SKYLER_DOMAIN \${SKYLER_DOMAINS})
                if(EXISTS \"\${EXTRA_PO}\")
                    set(TMP_DIR \"\${CMAKE_CURRENT_BINARY_DIR}/tl-tmp\")
                    file(MAKE_DIRECTORY \"\${TMP_DIR}\")
                    execute_process(COMMAND msgunfmt \"\${ORIG_MO}\"
                        OUTPUT_FILE \"\${TMP_DIR}/orig.po\")
                    execute_process(COMMAND msgcat --use-first
                        \"\${TMP_DIR}/orig.po\" \"\${EXTRA_PO}\"
                        OUTPUT_FILE \"\${TMP_DIR}/merged.po\")
                    execute_process(COMMAND msgfmt
                        -o \"\${LOCALE_DIR}/\${SKYLER_DOMAIN}.mo\"
                        \"\${TMP_DIR}/merged.po\")
                    file(REMOVE_RECURSE \"\${TMP_DIR}\")
                else()
                    execute_process(COMMAND \${CMAKE_COMMAND} -E create_symlink
                        \"\${ORIG_DOMAIN}.mo\"
                        \"\${LOCALE_DIR}/\${SKYLER_DOMAIN}.mo\")
                endif()
            endforeach()
        endif()
    endforeach()
")

# ── Reuse original kickoff translations ─────────────
install(CODE "
    set(ORIG_DOMAIN plasma_applet_org.kde.plasma.kickoff)
    set(SKYLER_DOMAIN plasma_applet_org.kde.plasma.kickoff.skyler)
    file(GLOB LOCALE_DIRS /usr/share/locale/*/LC_MESSAGES)
    foreach(LOCALE_DIR \${LOCALE_DIRS})
        set(ORIG_MO \"\${LOCALE_DIR}/\${ORIG_DOMAIN}.mo\")
        if(EXISTS \"\${ORIG_MO}\")
            execute_process(COMMAND \${CMAKE_COMMAND} -E create_symlink
                \"\${ORIG_DOMAIN}.mo\"
                \"\${LOCALE_DIR}/\${SKYLER_DOMAIN}.mo\")
        endif()
    endforeach()
")

# ── Status ───────────────────────────────────────────
message(STATUS "")
message(STATUS "Skyler Plasma Task Manager ${PROJECT_VERSION}")
message(STATUS "  Install prefix: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "  Plugin ID:      org.kde.plasma.icontasks.skyler")
message(STATUS "")
