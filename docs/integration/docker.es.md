# Instrucción de Docker
## Preparar
Instalar la aplicación Docker en el servidor que se va a implementar
## El servidor importa la imagen de Docker
Enlace de descarga del disco en la nube: (archivo autoextraíble 7z)

[Compartir en Aliyun Drive](https://www.aliyundrive.com/s/1abKfjYKWJ6)

Imagen Docker MD5SUM: 3ebafca4bfaa53c93abaf6ab3a3a8626

Luego de ingresar al servidor, ejecute el siguiente comando, los resultados son los siguientes
```
docker load --input <ruta de la imagen del servidor (parte azul)>
imágenes de docker # Después de la ejecución, puede ver el image_id, el tamaño es 4,53 GB
pwd #Después de la ejecución, obtenga <ruta absoluta de la carpeta en la máquina host>
docker run -it -v <ruta absoluta de la carpeta en el host>:</home/your_name (intenta completar el nombre para facilitar la búsqueda)> image_id /bin/bash
```
![cargar_imagen_docker.jpeg](../figs/docker_images/cargar_imagen_docker.jpeg)
## Ingrese al contenedor y complete el cambio de usuario
```
addgroup --gid 5 tu_nombre
adduser --uid 4 --gid 5 tu_nombre
su your_name #Cambiar a usuario, la implementación del entorno ahora está completa
```
![usuario_switch_1.jpeg](../figs/docker_images/usuario_switch_1.jpeg)
![usuario_switch_2.jpeg](../figs/docker_images/usuario_switch_2.jpeg)
## Agregar variables de entorno
```
vim ~/.bashrc
exportar RISCV="/opt/riscv"
exportar RUTA=$RISCV/bin:$RUTA
exportar CROSS_COMPILE=riscv64-desconocido-linux-gnu-
exportar ARCH=riscv
fuente ~/.bashrc
```
![establecer_env.jpeg](../figs/docker_images/establecer_env.jpeg)
## Obtener el código fuente y compilar
### Obtenga el código fuente y cp al servidor
```
clon git git@github.com:OpenXiangShan/ns-bbl.git -b nanhu-dualcore-fpga
CD NS-BBL
rm -r riscv-linux riscv-pk riscv-rootfs
clon git git@github.com:openxiangshan/riscv-linux.git -b linux-5.10.167
clon git git@github.com:openxiangshan/riscv-pk.git -b nanhu-mini-sys
clon git git@github.com:openxiangshan/riscv-rootfs.git -b nanhu-nfs
compilación de mkdir
cd ../
scp -r ns-bbl/ your_name@<IP del servidor>:<ruta absoluta de la carpeta en el host>
```
### Personaliza la aplicación en rootfs.cpio (opcional)

Prepárese para compilar de forma cruzada la aplicación personalizada en Docker y generar un archivo ejecutable

```
Ejemplo: agregue el comando i2cdelect a rootfs.cpio
cpio -imdv < rootfs.cpio //Descomprime el rootfs.cpio original
chmod 777 i2cdelect | cp i2cdelect /usr/bin/ //El archivo ejecutable debe tener permisos 777
sudo chown -R root:root .
buscar . | cpio -o -H newc > ../rootfs.cpio //Paquete nuevo rootfs.cpio
cp ../rootfs.cpio ns-bbl //copiar y usar
```
### Compilar
```
cd riscv-linux
crear nanhu_fpga_defconfig
cd ../
make sw -j200 #Después de completar, hay linux.bin en el directorio buid
```
![compilar.jpeg](../figs/docker_images/compilar.jpeg)
### Generar un archivo txt para grabar FPGA
Utilice las siguientes herramientas en el entorno Linux para generar un archivo txt para grabar FPGA desde linux.bin

[bin2fpgadata.tar.gz](https://raw.githubusercontent.com/OpenXiangShan/XiangShan-doc/main/docs/integration/resources/bin2fpgadata.tar.gz)
```
#Generar datos.txt
<ruta bin2fpgadata>/bin2fpgadata -i linux.bin
```
## prueba
De acuerdo con la operación demostrada en [Construcción de un sistema mínimo de código abierto de FPGA basado en Nanhu](https://xiangshan-doc.readthedocs.io/zh_CN/latest/integration/fpga/#_1), coloque el archivo txt generado en El sistema mínimo lo puedes probar en
