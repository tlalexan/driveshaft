set(
    driveshaft_lib_sources
    ./driveshaft-config.cpp
    ./gearman-client.cpp
    ./main-loop.cpp
    ./thread-loop.cpp
    ./thread-registry.cpp
    ./pidfile.cpp
    ./dist/jsoncpp.cpp
    metric-proxy.cpp metric-proxy.h)

add_library(driveshaft ${driveshaft_lib_sources})
add_executable(driveshaft-bin ./main.cpp)
set_target_properties(driveshaft-bin PROPERTIES OUTPUT_NAME driveshaft)
target_link_libraries(driveshaft-bin driveshaft ${DRIVESHAFT_LINK_LIBRARIES})

install(TARGETS driveshaft-bin DESTINATION bin)