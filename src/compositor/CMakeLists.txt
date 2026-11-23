# This Source Code Form is subject to the terms of the Mozilla Public
# License, v. 2.0. If a copy of the MPL was not distributed with this
# file, You can obtain one at https://mozilla.org/MPL/2.0/.

# The Compositor is an INTERFACE target for now. It becomes a compiled
# library exposing the C ABI once implementation starts.
add_library(orchidee_compositor INTERFACE)
add_library(orchidee::compositor ALIAS orchidee_compositor)

target_include_directories(
  orchidee_compositor
  INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>)
