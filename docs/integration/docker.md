# Docker Instruction
## Preparation
Install the docker application on the server to be deployed
## Import the docker image on the server
Cloud disk download link: (7z self-extracting file)

[Aliyun disk sharing](https://www.aliyundrive.com/s/1abKfjYKWJ6)

Docker image MD5SUM: 3ebafca4bfaa53c93abaf6ab3a3a8626

After entering the server, execute the following command, the result is as follows
```
docker load --input <server image path (blue part)>
docker images # After execution, you can see image_id, the size is 4.53GB
pwd #After execution, get <absolute path of the folder on the host>
docker run -it -v <absolute path of the folder on the host>:</home/your_name (try to fill in the name for easy search)> image_id /bin/bash
```
![load_docker_image.jpeg](../figs/docker_images/load_docker_image.jpeg)
## Enter the container and complete the user switch
```
addgroup --gid 5 your_name
adduser --uid 4 --gid 5 your_name
su your_name #Switch to the user, and the environment deployment is now complete
```
![user_switch_1.jpeg](../figs/docker_images/user_switch_1.jpeg)
![user_switch_2.jpeg](../figs/docker_images/user_switch_2.jpeg)
## Add environment variables
```
vim ~/.bashrc
export RISCV="/opt/riscv"
export PATH=$RISCV/bin:$PATH
export CROSS_COMPILE=riscv64-unknown-linux-gnu-
export ARCH=riscv
source ~/.bashrc
```
![set_env.jpeg](../figs/docker_images/set_env.jpeg)
## Get the source code and compile
### Get the source code and cp to the server
```
git clone git@github.com:OpenXiangShan/ns-bbl.git -b nanhu-dualcore-fpga
cd ns-bbl
rm -r riscv-linux riscv-pk riscv-rootfs
git clone git@github.com:openxiangshan/riscv-linux.git -b linux-5.10.167
git clone git@github.com:openxiangshan/riscv-pk.git -b nanhu-mini-sys
git clone git@github.com:openxiangshan/riscv-rootfs.git -b nanhu-nfs
mkdir build
cd ../
scp -r ns-bbl/ your_name@<server IP>:<absolute path of the folder on the host>
```
### Customize app to rootfs.cpio (optional)

Preparation, cross-compile the customized app in docker first and generate an executable file

```
Example: add i2cdelect command to rootfs.cpio
cpio -imdv < rootfs.cpio //Unzip the original rootfs.cpio
chmod 777 i2cdelect | cp i2cdelect /usr/bin/ //The executable file needs to be given 777 permissions
sudo chown -R root:root .
find . | cpio -o -H newc > ../rootfs.cpio //Package the new rootfs.cpio
cp ../rootfs.cpio ns-bbl //copy and use
```
### compile
```
cd riscv-linux
make nanhu_fpga_defconfig
cd ../
make sw -j200 #After completion, there is linux.bin in the buid directory
```
![compile.jpeg](../figs/docker_images/compile.jpeg)
### Generate a txt file for burning FPGA
Use the following tools in the Linux environment to generate a txt file for burning FPGA from linux.bin

[bin2fpgadata.tar.gz](https://raw.githubusercontent.com/OpenXiangShan/XiangShan-doc/main/docs/integration/resources/bin2fpgadata.tar.gz)
```
#Generate data.txt
<bin2fpgadata path>/bin2fpgadata -i linux.bin
```
## Test
Follow the operation demonstrated in [Building an open source minimum system for FPGA based on Nanhu](https://xiangshan-doc.readthedocs.io/zh_CN/latest/integration/fpga/#_1), put the generated txt file into the minimum system and you can test it
