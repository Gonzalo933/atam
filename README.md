# ATAM

## Explicación de la solución

(todo el código esta disponible en los notebooks de la carpeta `notebooks/`)
### Exploración de datos

He comenzado realizando un análisis de los datos y un estudio de como agruparlos en un solo dataframe.

Se observan diferencias en los valores de aceleración (de cada eje), como que la distribución de las aceleraciones del eje `Y` presenta mas valores positivos que negativos.
Al calcular la distribución de la media de los valores absolutos para cada eje las distribuciones son más similares. Como regla general estos valores estan por debajo de 1.0

He hecho un pequeño análisis de correlaciones pero no he encontrado nada significativo.

Dado que cada medición de energía esta asociada con 1041 mediciones de aceleraciones hay que realizar una pequeña transformación para transformar cada `accx_minus_XXXX` a una columna.
El problema de este enfoque es que se generarían 3123 columnas para cada medición de energía. Esto es un problema ya que `numero columnas` >> `numero de muestras` y esto crea vectores de una dimensionalidad muy alta (ver https://en.wikipedia.org/wiki/Curse_of_dimensionality).

Este problema se podría afrontar con PCA o LDA, pero introduciendo conocimiento sobre el problema podemos llegar a la conclusión de que no estamos interesados en los valores concretos de la aceleración en cada momento, sino en como cambía durante todo el intervalo de tiempo. Para ello se ha construido un nuevo dataset con esta información.

Las nuevas columnas son:

- `x_y_z_module`: Suma de los módulos de cada una de las mediciones `accx_minus_XXXX`. Es decir, para cada `id_`: $\sum_1^T \sqrt{x_t^2 + y_t^2 + z_t^2}$ donde $t$ es cada una de las mediciones `accx_minus_XXXX`.
- `x_mean`, `y_mean`, `z_mean`: La media de las aceleraciones.
- `x_var`, `x_var`, `x_var`: La varianza de las aceleraciones.
- `x_std`, `x_std`, `x_std`: La desviación estándar de las aceleraciones.
- `x_min`, `x_min`, `x_min`: El valor mínimo de la aceleración.
- `x_max`, `x_max`, `x_max`: El valor máximo de la aceleración.

Adicionalmente se han creado 3 versiones de los datasets:

- V1: Se incluyen todas las columnas listadas arriba.
- V2: Se incluyen todas las columnas listadas arriba y el ritmo cardiaco.
- V3: Se incluyen todas las columnas listadas arriba y el gradiente.

Se ha separado en múltiples datasets para poder comparar si los cambios mejoran o empeoran los modelos.

## Modelos

Se han lanzado pruebas con los modelos de machine learning. Se trata de un problema de regresión, por lo que se han elegido modelos de regresión.

Para cada modelo se realiza lo siguiente:

- Se realiza validación cruzada con 4 splits para reducir la variablidad de los resultados (es decir, se entrena y predice 4 veces). Se calcula la media del MAPE de cada uno de los "folds" y se reporta media y desviación estándar.
- Para los modelos en los que haya que encontrar los mejores hiperparámetros se realiza una búsqueda en rejilla. Esta busqueda se hace en paralelo y se repite para cada fold de la validación cruzada explicada en el paso anterior.
- En todos los modelos, los datos son estandarizados.

Además de comparar los modelos, se comparan los datasets para ver si la inclusión del rimo cardiaco y los gradientes mejoran los resultados.

Nota: DummyRegressor es el baseline.
#### Dataset 1

|           Modelo          	| MAPE 	|
|:-------------------------:	|:----:	|
|       DummyRegressor      	|  0.84    	|
|     linear regression     	|   0.52   	|
|      ridge regression     	|  0.52    	|
|           lasso           	|  0.68    	|
|         elasticNet        	|  0.58    	|
|            svr            	|  0.36    	|
|   DecisionTreeRegressor   	| 0.54     	|
|    KNeighborsRegressor    	| 0.44     	|
| GradientBoostingRegressor 	| 0.42     	|


Buscamos el mínimo MAPE. Se obtienen los mejores resultados con una SVR con kernel gaussiano y `C=1`

#### Dataset 2

|           Modelo          	| MAPE 	|
|:-------------------------:	|:----:	|
|       DummyRegressor      	|  0.84    	|
|     linear regression     	|   0.52   	|
|      ridge regression     	|  0.52    	|
|           lasso           	|  0.70    	|
|         elasticNet        	|  0.58    	|
|            svr            	|  0.36    	|
|   DecisionTreeRegressor   	| 0.54     	|
|    KNeighborsRegressor    	| 0.44     	|
| GradientBoostingRegressor 	| 0.41     	|

Los cambios son mínimos, no parece que el ritmo cardiaco aporte mucha información

#### Dataset 3

|           Modelo          	| MAPE 	|
|:-------------------------:	|:----:	|
|       DummyRegressor      	|  0.84    	|
|     linear regression     	|   0.47   	|
|      ridge regression     	|  0.46    	|
|           lasso           	|  0.61    	|
|         elasticNet        	|  0.49    	|
|            svr            	|  0.36    	|
|   DecisionTreeRegressor   	| 0.51     	|
|    KNeighborsRegressor    	| 0.40     	|
| GradientBoostingRegressor 	| 0.38     	|

Los resultados mejoran levemente, parece que el gradiente de las aceleraciones si aporta algo de información.
## Cálculo de METS

Se han calculado los METS para cada valor de energía con la siguiente formula:

$E / 75 * (T / 3600) * 3600 = E / 75 * T$

Donde `E` es la energía medida y `T` es el tiempo de medición en segundos.

No he entendido muy bien donde aplica la parte de reducir el consumo de batería si se especifica que todas las mediciones se envían a un servidor externo, lo que evitaría que el wearable realizara ningún cálculo. No obstante se podría reducir el consumo de batería realizando menos mediciones de aceleración (es decir, menos `accx_minus_XXXX`) pero presumiblemente esto empeoraría los resultados de los modelo.

## Otras notas

Si esto fuera un proyecto real, los datos (.csv) nunca serían comiteados junto con el código. En su lugar usaría una solución como [DVC](https://dvc.org/) + almacenamiento S3. De esta manera, con el código se comitearía un hash de los datos en el estado actual, y cualquier persona del proyecto podría acceder  a cualquier versión de los mismos.

Por otro lado, para realizar los experimentos usaría un servidor de [mlflow](https://mlflow.org/) corriendo en EC2.

No he podido aplicarlo a este proyecto porque no tengo una cuenta personal de AWS.

## Trabajo futuro

- Crear el dataframe de medias, varianzas, std, usando el valor absoluto de las aceleraciones. La lógica detrás de de esta idea es que nos da igual el signo de la aceleración ya que sigue siendo trabajo para la persona que realiza el movimiento.

- Reducir el número de `accx_minus_XXXX` para simular menos mediciones del wearable y por tanto un ahorro de batería.