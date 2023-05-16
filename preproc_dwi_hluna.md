# Preprocesar DWI in vivo

Este tutorial aplica para las imágenes que adquiere Hiram en ratas in vivo, con adquisición EPI 2D (rebanadas de aprox 1 mm), pero puede ser bastante flexible para otras adquisiciones similares. Para adquisiciones ex vivo con DTI 3D es mejor usar otras herramientas.

## Convertir
Usar `brkraw`. Hay instrucciones completas [aquí](https://github.com/c13inb/c13inb.github.io/wiki/brkraw_tutorial), pero lo vamos a resumir. 
1. Tener activo [anaconda](https://github.com/c13inb/c13inb.github.io/wiki/Anaconda).
2. Usar el ambiente de brkraw, lo que se hace con:
```
conda activate /home/inb/lconcha/fmrilab_software/inb_anaconda3/envs/brkraw 
```  
3. Para convertir, usamos:
```
brkraw gui -i $RUTA
```
donde `$RUTA` significa la carpeta donde están los archivos en formato bruker. Esto es fácil, pues es posible ver el disco duro del resonador desde cualquier PC de don clusterio. Podemos escribir la ruta completa, o también arrastrar el estudio que nos interesa desde un navegador de archivos:
![](https://hackmd.io/_uploads/SyT_oGWSn.gif)

4. Seleccionamos lo que queremos convertir (:one:), indicamos en qué carpeta lo queremos grabar (:two:), y damos clic en _Convert_ (:three:).
![](https://hackmd.io/_uploads/HJJS2MZHn.png)

En el caso de DWI, esto debe generar cuatro archivos, todos con el mismo nombre, pero diferentes extensiones:  `bval`, `bvec`, `bmat`, y `nii.gz`. El nombre del archivo comienza con la fecha de adquisición, e incluye el nombre que se le haya dado a la rata/estudio.
![](https://hackmd.io/_uploads/SkxhhzWr2.png)

## Preprocesar
Ahora estamos listos para preprocesar los datos. Es conveniente tener una carpeta donde vamos a ir colocando los nuevos archivos, de manera que no se nos anden amontonando con los datos crudos. Dependiendo de nuestros gustos, puede ser una carpeta para todas las ratas, o una carpeta para cada rata. Voy a elegir lo segundo para este tutorial, haciendo una carpeta `preproc`, y adentro de ella una carpeta para cada rata. Por ejemplo, la rata6 va a quedar en `preproc/rata6`.

Hacemos la carpeta `proc` y luego la de la rata

    mkdir proc
    mkdir proc/rata6

Y ahora sí, podemos correr el script de preprocesamiento que corre (casi) todo lo que está descrito [aquí](https://github.com/c13inb/c13inb.github.io/wiki/dwipreproc-rat).

El comando en forma general es:

    inb_dwi_bruker_preproc.sh -i $DWIcrudas.nii.gz \
    -o $RUTAdeSALIDA \
    -m \
    -s 10 \
    -x -z

Donde con `-i DWIcrudas.nii.gz` estamos diciendo que el input (`-i`) es el archivo que contiene las imágenes DWI. Se asume que existen archivos `bval` y `bvec` con exactamente el mismo nombre (si no es así, el comando va a fallar). La carpeta de salida (output, `-o`) es donde queremos grabar los resultados del preprocesamiento. No se recomienda usar solo la carpeta, sino indicar algún prefijo para los datos de salida (por ejemplo, usar `proc/rata6/dwi` en lugar de solo `proc/rata6`), porque así los nombres de los archivos generados serán más adecuados. Se vale usar como prefijo algo como `PTZ`,  `control`, o lo que queramos. `-s 10` está diciendo que escale las imágenes 10X en tamaño, para que la corrección de movimiento funcione bien (al terminar el script regresará las imágenes resultantes a las dimensiones originales). `-m` está pidiendo corrección de movimiento. Por último `-x` y `-z` están diciendo que hay que cambiarle el signo a los componentes $x$ y $z$ de los `bvecs` por razones propias del convertidor brkraw (esto puede ser distinto para otras adquisiciones que no son las de Hiram, o si cambiamos de convertidor).

:warning: Este comando va a mandar llamar a `eddy`, que necesita NVIDIA CUDA. No todas las máquinas del cluster tienen esa capacidad. Revisa [ésto](https://github.com/c13inb/c13inb.github.io/wiki/CUDA) para activar CUDA, pero para hacer el cuento corto, corremos lo siguiente:

    source /home/inb/lconcha/fmrilab_software/tools/inb_config_cuda9.sh

Si nos responde que no se encontró CUDA, hay que usar otra PC que sí tenga CUDA. Para saber qué PCs tienen CUDA, podemos usar el comando `qstat -q nvidia.q -f`, y nos va a dar el chisme.

Ahora sí, podremos correr el script `inb_dwi_bruker_preproc.sh`.

Entonces, para nuestro caso en particular de este tutorial, el comando queda:

    inb_dwi_bruker_preproc.sh -i 230504_hluna_PTZirm4_rata6_hluna_PTZirm4_rata6_1_6_1.nii.gz -o proc/rata6/dwi -m -s 10 -x -z


El script tarda aproximadamente 15 minutos en terminar.

### Preprocesando varias imágenes a través del cluster

Podemos mandar el comando al cluster, pidiéndole explícitamente que lo mande a puras PCs con CUDA. Esto es fácil, solo tenemos que agregar (antes del comando) las siguientes opciones: `fsl_sub -q nvidia.q`.

:warning: Asegúrate que estás usando la versión correcta de fsl (para que use bien `fsl_sub`) con el comando `fsl602`.

Nuestro comando completo queda:
    
    fsl_sub -q nvidia.q -N preRata6 inb_dwi_bruker_preproc.sh -i 230504_hluna_PTZirm4_rata6_hluna_PTZirm4_rata6_1_6_1.nii.gz -o proc/rata6/dwi -m -s 10 -x -z
    
En lugar de salir todo el output de nuestro comando en la terminal, nos dará un número, que es nuestro ticket del trabajo (_job_) que acabamos de enviar al cluster. Lo que hubiera salido a la terminal se va a unos archivos que se llaman como le pusimos de nombre a nuestro job (con `-N` y en este ejemplo se llama `preRata6`) y luego tienen sufijos `.e????` y `.o????` donde `????` indica el job-ID.

Para revisar el progreso de nuestro comando enviado al cluster, usamos `qstat`.

<pre>(brkraw) lconcha@mansfield:/misc/mansfield/lconcha/TMP/hiram$ qstat
job-ID  prior   name       user         state submit/start at     queue                          slots ja-task-ID 
-----------------------------------------------------------------------------------------------------------------
   3272 0.25000 preRata6   lconcha      r     05/16/2023 10:10:36 nvidia.q@copula2.inb.unam.mx       1  </pre>

El estado (_state_) tipo `r` significa corriendo (_running_). Un estado `q` significa que está en la cola, esperando una PC donde correr. Un estado con `E` significa error.

Para borrar un job, usamos `qdel $job-ID` donde `$job-ID` es el numerito del ticket, y el mismo que podemos ver en la primer columna de `qstat`, en nuestro ejemplo el `3271`

:information_source: Hay mucha más información de cómo usar Don Clusterio [aquí](https://github.com/c13inb/c13inb.github.io/wiki/Cl%C3%BAster%3A-Uso-del-cl%C3%BAster).