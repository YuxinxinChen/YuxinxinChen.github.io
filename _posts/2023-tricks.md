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

### Pass GPU pointer between C++ extensions

To return a GPU memory pointer from a C++ extension and pass it to another C++ extension function in PyTorch, you can follow these steps:

1. Create a C++ extension function that returns a GPU memory pointer:
```
#include <torch/extension.h>

uintptr_t get_gpu_pointer(torch::Tensor tensor) {
    return reinterpret_cast<uintptr_t>(tensor.data_ptr());
}

PYBIND11_MODULE(TORCH_EXTENSION_NAME, m) {
    m.def("get_gpu_pointer", &get_gpu_pointer, "Get GPU memory pointer");
}

```

2. Create another C++ extension function that accepts a GPU memory pointer and creates a tensor from it:
```
#include <torch/extension.h>

torch::Tensor create_from_gpu_pointer(uintptr_t ptr, torch::IntArrayRef sizes, torch::ScalarType dtype, int device_id) {
    auto* data_ptr = reinterpret_cast<void*>(ptr);
    auto options = torch::TensorOptions().dtype(dtype).device(torch::kCUDA, device_id);
    return torch::from_blob(data_ptr, sizes, options);
}

PYBIND11_MODULE(TORCH_EXTENSION_NAME, m) {
    m.def("create_from_gpu_pointer", &create_from_gpu_pointer, "Create tensor from GPU memory pointer");
}
```
3. In your Python code, import both extension modules and use the functions:
```
import torch
import get_pointer_ext
import create_tensor_ext

# Create a tensor on GPU
tensor = torch.randn(2, 3).cuda()

# Get the GPU memory pointer
ptr = get_pointer_ext.get_gpu_pointer(tensor)

# Pass the GPU memory pointer to another extension function
device_id = tensor.device.index
new_tensor = create_tensor_ext.create_from_gpu_pointer(ptr, tensor.shape, tensor.dtype, device_id)

# Verify that the new tensor is the same as the original tensor
print(torch.allclose(tensor, new_tensor))

```
