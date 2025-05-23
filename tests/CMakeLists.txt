SET(TEST_WITH_REDIS_VERSION "6.2.1" CACHE STRING "Redis version used when running tests.")

# Re-enable `-std=gnu99` for tests only, this avoids the need to sprinkle
# `#define  _XOPEN_SOURCE 600` in test code for e.g. strdup()
set(CMAKE_C_EXTENSIONS ON)

if(ENABLE_SSL)
  # Generate SSL certs and keys when needed
  set(SSL_CONFIGS ca.crt ca.key ca.txt redis.crt redis.key client.crt client.key)
  add_custom_command(
    OUTPUT ${SSL_CONFIGS}
    COMMAND openssl genrsa -out ca.key 4096
    COMMAND openssl req -x509 -new -nodes -sha256 -key ca.key -days 3650 -subj '/CN=Redis Test CA' -out ca.crt
    COMMAND openssl genrsa -out redis.key 2048
    COMMAND openssl req -new -sha256 -key redis.key -subj '/CN=Redis Server Test Cert' | openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAserial ca.txt -CAcreateserial -days 365 -out redis.crt
    COMMAND openssl genrsa -out client.key 2048
    COMMAND openssl req -new -sha256 -key client.key -subj '/CN=Redis Client Test Cert' | openssl x509 -req -sha256 -CA ca.crt -CAkey ca.key -CAserial ca.txt -CAcreateserial -days 365 -out client.crt
    )
  add_custom_target(generate_tls_configs DEPENDS ${SSL_CONFIGS})

  set(SSL_LIBRARY hiredis_cluster_ssl)
endif()

# Targets to setup Redis Clusters for testing
if(ENABLE_IPV6_TESTS)
  set(NO_IPV6 "")
else()
  set(NO_IPV6 "true") # Ignore command
endif()

if(TEST_WITH_VALKEY_VERSION)
  set(CLUSTER_SCRIPT "${CMAKE_SOURCE_DIR}/tests/scripts/valkey-cluster")
  set(CLUSTER_VERSION "VALKEY_VERSION=${TEST_WITH_VALKEY_VERSION}")
else()
  set(CLUSTER_SCRIPT "${CMAKE_SOURCE_DIR}/tests/scripts/redis-cluster")
  set(CLUSTER_VERSION "REDIS_VERSION=${TEST_WITH_REDIS_VERSION}")
endif()

add_custom_target(start
  COMMAND PORT=7000 ${CLUSTER_VERSION} ${CLUSTER_SCRIPT} start
  COMMAND PORT=7100 ${CLUSTER_VERSION} ADDITIONAL_OPTIONS='--requirepass secretword --masterauth secretword' ADDITIONAL_CLI_OPTIONS='-a secretword' ${CLUSTER_SCRIPT} start
  COMMAND ${NO_IPV6} PORT=7200 ${CLUSTER_VERSION} CLUSTER_HOST=::1 ADDITIONAL_OPTIONS='--bind ::1' ADDITIONAL_CLI_OPTIONS='-h ::1' ${CLUSTER_SCRIPT} start
)
add_custom_target(stop
  COMMAND PORT=7000 ${CLUSTER_SCRIPT} stop
  COMMAND PORT=7100 ${CLUSTER_SCRIPT} stop
  COMMAND ${NO_IPV6} PORT=7200 ${CLUSTER_SCRIPT} stop
)

# Find dependencies
find_package(PkgConfig REQUIRED)
pkg_check_modules(GLIB_LIBRARY IMPORTED_TARGET glib-2.0)
find_library(LIBUV_LIBRARY uv HINTS /usr/lib/x86_64-linux-gnu)
find_library(LIBEV_LIBRARY ev HINTS /usr/lib/x86_64-linux-gnu)
find_library(LIBEVENT_LIBRARY event HINTS /usr/lib/x86_64-linux-gnu)
find_path(LIBEVENT_INCLUDES event2/event.h)
include_directories(${LIBEVENT_INCLUDES})

if(MSVC OR MINGW)
  find_library(LIBEVENT_LIBRARY Libevent)
else()
  add_compile_options(-Wall -Wextra -pedantic -Werror)
  # Debug mode for tests
  set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "" FORCE)
endif()

# Make sure ctest gives the output when tests fail
list(APPEND CMAKE_CTEST_ARGUMENTS "--output-on-failure")

add_executable(ct_async ct_async.c)
target_link_libraries(ct_async hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
add_test(NAME ct_async COMMAND "$<TARGET_FILE:ct_async>")
set_tests_properties(ct_async PROPERTIES LABELS "CT")

add_executable(ct_commands ct_commands.c test_utils.c)
target_link_libraries(ct_commands hiredis_cluster ${SSL_LIBRARY})
add_test(NAME ct_commands COMMAND "$<TARGET_FILE:ct_commands>")
set_tests_properties(ct_commands PROPERTIES LABELS "CT")

add_executable(ct_connection ct_connection.c test_utils.c)
target_link_libraries(ct_connection hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
add_test(NAME ct_connection COMMAND "$<TARGET_FILE:ct_connection>")
set_tests_properties(ct_connection PROPERTIES LABELS "CT")

add_executable(ct_pipeline ct_pipeline.c)
target_link_libraries(ct_pipeline hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
add_test(NAME ct_pipeline COMMAND "$<TARGET_FILE:ct_pipeline>")
set_tests_properties(ct_pipeline PROPERTIES LABELS "CT")

add_executable(ct_connection_ipv6 ct_connection_ipv6.c)
target_link_libraries(ct_connection_ipv6 hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
add_test(NAME ct_connection_ipv6 COMMAND "$<TARGET_FILE:ct_connection_ipv6>")
set_tests_properties(ct_connection_ipv6 PROPERTIES LABELS "CT")
if(NOT ENABLE_IPV6_TESTS)
  set_tests_properties(ct_connection_ipv6 PROPERTIES DISABLED True)
endif()

add_executable(ct_out_of_memory_handling ct_out_of_memory_handling.c)
target_link_libraries(ct_out_of_memory_handling hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
add_test(NAME ct_out_of_memory_handling COMMAND "$<TARGET_FILE:ct_out_of_memory_handling>")
set_tests_properties(ct_out_of_memory_handling PROPERTIES LABELS "CT")

add_executable(ct_specific_nodes ct_specific_nodes.c test_utils.c)
target_link_libraries(ct_specific_nodes hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
add_test(NAME ct_specific_nodes COMMAND "$<TARGET_FILE:ct_specific_nodes>")
set_tests_properties(ct_specific_nodes PROPERTIES LABELS "CT")

add_executable(ut_parse_cmd ut_parse_cmd.c test_utils.c)
target_link_libraries(ut_parse_cmd hiredis_cluster ${SSL_LIBRARY})
add_test(NAME ut_parse_cmd COMMAND "$<TARGET_FILE:ut_parse_cmd>")
set_tests_properties(ut_parse_cmd PROPERTIES LABELS "UT")

if(ENABLE_SSL)
  # Executable: tls
  add_executable(example_tls main_tls.c)
  target_link_libraries(example_tls hiredis_cluster ${SSL_LIBRARY})
  add_dependencies(example_tls generate_tls_configs)

  # Executable: async tls
  add_executable(example_async_tls main_async_tls.c)
  target_link_libraries(example_async_tls hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
  add_dependencies(example_async_tls generate_tls_configs)
endif()

if(LIBUV_LIBRARY)
  add_executable(ct_async_libuv ct_async_libuv.c)
  target_link_libraries(ct_async_libuv hiredis_cluster ${SSL_LIBRARY} ${LIBUV_LIBRARY})
  add_test(NAME ct_async_libuv COMMAND "$<TARGET_FILE:ct_async_libuv>")
  set_tests_properties(ct_async_libuv PROPERTIES LABELS "CT")
else()
  add_test(NAME ct_async_libuv COMMAND "")
  set_tests_properties(ct_async_libuv PROPERTIES DISABLED True)
endif()

if(LIBEV_LIBRARY)
  add_executable(ct_async_libev ct_async_libev.c)
  # Temporary remove warning of unused parameter due to an issue in hiredis libev adapter
  target_compile_options(ct_async_libev PRIVATE -Wno-unused-parameter)
  target_link_libraries(ct_async_libev hiredis_cluster ${SSL_LIBRARY} ${LIBEV_LIBRARY})
  add_test(NAME ct_async_libev COMMAND "$<TARGET_FILE:ct_async_libev>")
  set_tests_properties(ct_async_libev PROPERTIES LABELS "CT")
else()
  add_test(NAME ct_async_libev COMMAND "")
  set_tests_properties(ct_async_libev PROPERTIES DISABLED True)
endif()

if(GLIB_LIBRARY_FOUND)
  add_executable(ct_async_glib ct_async_glib.c)
  target_link_libraries(ct_async_glib hiredis_cluster ${SSL_LIBRARY} PkgConfig::GLIB_LIBRARY)
  add_test(NAME ct_async_glib COMMAND "$<TARGET_FILE:ct_async_glib>")
  set_tests_properties(ct_async_glib PROPERTIES LABELS "CT")
else()
  add_test(NAME ct_async_glib COMMAND "")
  set_tests_properties(ct_async_glib PROPERTIES DISABLED True)
endif()

# Tests using simulated redis node
add_executable(clusterclient clusterclient.c)
target_link_libraries(clusterclient hiredis_cluster ${SSL_LIBRARY})
add_executable(clusterclient_async clusterclient_async.c)
target_link_libraries(clusterclient_async hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
add_executable(clusterclient_reconnect_async clusterclient_reconnect_async.c)
target_link_libraries(clusterclient_reconnect_async hiredis_cluster ${SSL_LIBRARY} ${LIBEVENT_LIBRARY})
add_test(NAME set-get-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/set-get-test.sh"
                 "$<TARGET_FILE:clusterclient>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME set-get-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/set-get-test.sh"
         "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME ask-redirect-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/ask-redirect-test.sh"
         "$<TARGET_FILE:clusterclient>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME ask-redirect-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/ask-redirect-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME ask-redirect-using-cluster-nodes-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/ask-redirect-using-cluster-nodes-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME moved-redirect-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/moved-redirect-test.sh"
                 "$<TARGET_FILE:clusterclient>"
                 WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME moved-redirect-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/moved-redirect-test.sh"
         "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME moved-redirect-using-cluster-nodes-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/moved-redirect-using-cluster-nodes-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME dbsize-to-all-nodes-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/dbsize-to-all-nodes-test.sh"
                 "$<TARGET_FILE:clusterclient>"
                 WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME dbsize-to-all-nodes-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/dbsize-to-all-nodes-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
                 WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME dbsize-to-all-nodes-during-scaledown-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/dbsize-to-all-nodes-during-scaledown-test.sh"
                 "$<TARGET_FILE:clusterclient>"
                 WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME dbsize-to-all-nodes-during-scaledown-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/dbsize-to-all-nodes-during-scaledown-test-async.sh"
                 "$<TARGET_FILE:clusterclient_async>"
                 WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME reconnect-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/reconnect-test.sh"
                "$<TARGET_FILE:clusterclient_reconnect_async>"
                WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME timeout-handling-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/timeout-handling-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME connect-error-using-cluster-nodes-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/connect-error-using-cluster-nodes-test.sh"
                 "$<TARGET_FILE:clusterclient>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME command-from-callback-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/command-from-callback-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME ask-redirect-connection-error-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/ask-redirect-connection-error-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME cluster-down-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/cluster-down-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME connection-error-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/connection-error-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME redirect-with-hostname-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/redirect-with-hostname-test.sh"
                 "$<TARGET_FILE:clusterclient>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME redirect-with-ipv6-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/redirect-with-ipv6-test.sh"
                 "$<TARGET_FILE:clusterclient>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME redirect-with-ipv6-async-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/redirect-with-ipv6-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
if(NOT ENABLE_IPV6_TESTS)
  set_tests_properties(redirect-with-ipv6-test PROPERTIES DISABLED True)
  set_tests_properties(redirect-with-ipv6-async-test PROPERTIES DISABLED True)
endif()
# This test can't be run on hiredis v1.1.0 due to hiredis issue #1171.
# Disabling the testcase if hiredis contains the issue or if the version is unknown.
add_test(NAME redirect-with-hostname-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/redirect-with-hostname-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
if(hiredis_VERSION VERSION_EQUAL "1.1.0" OR hiredis_VERSION VERSION_EQUAL "0")
  set_tests_properties(redirect-with-hostname-test-async PROPERTIES DISABLED True)
endif()
add_test(NAME cluster-scale-down-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/cluster-scale-down-test.sh"
                "$<TARGET_FILE:clusterclient>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME slots-not-served-test
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/slots-not-served-test.sh"
                 "$<TARGET_FILE:clusterclient>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME slots-not-served-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/slots-not-served-test-async.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME client-disconnect-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/client-disconnect-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME client-disconnect-without-slotmap-update-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/client-disconnect-without-slotmap-update-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME connect-during-cluster-startup-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/connect-during-cluster-startup-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
add_test(NAME connect-during-cluster-startup-using-cluster-nodes-test-async
         COMMAND "${CMAKE_SOURCE_DIR}/tests/scripts/connect-during-cluster-startup-using-cluster-nodes-test.sh"
                 "$<TARGET_FILE:clusterclient_async>"
         WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/tests/scripts/")
