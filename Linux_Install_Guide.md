# Comprehensive Setup Guide for HopeFOAM-0.1 on Ubuntu 22.04

This guide provides step-by-step instructions for setting up **HopeFOAM-0.1** on Ubuntu 22.04. Each section includes detailed commands and explanations to ensure ease of understanding and successful configuration.

---

## 1. Start the Docker Container

Run the following command to create and start a Docker container for HopeFOAM:

```bash
docker run -d --restart=on-failure --name hopefoam-desktop --cap-add=SYS_PTRACE --gpus all \
    -e USER=ubuntu -e PASSWORD=ubuntu -e GID=1001 -e UID=1001 \
    --shm-size=4096m -p 10022:22 -p 14000:4000 alundilong/ubuntu-desktop
```

### Key Points:
- **Container Name**: `hopefoam-desktop`
- **Host Ports**: 10022 for SSH, 14000–4000 for NoMachine.
- **User**: `ubuntu`
- **Password**: `ubuntu`

---

## 2. Access the NoMachine Desktop

To connect to the container’s GUI via NoMachine:
- **Host**: `127.0.0.1`
- **User/Password**: `ubuntu`

---

## 3. Setup GCC/G++ 4.8.5

HopeFOAM requires GCC 4.8.5. Use the following steps to install and configure it:

### Download and Install GCC 4.8.5:
```bash
wget http://mirrors.kernel.org/ubuntu/pool/universe/g/gcc-4.8/g++-4.8_4.8.5-4ubuntu8_amd64.deb
wget http://mirrors.kernel.org/ubuntu/pool/universe/g/gcc-4.8/libstdc++-4.8-dev_4.8.5-4ubuntu8_amd64.deb
wget http://mirrors.kernel.org/ubuntu/pool/universe/g/gcc-4.8/gcc-4.8-base_4.8.5-4ubuntu8_amd64.deb
wget http://mirrors.kernel.org/ubuntu/pool/universe/g/gcc-4.8/gcc-4.8_4.8.5-4ubuntu8_amd64.deb
wget http://mirrors.kernel.org/ubuntu/pool/universe/g/gcc-4.8/libgcc-4.8-dev_4.8.5-4ubuntu8_amd64.deb
wget http://mirrors.kernel.org/ubuntu/pool/universe/g/gcc-4.8/cpp-4.8_4.8.5-4ubuntu8_amd64.deb
wget http://mirrors.kernel.org/ubuntu/pool/universe/g/gcc-4.8/libasan0_4.8.5-4ubuntu8_amd64.deb

sudo apt install ./gcc-4.8_4.8.5-4ubuntu8_amd64.deb ./gcc-4.8-base_4.8.5-4ubuntu8_amd64.deb \
    ./libstdc++-4.8-dev_4.8.5-4ubuntu8_amd64.deb ./cpp-4.8_4.8.5-4ubuntu8_amd64.deb \
    ./libgcc-4.8-dev_4.8.5-4ubuntu8_amd64.deb ./libasan0_4.8.5-4ubuntu8_amd64.deb \
    ./g++-4.8_4.8.5-4ubuntu8_amd64.deb
```

### Configure GCC/G++ Defaults:
```bash
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.8 50
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 50
```

---

## 4. Install PETSc and SLEPc

### Install PETSc (Version 3.7.6):
#### **Downgrade OpenMPI to 1.8.5**
This workaround ensures compatibility with PETSc 3.7.6. During PETSc configuration, OpenMPI will be installed.

```bash
python2 configure --force --with-debugging=0 --prefix=/home/ubuntu/software/petsc376 \
    --download-openmpi=yes --download-hypre=yes
```

#### **Fix HYPRE Download Issue**:
Modify `config/BuildSystem/config/packages/hypre.py` to correct the HYPRE download URL:
```python
self.download = ['git://https://github.com/hypre-space/hypre', 
                 'https://github.com/hypre-space/hypre/archive/'+self.gitcommit+'.tar.gz']
```

---

### Install SLEPc (Version 3.7.4):
```bash
python2 configure --prefix=/home/ubuntu/software/slepc374/
```

### Update Environment Variables:
```bash
export PATH=/home/ubuntu/software/petsc376/bin:$PATH
export LD_LIBRARY_PATH=/home/ubuntu/software/petsc376/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=/home/ubuntu/software/slepc374/lib:$LD_LIBRARY_PATH
export CPATH=/home/ubuntu/software/petsc376/include:$CPATH
```

---

## 5. Set Up HopeFOAM Environment

### Install Dependencies:
```bash
sudo apt install libmpfr-dev flex
```

### Install Boost and CGAL:
Download and extract **Boost 1.55.0** and **CGAL 4.8** into `HopeFOAM/ThirdParty-0.1`. Update the configurations:

```bash
cd $HOME/HopeFOAM
sed -i -e 's/\(boost_version=\)boost-system/\1boost-1.55.0/' HopeFOAM-0.1/etc/config.sh/CGAL
sed -i -e 's/\(cgal_version=\)cgal-system/\1CGAL-4.8/' HopeFOAM-0.1/etc/config.sh/CGAL
sed -i -e 's=\-lmpfr=-lmpfr -lboost_thread=' HopeFOAM-0.1/wmake/rules/General/CGAL
cd $HOME/HopeFOAM/ThirdParty-0.1
./Allwmake
```

**Note**: Comment out this line in `Allwmake`:
```bash
rm util/machines/machines.*
```

---

## 6. Compile OpenFOAM

### Configure and Compile OpenFOAM:
```bash
cd $HOME/HopeFOAM-0.1
vim etc/bashrc
# Change WM_MPLIB to SYSTEMOPENMPI
source etc/bashrc
./AllwmakeOpenFOAM
```

---

## 7. Compile the DG Module

### Fix `dgMesh.C` Bug:
Edit `dgMesh.C`:
```cpp
PetscBool ifInit;
PetscInitialized(&ifInit);
if (!ifInit) {
    int argc = 1; // was 2
    char **argv = new char*[1]; // was char **argv = new char*[3];
    argv[0] = new char[20]; // was argv[1] = new char[20];
    sprintf(argv[0], "-no_signal_handler"); // was sprintf(argv[1], "-no_signal_handler");
    PetscInitialize(&argc, &argv, (char *)0, "");
}
```

### Compile DG:
```bash
./AllwmakeDG
```

---

## 8. Run the Cylinder Tutorial

Navigate to the tutorial directory and run the case:

```bash
cd HopeFOAM-0.1/tutorials/DG/2D/cylinder/dgChorinFoam/
wmake
cd ..
./dgChorinFoamCylinder
dgToVTK
```

If you see the simulation output, the setup is complete!

---

## Troubleshooting Tips:
1. **Environment Variables**: Ensure all paths are correctly set using `echo $PATH` and `echo $LD_LIBRARY_PATH`.
2. **Dependencies**: Install missing libraries with `sudo apt install`.
3. **Check Logs**: Review logs for compilation issues.

This concludes the setup process for HopeFOAM-0.1. Enjoy using it for your simulations!