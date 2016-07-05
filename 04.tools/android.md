#有用的命令 eventsetup

USER-NAME@MACHINE-NAME:~/Android$ .  ./build/envsetup.sh

      注意，这是一个source命令，执行之后，就会有一些额外的命令可以使用：

      - croot: Changes directory to the top of the tree.
      - m: Makes from the top of the tree.
      - mm: Builds all of the modules in the current directory.
      - mmm: Builds all of the modules in the supplied directories.
      - cgrep: Greps on all local C/C++ files.
      - jgrep: Greps on all local Java files.
      - resgrep: Greps on all local res/*.xml files.
      - godir: Go to the directory containing a file.

##hierarchyviewer


我们通过hierarchyviewer这个工具来查看一下系统启动后的布局情况
(注：hierarchyviewer在SDK/tools目录下，在windows环境下直接运行hierarchyviewer.bat，linux环境下终端执行./hierarchyviewer；

##编译环境

sudo apt-get install zlib1g-dev
sudo apt-get install flex
sudo apt-get install bison
sudo apt-get install gperf
sudo apt-get install libsdl-dev
sudo apt-get install libesd0-dev
sudo apt-get install libncurses5-dev
sudo apt-get install libx11-dev


sudo apt-get install git-core gnupg flex bison gperf build-essential \
  zip curl libc6-dev libncurses5-dev:i386 x11proto-core-dev \
  libx11-dev:i386 libreadline6-dev:i386 libgl1-mesa-glx:i386 \
  libgl1-mesa-dev g++-multilib mingw32 tofrodos \
  python-markdown libxml2-utils xsltproc zlib1g-dev:i386

sudo ln -s /usr/lib/i386-linux-gnu/mesa/libGL.so.1 /usr/lib/i386-linux-gnu/libGL.so
