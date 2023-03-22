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
