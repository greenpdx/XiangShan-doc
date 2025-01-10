# ¿Qué es esto?

Esta es la documentación de Xiangshan creada con la herramienta [mkdocs](https://www.mkdocs.org/). Esta herramienta puede convertir documentos en formato Markdown en páginas web y mostrarlos de una manera atractiva. Los proyectos creados con mkdocs pueden alojarse en Internet mediante readthedocs para facilitar el acceso. Proyectos como [Chipyard](https://chipyard.readthedocs.io/en/stable/index.html) y [BOOM](https://docs.boom-core.org/) hacen públicos sus documentos de esta manera. (La ligera diferencia es que ellos utilizan el formato estándar de escritura RST, mientras que nosotros preferimos Markdown)

# Proceso para agregar nuevo contenido

Este documento está incluido en el proyecto [XiangShan-doc](https://github.com/OpenXiangShan/XiangShan-doc) en Github. Cada vez que la rama principal recibe una actualización git push, la página web se reconstruye automáticamente para mostrar el nuevo contenido.

Entonces, si deseas agregar contenido nuevo, sigue estos pasos:

- Clonar el proyecto [XiangShan-doc](https://github.com/OpenXiangShan/XiangShan-doc).
- Instalar el entorno mkdocs localmente (ver [Instalación de MkDocs](https://www.mkdocs.org/user-guide/installation/))
- Instalar el tema Material para MkDocs localmente (ver [Material para MkDocs](https://squidfunk.github.io/mkdocs-material/getting-started/)) (TLDR: pip install -r docs/requirements.txt)
- Modificar el contenido del documento localmente
- Utilice el comando mkdocs serve para obtener una vista previa de la documentación localmente y verificar si cumple con sus expectativas en el navegador.
-  `git push`` al control remoto

# Estructura del proyecto

La estructura del proyecto mkdocs es muy sencilla. Hay un archivo mkdocs.yml en el directorio raíz para registrar la información de configuración. Necesitamos centrarnos en el subelemento nav, que determina la estructura de directorio de todo el documento que se muestra en la página web y se puede modificar según corresponda; la carpeta docs debajo del directorio raíz guarda los documentos en formato markdown, que es también lo que Necesitamos agregar o el contenido real de la modificación.

# Material para MkDocs

我们使用 Material for MkDocs 基于以下的原因：

1. Mejor soporte para encabezados de varios niveles
1.  Amigable para exportar a pdf
1.  Hay muchas cosas que puedes hacer con él, consulta: https://squidfunk.github.io/mkdocs-material/reference/ce/

!!! note
   Por ejemplo, este tipo de trabajo de lujo. 

# Exportar a pdf.

Al utilizar el tema Material para MkDocs con el complemento mkdocs-with-pdf, podemos exportar automáticamente todo el documento como un archivo pdf. mkdocs.yml ya contiene una configuración de exportación de PDF preestablecida. Para obtener información sobre cómo configurar el entorno de exportación de PDF y cómo usarlo, consulte https://pypi.org/project/mkdocs-with-pdf/

!!! info
    Tenga en cuenta que el tema nativo tiene algunos problemas al exportar bloques de código a PDF. Se recomienda utilizar el tema de material al exportar PDF.
