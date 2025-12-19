cmake_minimum_required(VERSION 3.28...3.30)

set(obs-websocket_VERSION 5.6.3)
set(OBS_WEBSOCKET_RPC_VERSION 1)

include(cmake/obs-websocket-api.cmake)

# Headless mode option - skips Qt UI components for server-only usage
option(OBS_WEBSOCKET_HEADLESS "Build obs-websocket for headless mode (no Qt UI)" OFF)

if(OBS_WEBSOCKET_HEADLESS)
  message(STATUS "[obs-websocket] Building in HEADLESS mode - Qt UI components disabled")
endif()

option(ENABLE_WEBSOCKET "Enable building OBS with websocket plugin" ON)
if(NOT ENABLE_WEBSOCKET)
  target_disable(obs-websocket)
  return()
endif()

# Find Qt - only need Core, Network, and Gui (for QImage) for headless mode
if(OBS_WEBSOCKET_HEADLESS)
  find_package(Qt6 REQUIRED Core Network Gui)
else()
  find_package(Qt6 REQUIRED Core Widgets Svg Network)
endif()

# Find nlohmann JSON
find_package(nlohmann_json 3.11 REQUIRED)

# Find qrcodegencpp - only needed for UI mode
if(NOT OBS_WEBSOCKET_HEADLESS)
  set(CMAKE_FIND_PACKAGE_PREFER_CONFIG ON)
  find_package(qrcodegencpp REQUIRED)
  set(CMAKE_FIND_PACKAGE_PREFER_CONFIG OFF)
endif()

# Find WebSocket++
find_package(Websocketpp 0.8 REQUIRED)

# Find Asio
find_package(Asio 1.12.1 REQUIRED)

add_library(obs-websocket MODULE)
add_library(OBS::websocket ALIAS obs-websocket)

# Core sources - always included
target_sources(
  obs-websocket
  PRIVATE # cmake-format: sortable
          src/Config.cpp
          src/Config.h
          src/obs-websocket.cpp
          src/obs-websocket.h
          src/WebSocketApi.cpp
          src/WebSocketApi.h)

# UI sources - only included if not in headless mode
if(NOT OBS_WEBSOCKET_HEADLESS)
  target_sources(
    obs-websocket
    PRIVATE # cmake-format: sortable
            src/forms/ConnectInfo.cpp
            src/forms/ConnectInfo.h
            src/forms/resources.qrc
            src/forms/SettingsDialog.cpp
            src/forms/SettingsDialog.h)
endif()

target_sources(
  obs-websocket
  PRIVATE # cmake-format: sortable
          src/websocketserver/rpc/WebSocketSession.h
          src/websocketserver/types/WebSocketCloseCode.h
          src/websocketserver/types/WebSocketOpCode.h
          src/websocketserver/WebSocketServer.cpp
          src/websocketserver/WebSocketServer.h
          src/websocketserver/WebSocketServer_Protocol.cpp)

target_sources(
  obs-websocket
  PRIVATE # cmake-format: sortable
          src/eventhandler/EventHandler.cpp
          src/eventhandler/EventHandler.h
          src/eventhandler/EventHandler_Config.cpp
          src/eventhandler/EventHandler_Filters.cpp
          src/eventhandler/EventHandler_General.cpp
          src/eventhandler/EventHandler_Inputs.cpp
          src/eventhandler/EventHandler_MediaInputs.cpp
          src/eventhandler/EventHandler_Outputs.cpp
          src/eventhandler/EventHandler_SceneItems.cpp
          src/eventhandler/EventHandler_Scenes.cpp
          src/eventhandler/EventHandler_Transitions.cpp
          src/eventhandler/EventHandler_Ui.cpp
          src/eventhandler/types/EventSubscription.h)

target_sources(
  obs-websocket
  PRIVATE # cmake-format: sortable
          src/requesthandler/RequestBatchHandler.cpp
          src/requesthandler/RequestBatchHandler.h
          src/requesthandler/RequestHandler.cpp
          src/requesthandler/RequestHandler.h
          src/requesthandler/RequestHandler_Config.cpp
          src/requesthandler/RequestHandler_Filters.cpp
          src/requesthandler/RequestHandler_General.cpp
          src/requesthandler/RequestHandler_Inputs.cpp
          src/requesthandler/RequestHandler_MediaInputs.cpp
          src/requesthandler/RequestHandler_Outputs.cpp
          src/requesthandler/RequestHandler_Record.cpp
          src/requesthandler/RequestHandler_SceneItems.cpp
          src/requesthandler/RequestHandler_Scenes.cpp
          src/requesthandler/RequestHandler_Sources.cpp
          src/requesthandler/RequestHandler_Stream.cpp
          src/requesthandler/RequestHandler_Transitions.cpp
          src/requesthandler/RequestHandler_Ui.cpp
          src/requesthandler/rpc/Request.cpp
          src/requesthandler/rpc/Request.h
          src/requesthandler/rpc/RequestBatchRequest.cpp
          src/requesthandler/rpc/RequestBatchRequest.h
          src/requesthandler/rpc/RequestResult.cpp
          src/requesthandler/rpc/RequestResult.h
          src/requesthandler/types/RequestBatchExecutionType.h
          src/requesthandler/types/RequestStatus.h)

target_sources(
  obs-websocket
  PRIVATE # cmake-format: sortable
          src/utils/Compat.cpp
          src/utils/Compat.h
          src/utils/Crypto.cpp
          src/utils/Crypto.h
          src/utils/Json.cpp
          src/utils/Json.h
          src/utils/Obs.cpp
          src/utils/Obs.h
          src/utils/Obs_ActionHelper.cpp
          src/utils/Obs_ArrayHelper.cpp
          src/utils/Obs_NumberHelper.cpp
          src/utils/Obs_ObjectHelper.cpp
          src/utils/Obs_SearchHelper.cpp
          src/utils/Obs_StringHelper.cpp
          src/utils/Obs_VolumeMeter.cpp
          src/utils/Obs_VolumeMeter.h
          src/utils/Obs_VolumeMeter_Helpers.h
          src/utils/Platform.cpp
          src/utils/Platform.h
          src/utils/Utils.h)

configure_file(src/plugin-macros.h.in plugin-macros.generated.h)
target_sources(obs-websocket PRIVATE plugin-macros.generated.h)

target_compile_definitions(
  obs-websocket PRIVATE ASIO_STANDALONE $<$<BOOL:${PLUGIN_TESTS}>:PLUGIN_TESTS>
                        $<$<BOOL:${OBS_WEBSOCKET_HEADLESS}>:OBS_WEBSOCKET_HEADLESS>
                        $<$<PLATFORM_ID:Windows>:_WEBSOCKETPP_CPP11_STL_> $<$<PLATFORM_ID:Windows>:_WIN32_WINNT=0x0603>)

target_compile_options(
  obs-websocket
  PRIVATE $<$<PLATFORM_ID:Windows>:/wd4267>
          $<$<PLATFORM_ID:Windows>:/wd4996>
          $<$<COMPILE_LANG_AND_ID:CXX,GNU,AppleClang,Clang>:-Wall>
          $<$<COMPILE_LANG_AND_ID:CXX,GNU,AppleClang,Clang>:-Wno-error=float-conversion>
          $<$<COMPILE_LANG_AND_ID:CXX,GNU,AppleClang,Clang>:-Wno-error=shadow>
          $<$<COMPILE_LANG_AND_ID:CXX,GNU>:-Wno-error=format-overflow>
          $<$<COMPILE_LANG_AND_ID:CXX,GNU>:-Wno-error=int-conversion>
          $<$<COMPILE_LANG_AND_ID:CXX,GNU>:-Wno-error=comment>
          $<$<COMPILE_LANG_AND_ID:CXX,GNU>:-Wno-error=deprecated-declarations>
          $<$<COMPILE_LANG_AND_ID:CXX,AppleClang,Clang>:-Wno-error=null-pointer-subtraction>
          $<$<COMPILE_LANG_AND_ID:CXX,AppleClang,Clang>:-Wno-error=deprecated-declarations>
          $<$<COMPILE_LANG_AND_ID:CXX,AppleClang,Clang>:-Wno-error=implicit-int-conversion>
          $<$<COMPILE_LANG_AND_ID:CXX,AppleClang,Clang>:-Wno-error=shorten-64-to-32>
          $<$<COMPILE_LANG_AND_ID:CXX,AppleClang,Clang>:-Wno-comma>
          $<$<COMPILE_LANG_AND_ID:CXX,AppleClang,Clang>:-Wno-quoted-include-in-framework-header>)

# Core libraries - always linked
target_link_libraries(
  obs-websocket
  PRIVATE OBS::libobs
          OBS::frontend-api
          OBS::websocket-api
          Qt::Core
          Qt::Network
          Qt::Gui
          nlohmann_json::nlohmann_json
          Websocketpp::Websocketpp
          Asio::Asio)

# UI libraries - only linked if not in headless mode
if(NOT OBS_WEBSOCKET_HEADLESS)
  target_link_libraries(
    obs-websocket
    PRIVATE Qt::Widgets
            Qt::Svg
            qrcodegencpp::qrcodegencpp)
endif()

target_link_options(obs-websocket PRIVATE $<$<PLATFORM_ID:Windows>:/IGNORE:4099>)

# Configure target properties
if(OBS_WEBSOCKET_HEADLESS)
  # Headless mode - still need AUTOMOC for WebSocketServer signals, but no UIC/RCC
  set_target_properties_obs(
    obs-websocket
    PROPERTIES FOLDER plugins
               PREFIX ""
               AUTOMOC ON
               AUTOUIC OFF
               AUTORCC OFF)
else()
  # Full UI mode - enable Qt automation
  set_target_properties_obs(
    obs-websocket
    PROPERTIES FOLDER plugins
               PREFIX ""
               AUTOMOC ON
               AUTOUIC ON
               AUTORCC ON)
endif()

if(OS_WINDOWS)
  set_property(
    TARGET obs-websocket
    APPEND
    PROPERTY AUTORCC_OPTIONS --format-version 1)
endif()
