# Getting started (Linux)

Prereqs:
- sudo apt install build-essential cmake git doxygen graphviz python3-pip
- pip3 install sphinx breathe sphinx_rtd_theme

Build:
mkdir -p Build && cd Build
cmake ..
make -j$(nproc)

Run:
./Build/overgrowth   # adjust executable name

Run unit tests:
ctest --output-on-failure
