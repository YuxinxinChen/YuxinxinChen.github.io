### Change g++ symbolic link without sudo
If you don't have sudo access and your g++ compiler is not named with a version number, you can create a symbolic link in a directory where you have write access and update your PATH environment variable to include that directory. Here's how to do it:

Choose a directory where you have write access, e.g., $HOME/bin. If the directory doesn't exist, create it:
```
mkdir -p $HOME/bin
```
Create a symbolic link to the installed g++ compiler with the expected name in the chosen directory:
```
ln -s $(which g++) $HOME/bin/g++-9
```
This command creates a symbolic link named g++-9 in the $HOME/bin directory, pointing to the installed g++ compiler.

Update your PATH environment variable to include the chosen directory:
```
export PATH=$HOME/bin:$PATH
```
Add this line to your shell startup script (e.g., ~/.bashrc, ~/.bash_profile, or ~/.profile) to make the change permanent.

Now, when you run nvcc, it should find the g++-9 compiler in your $HOME/bin directory and use it for compilation. Make sure to recompile your project after making these changes.

### CMake check if POWER9

```
cmake_minimum_required(VERSION 3.0)

project(ConditionalMPI)

# Check if the system is IBM POWER9
message("CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(powerpc|ppc)64")
  message("Detected PowerPC 64-bit architecture")
  if(CMAKE_SYSTEM_PROCESSOR STREQUAL "ppc64le")
    message("Detected little-endian PowerPC 64-bit architecture (POWER8 or later)")
    execute_process(
      COMMAND grep -q "POWER9" /proc/cpuinfo
      RESULT_VARIABLE IS_POWER9
    )
    if(IS_POWER9 EQUAL 0)
      message("Detected IBM POWER9 architecture")
      set(IBM_POWER9_FOUND TRUE)
    endif()
  endif()
endif()

# Find MPI
find_package(MPI REQUIRED)

if(IBM_POWER9_FOUND)
  # Use libmpi_ibm.so for POWER9
  set(MPI_LIB_NAME "mpi_ibm")
else()
  # Use libmpi.so for other architectures
  set(MPI_LIB_NAME "mpi")
endif()

# Locate the appropriate MPI library
find_library(MPI_LIBRARY
  NAMES ${MPI_LIB_NAME}
  HINTS ${MPI_LIB_DIR}
  REQUIRED
)

message("Using MPI library: ${MPI_LIBRARY}")

# Create executable
add_executable(my_executable main.cpp)
target_link_libraries(my_executable PRIVATE ${MPI_LIBRARY})

```
