# Drivers independent of any windowing system. Shared by the Linux port and
# the Windows port; the Linux target links only this subset.

set(WB_PORTABLE_DRIVERS
    zebronics.cpp
    glorious.cpp
    kysona.cpp
    keychron.cpp
    logitech.cpp
    razer.cpp
    steelseries.cpp
    corsair.cpp
    hyperx.cpp
    playstation.cpp
    coolermaster.cpp
    finalmouse.cpp
    lamzu.cpp
    aula.cpp
    ninjutso.cpp
    rapoo.cpp)

list(TRANSFORM WB_PORTABLE_DRIVERS PREPEND "${CMAKE_CURRENT_SOURCE_DIR}/")

add_library(wb_drivers STATIC ${WB_PORTABLE_DRIVERS})
target_link_libraries(wb_drivers PUBLIC wb_headers)
target_compile_options(wb_drivers PRIVATE
    -Wall -Wno-missing-field-initializers)
