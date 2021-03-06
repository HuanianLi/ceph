set(MGR_DASHBOARD_VIRTUALENV ${CEPH_BUILD_VIRTUALENV}/mgr-dashboard-virtualenv)

add_custom_target(mgr-dashboard-test-venv
  COMMAND
  ${CMAKE_SOURCE_DIR}/src/tools/setup-virtualenv.sh --python=${MGR_PYTHON_EXECUTABLE} ${MGR_DASHBOARD_VIRTUALENV}
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/src/pybind/mgr/dashboard
  COMMENT "dashboard tests virtualenv is being created")
add_dependencies(tests mgr-dashboard-test-venv)

include(CMakeParseArguments)
function(add_npm_command)
  set(options NODEENV)
  set(single_kw OUTPUT COMMENT WORKING_DIRECTORY)
  set(multi_kw COMMAND DEPENDS)
  cmake_parse_arguments(NC "${options}" "${single_kw}" "${multi_kw}" ${ARGN})
  string(REPLACE ";" " " command "${NC_COMMAND}")
  if(NC_NODEENV)
    string(REGEX REPLACE "^(npm .*)$" ". ${mgr-dashboard-nodeenv}/bin/activate && \\1 && deactivate" command ${command})
  endif()
  string(REPLACE " " ";" command "${command}")
  add_custom_command(
    OUTPUT "${NC_OUTPUT}"
    COMMAND ${command}
    DEPENDS ${NC_DEPENDS}
    WORKING_DIRECTORY "${NC_WORKING_DIRECTORY}"
    COMMENT ${NC_COMMENT})
endfunction(add_npm_command)

if(WITH_MGR_DASHBOARD_FRONTEND AND NOT CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|AARCH64|arm|ARM")

set(mgr-dashboard-nodeenv ${CMAKE_CURRENT_BINARY_DIR}/node-env)

add_custom_command(
  OUTPUT "${mgr-dashboard-nodeenv}/bin/npm"
  COMMAND ${CMAKE_SOURCE_DIR}/src/tools/setup-virtualenv.sh --python=${MGR_PYTHON_EXECUTABLE} ${mgr-dashboard-nodeenv}
  COMMAND ${mgr-dashboard-nodeenv}/bin/pip install nodeenv
  COMMAND ${mgr-dashboard-nodeenv}/bin/nodeenv -p -n 8.11.3
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  COMMENT "dashboard nodeenv is being installed"
)
add_custom_target(mgr-dashboard-nodeenv
  DEPENDS ${mgr-dashboard-nodeenv}/bin/npm
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
)

add_npm_command(
  OUTPUT "${CMAKE_SOURCE_DIR}/src/pybind/mgr/dashboard/frontend/node_modules"
  COMMAND npm install
  DEPENDS frontend/package.json
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/src/pybind/mgr/dashboard/frontend
  COMMENT "dashboard frontend dependencies are being installed"
  NODEENV
)

add_custom_target(mgr-dashboard-frontend-deps
  DEPENDS frontend/node_modules mgr-dashboard-nodeenv
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/src/pybind/mgr/dashboard/frontend
)

# Glob some frontend files. With CMake 3.6, this can be simplified
# to *.ts *.html. Just add:
# list(FILTER frontend_src INCLUDE REGEX "frontend/src")
file(
  GLOB_RECURSE frontend_src
  frontend/src/*.ts
  frontend/src/*.html
  frontend/src/*/*.ts
  frontend/src/*/*.html
  frontend/src/*/*/*.ts
  frontend/src/*/*/*.html
  frontend/src/*/*/*/*.ts
  frontend/src/*/*/*/*.html
  frontend/src/*/*/*/*/*.ts
  frontend/src/*/*/*/*/*.html
  frontend/src/*/*/*/*/*/*.ts
  frontend/src/*/*/*/*/*/*.html)

if(NOT CMAKE_BUILD_TYPE STREQUAL Debug)
  set(npm_command npm run build -- --prod --progress=false)
else()
  set(npm_command npm run build -- --progress=false)
endif()

add_npm_command(
  OUTPUT "${CMAKE_SOURCE_DIR}/src/pybind/mgr/dashboard/frontend/dist"
  COMMAND ${npm_command}
  DEPENDS ${frontend_src} frontend/node_modules
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/src/pybind/mgr/dashboard/frontend
  COMMENT "dashboard frontend is being created"
  NODEENV
)
add_custom_target(mgr-dashboard-frontend-build
  ALL
  DEPENDS frontend/dist mgr-dashboard-frontend-deps
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/src/pybind/mgr/dashboard/frontend)
add_dependencies(tests mgr-dashboard-frontend-build)
endif(WITH_MGR_DASHBOARD_FRONTEND AND NOT CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|AARCH64|arm|ARM")
