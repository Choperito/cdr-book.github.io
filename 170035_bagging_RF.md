# Métodos ensamblados: bagging y random forest {#cap-bagg-rf}

*Ramón A. Carrasco*$^{a}$ e *Itzcóatl Bueno*$^{b,a}$

$^{a}$Universidad Complutense de Madrid 
$^{b}$Instituto Nacional de Estadística


## Introducción a los métodos ensamblados

Puede ocurrir que ninguno de los algoritmos hasta ahora presentados (Caps. \@ref(cap-arboles), \@ref(cap-svm), \@ref(cap-knn) y \@ref(cap-naive-bayes)) proporcionen resultados convincentes para el problema que se quiere modelar. El *aprendizaje ensamblado*\index{aprendizaje ensamblado} [@zhou2012ensemble] es un paradigma que, como muestra la Fig. \@ref(fig:metamodel), en lugar de entrenar un modelo muy preciso, se centra en entrenar un gran número de modelos con menor precisión, y después combinar sus predicciones para obtener un metamodelo de una precisión más alta.

<div class="figure" style="text-align: center">
<img src="img/metamodelo.png" alt="Esquema de un metamodelo." width="70%" />
<p class="caption">(\#fig:metamodel)Esquema de un metamodelo.</p>
</div>

A los modelos de menor precisión se les suele nombrar como algoritmos "débiles", es decir, algoritmos con menor capacidad de aprender patrones complejos en los datos.Por tanto, generalmente, son rápidos tanto en tiempo de entrenamiento como de procesamiento. Existen dos paradigmas de aprendizaje ensamblado\index{aprendizaje ensamblado}: el *bagging*\index{bagging} y el *boosting*\index{boosting} (Cap. \@ref(cap-boosting-xgboost)).

## Bagging

En lugar de buscar la división más eficiente en cada capa, como ocurre en el árbol de decisión\index{árbol!de decisión}, una alternativa sería construir un metamodelo combinando los resultados de múltiples árboles de decisión. Esta técnica se conoce como *bagging*\index{bagging} y consiste en construir varios árboles\index{árbol!de decisión} utilizando una selección aleatoria de los datos que se utilizan para cada árbol y, finalmente, combinar la predicción de cada uno de ellos a través de la media (en el caso de regresión\index{árbol!de regresión}) o mediante un sistema de votación (en el caso de un problema de clasificación\index{árbol!de clasificación}). 

La principal característica del *bagging*\index{bagging} es el llamado muestreo bootstrap\index{bootstrap}. La idea básica del bootstrap es que la inferencia sobre una población se haga a partir de una muestra, tomando el papel de población y se remuestree, permitiendo comparar valor poblacional y el valor muestral. En el *bagging*, el objetivo de este remuestreo es que cada árbol\index{árbol!de decisión} esté entrenado con una muestra única, y por tanto, generen respuestas únicas,esto es modelos débiles distintos. Para ello, debe existir aleatoriedad y variación en cada árbol\index{árbol!de decisión} que conforme el modelo final, puesto que no tendría sentido construir varios árboles idénticos. Como se ha comentado, este problema queda resuelto por el muestreo bootstrap\index{bootstrap}, el cual extrae una muestra aleatoria de los datos en cada ronda. En el caso del *bagging*\index{bagging}, se extraen distintas muestras de datos para el entrenamiento de cada árbol\index{árbol!de decisión}. Aunque esto no elimina la problemática del sobreajuste\index{sobreajuste}, los patrones presentes en el conjunto de datos aparecerán en la mayoría de los árboles\index{árbol!de decisión} entrenados y, por tanto, en la predicción final. Es por ello que el *bagging*\index{*bagging*} es una técnica de gran eficacia para el tratamiento de los valores atípicos y para la reducción de la varianza que generalmente afecta a un modelo compuesto por un único árbol de decisión\index{árbol!de decisión}.

### Procedimiento con R: la función `bagging()` {#rbagging}

En el paquete `ipred` de **R** se encuentra la función `bagging()` que se utiliza para entrenar un modelo *bagging*:


```r
bagging(formula, data, ...)
```

+ `formula`: Refleja la relación lineal entre la variable dependiente y los predictores $Y \sim X_1 + ... + X_p$.
+ `data`: Conjunto de datos con el que se entrena el modelo.
+ `nbagg`: Número de replicaciones bootstrap.
+ `coob`: Indica si se debe calcular una estimación del ratio de error de predicción. 

### Implementando *bagging* en R

Es posible la implementación de un modelo de predicción de agregación bootstrap en R. Para ello, se pueden utilizar múltiples funciones como la ya mencionada en la Sec. \@ref(rbagging) `bagging()`. En este ejemplo se utilizan los datos sobre compras de clientes `dp_entr` del paquete `CDR`, cuyo objetivo es clasificar a los clientes entre quienes comprarían un nuevo producto y quienes no.


```r
library("CDR")
library("ipred")
library("caret")
library("reshape")
library("ggplot2")

data("dp_entr")
```


```r
# se fija la semilla aleatoria
set.seed(101)

# Se entrena el modelo
bag_model <- bagging(
  formula = CLS_PRO_pro13 ~ .,
  data = dp_entr,
  nbagg = 100,  
  coob = TRUE,
  control = rpart.control(minsplit = 2, cp = 0)
)
```




```r
bag_model

Bagging classification trees with 100 bootstrap replications 

Call: bagging.data.frame(formula = CLS_PRO_pro13 ~ ., data = dp_entr, 
    nbagg = 100, coob = TRUE, control = rpart.control(minsplit = 2, 
        cp = 0))

Out-of-bag estimate of misclassification error:  0.1416 
```

El error de clasificación de este modelo es del 14,16%, o lo que es equivalente, el modelo tiene una precisión del 85,84%. Desafortunadamente, `bagging()` no selecciona el número óptimo de replicaciones reduciendo el error de clasificación. Para seleccionar el número de replicaciones que minimice el error, se puede graficar la curva de error por número de replicaciones como en la Fig. \@ref(fig:bagg-plot). Se itera el modelo variando los valores del hiperparámetro `nbagg` (en este ejemplo entre 10 y 150, incrementándose de cinco en cinco). Se observa que el error mínimo (13,79%) se obtiene al establecer el hiperparámetro igual a 60. 


```r
missclass <- c() # vector vacio para recopilar el error en cada iteración
for (n in seq(10,150,5)) { # valores a probar para nbagg
  # se establece la semilla aleatoria
  set.seed(101)
  # se entrena el modelo
  bag_model <- bagging(
  formula = CLS_PRO_pro13 ~ .,
  data = dp_entr,
  nbagg = n,  
  coob = TRUE,
  control = rpart.control(minsplit = 2, cp = 0)
  )
  # se agrega el error de esta iteración
  missclass <- c(missclass, bag_model$err) # se agrega el error de esta iteración
}
```


```r
plot(seq(10,150,5),missclass,type = "l",xlab = "Número de árboles", ylab="Missclassification error")
```

<div class="figure" style="text-align: center">
<img src="img/bagging_missclass.png" alt="Número de replicaciones vs Error de clasificación." width="60%" />
<p class="caption">(\#fig:bagg-plot)Número de replicaciones vs Error de clasificación.</p>
</div>

La función `train()` del paquete `caret` es otro método para entrenar un algoritmo de *bagging* en R. Para ello, el argumento `method` debe tomar el valor `"treebag"`. Sin embargo, este algoritmo no incluye hiperparámetros a optimizar. Dado que se ha obtenido recursivamente el número óptimo de replicaciones, se puede entrenar el modelo con el valor obtenido y comprobar que el error dado coincide. Se observa que si se entrena un modelo *bagging* con 60 replicaciones, la precisión del modelo es del 86,93%. Esto es aproximadamente el resultado obtenido anteriormente en el que para 60 replicaciones el modelo tenía un error de clasificación del 13,79%.


```r
set.seed(101)
model_bag <- train(
  CLS_PRO_pro13 ~ .,
  data = dp_entr,
  method = "treebag",
  trControl = trainControl(method = "cv", number = 10),
  nbagg = 60, 
  control = rpart.control(minsplit = 2, cp = 0)
)
```




```r
model_bag

Bagged CART 

558 samples
 17 predictor
  2 classes: 'S', 'N' 

No pre-processing
Resampling: Cross-Validated (10 fold) 
Summary of sample sizes: 502, 502, 502, 503, 503, 502, ... 
Resampling results:

  Accuracy   Kappa    
  0.8692532  0.7385449
```

### Interpretación de variables en el *bagging*

Una de las principales desventajas de los algoritmos ensamblados (incluido el *bagging*\index{bagging}) es que mientras que los modelos base son interpretables, el metamodelo resultante no lo es. Pese a esto, aún es posible hacer inferencia de cómo cada una de las variables influye en el modelo entrenado. La manera de medir la importancia\index{importancia} de las variables incluidas en un árbol es registrar para cada variable la reducción de la función de pérdida que se le atribuye en cada partición. Dado que una variable puede utilizarse varias veces para dividir el árbol\index{árbol!de decisión}, la importancia total de esa variable será la suma de la reducción de la función de pérdida que se le atribuya por todas las particiones en las que intervenga. Este proceso es similar para el *bagging*. En este caso, para cada árbol se calcula la reducción de la función de pérdida en todas las divisiones. Tras esto, se agrega esta medida en todos los árboles que forman el metamodelo. El paquete `ipred`, en el que se encuentra la función `bagging()`, no captura la información requerida para calcular la importancia de las variables. Sin embargo, el paquete `caret` si lo hace y se puede construir un gráfico de importancia utilizando la función `vip()` del paquete `vip`.


```r
library("vip")
vip(model_bag, num_features = 15,
    aesthetics = list(color = "skyblue", fill = "skyblue"))
```

<div class="figure" style="text-align: center">
<img src="img/model_bag_imp.png" alt="Importancia de las variables incluidas en el modelo bagging." width="60%" />
<p class="caption">(\#fig:BAGGINGVIP)Importancia de las variables incluidas en el modelo bagging.</p>
</div>

La Fig. \@ref(fig:BAGGINGVIP) muestra que las variables más importantes en el modelo *bagging* entrenado para predecir si un cliente comprará o no el *tensiómetro digital* son: si ha comprado la *depiladora eléctrica*, cuánto importe ha gastado en ese producto, si ha comprado el *estimulador muscular* y si ha comprado el *smartchwatch fitness*.

## Random Forest

El *bagging*\index{bagging} es el paradigma tras el algoritmo de *random forest*\index{random forest}. Este algoritmo fue desarrollado por primera vez por [@ho1995random]. Sin embargo, fueron [@cutler1999fast] y [@breiman2001random] quienes desarrollaron una versión extendida del modelo y registraron **Random Forest** como marca comercial. Este algoritmo básico de *bagging* funciona del siguiente modo: a partir del conjunto de datos de entrenamiento se generan $K$ muestras aleatorias $\mathbb{S}_{k}$, se entrena un modelo de árbol de decisión ($f_k$) utilizando la muestra $\mathbb{S}_{k}$ como conjunto de entrenamiento. Tras el entrenamiento, se dispone de $K$ árboles de decisión, como se observa en la Fig. \@ref(fig:ejemplo-rf). La predicción de una nueva observación $x$ se obtiene como la media de las $K$ predicciones:

\begin{equation}
y\leftarrow\hat{f}(x)=\frac{1}{K}\sum^{K}_{k=1}f_{k}(x)
\end{equation}

En el caso de regresión, o por la votación por mayoría en el caso de clasificación.

Tanto el *bagging*\index{bagging} como el *random forest*\index{random forest} desarrollan múltiples árboles y utilizan el muestreo bootstrap\index{bootstrap} para la aleatorización de los datos. Sin embargo, el *random forest* establece una limitación artificial a la selección de variables al no considerar todas en cada árbol.

<div class="figure" style="text-align: center">
<img src="img/randomforest.png" alt="Ejemplo de Random Forest." width="60%" />
<p class="caption">(\#fig:ejemplo-rf)Ejemplo de Random Forest.</p>
</div>

El *bagging*\index{bagging} considera las mismas variables para construir cada árbol\index{árbol!de decisión} con el objetivo de minimizar su entropía\index{entropía}, y, por tanto, todos los árboles suelen tener un aspecto similar. Esto lleva a que las predicciones dadas por los árboles estén altamente correlacionadas. El modelo *random forest*\index{random forest} evita este problema estableciendo la obligación, en cada división, de utilizar un subconjunto de las variables. Esto proporciona a algunas variables mayor probabilidad de ser seleccionadas, y al generar árboles únicos y no correlacionados se consigue una estructura de decisión final más fiable. 

En general, es mejor que el *random forest* esté formado por una gran cantidad de árboles (por lo menos 100) para suavizar el impacto de valores atípicos. Sin embargo, la tasa de efectividad disminuye a medida que se incorporan más árboles. Llegado a cierto punto, los nuevos árboles no aportan una mejora significativa al modelo, pero si incrementan los tiempos de procesamiento. 

El modelo *random forest* es rápido de entrenar y es una buena técnica para obtener un modelo de referencia. Aunque estos modelos funcionan bien en la interpretación de patrones complejos y son versátiles, otras técnicas, como por ejemplo el *gradient boosting*\index{gradient boosting} (Cap. \@ref(cap-boosting-xgboost)), proporcionan una mayor precisión en las predicciones en muchos casos.

Estos modelos se han vuelto populares porque tienden a proporcionar un muy buen rendimiento con los parámetros  predeterminados en las distintas implementaciones. En efecto, a pesar de tener muchos hiperparámetros\index{hiperparámetro} que pueden ser ajustados, los valores por defecto de dichos hiperparámetros tienden a ofrecer buenos resultados en la predicción. Los hiperparámetros más importantes que hay que ajustar al entrenar un modelo *random forest* son: el número de árboles ($K$), el número de variables incluidos en el subconjunto aleatorio en cada división (`mtry`), la complejidad\index{complejidad} de cada árbol, el esquema de muestreo y la regla de división a utilizar durante la construcción del árbol.

### Número de árboles ($K$)\index{número!de árboles}

El primer hiperparámetro a ajustar es el número de árboles que componen el modelo de *random forest*\index{random forest}. Su valor debe ser lo suficientemente grande como para que la tasa de error se estabilice. La regla general es que el valor mínimo de árboles sea igual a 10 veces el número de variables incluidas en el modelo. Sin embargo, cuando se tienen en cuenta otros hiperparámetros para optimizar, es posible que el número de árboles se vea afectado. El tiempo de procesamiento aumenta linealmente con la cantidad de árboles incluidos, pero cuantos más se incluyan, se obtendrán estimaciones de error más estables.

### Número de variables a considerar (`mtry`)

`mtry` se refiere al hiperparámetro\index{hiperparámetro} encargado de controlar la aleatorización de variables utilizadas para las particiones de los árboles. Este hiperparámetro ayuda a equilibrar la baja correlación del árbol con los demás, y una razonable fuerza predictiva. Existe un valor predeterminado para este hiperparámetro el cual se puede utilizar en caso de no querer o no poder ajustarlo. En el caso de la regresión\index{árbol!de regresión}, se determina que $mtry=\frac{p}{3}$ siendo $p$ el número de variables incluidas en el modelo. Y en problemas de clasificación\index{árbol!de clasificación}, el valor predeterminado es $mtry=\sqrt p$. Cuando hay pocas variables relevantes, es decir, los datos son muy ruidosos, tiende a funcionar mejor que el valor de `mtry` sea alto, pues hace que sea más probable seleccionar esas variables. En cambio, cuando muchas variables son importantes, funciona mejor un valor bajo de `mtry`.

### Complejidad de los árboles

Un modelo *random forest*\index{random forest} se construye con árboles de decisión\index{árbol!de decisión} a los que se les puede controlar su profundidad\index{profundidad!del árbol} y su complejidad\index{complejidad} como se vio en el Cap. \@ref(cap-arboles). Esto se puede hacer ajustando los hiperparámetros\index{hiperparámetro} de profundidad máxima permitida, tamaño del nodo o la cantidad máxima de nodos terminales. 

El tamaño del nodo es el hiperparámetro más común para controlar la complejidad\index{complejidad} del árbol y la mayoría de las implementaciones usan los valores predeterminados de 1 para árboles de clasificación\index{árbol!de clasificación} y 5 para los árboles de regresión\index{árbol!de regresión}, dado que estos valores tienden a producir buenos resultados. Si se quiere controlar el tiempo de procesamiento, se pueden conseguir reducciones significativas del tiempo aumentando el tamaño del nodo impactando de manera marginal en la estimación del error.

### Esquema de muestreo

Por defecto, el *random forest*\index{random forest} tiene como esquema de muestreo el bootstrapping\index{bootstrap}, explicado anteriormente, en el cual todas las observaciones se muestrean con reemplazo. Todas las replicaciones de bootstrap tienen el mismo tamaño que el conjunto de datos de entrenamiento. Sin embargo, el esquema de muestreo se puede ajustar tanto en el tamaño de la muestra como en el diseño muestral (con o sin reposición). El hiperparámetro de tamaño de muestra determina cuántas observaciones se extraen para el entrenamiento de cada árbol. Cuanto menor sea el tamaño muestral, menor será la correlación entre los árboles, lo cual puede llevar a mejores resultados de precisión en la predicción. La forma de determinar el tamaño muestral óptimo puede hallarse evaluando algunos valores que oscilen entre el 25% y el 100%, y en el caso de que haya variables no balanceadas respecto a los valores de las categóricas se puede intentar muestrear sin reposición. 

### Regla de división

Por defecto, la regla de división\index{partición} que utilizan los árboles de decisión\index{árbol!de decisión} que conforman un *random forest*\index{random forest} es la que se presentó en el Cap. \@ref(cap-arboles). Esto es, en el caso de regresión\index{árbol!de regresión} seleccionar la división que minimiza la desviación típica $(\sigma)$; y en el caso de clasificación\index{árbol!de clasificación} la división que minimiza la impureza de Gini\index{impureza!de Gini} o la entropía\index{entropía}.

### Procedimiento con R: la función `randomForest()`

En el paquete `randomForest` de **R** se encuentra la función `randomForest()` que se utiliza para entrenar un modelo de este tipo:


```r
randomForest(formula, data=..., ...)
randomForest(x, y, xtest, ytest, ntree=500, mtry, ...)
```

+ `formula`: Refleja la relación entre la variable dependiente $Y$ y los predictores tal que $Y \sim X_1 + ... + X_p$.
+ `data`: Conjunto de datos con el que entrenar el árbol de acuerdo a la fórmula indicada. 
+ `x`: Conjunto de datos de entrenamiento que contiene los predictores
+ `y`: Vector respuesta con las clases o valores de la variable respuesta.
+ `xtest`: Conjunto de datos que contiene los predictores del conjunto de datos de validación.
+ `ytest`: Variable respuesta del conjunto de datos de validación.
+ `ntree`: Número de árboles a construir en el modelo.
+ `mtry`: Número de variables muestreadas aleatoriamente como candidatas en cada partición.

### Aplicación del modelo *random forest* en **R**

En esta sección se aplica el modelo *random forest* al ejemplo de datos de retail incluido en el paquete `CDR`. Se carga el paquete y con ello, los datos `dp_entr`. Se busca predecir si un cliente va a comprar o no el nuevo producto de acuerdo a los productos que ha consumido, el importe que gasta en ellos y otras características como, por ejemplo, su nivel educativo.


```r
library("CDR")
library("randomForest")
library("caret")
library("reshape")
library("ggplot2")
data(dp_entr)
```

Este algoritmo al estar basado en árboles de clasificación tiene los mismos requisitos para el entrenamiento que tenían dichos árboles, así se construye el modelo usando el conjunto de datos de entrenamiento.


```r
# se fija la semilla aleatoria
set.seed(101)

# se entrena el modelo
model <- train(CLS_PRO_pro13~., data=dp_entr_NUM, 
             method="rf", metric="Accuracy", ntree=500,
             trControl=trainControl(method="cv", 
                                    number=10, 
                                    classProbs = TRUE))
```




```r
model

Random Forest 

558 samples
 19 predictor
  2 classes: 'S', 'N' 

No pre-processing
Resampling: Cross-Validated (10 fold) 
Summary of sample sizes: 502, 502, 502, 503, 503, 502, ... 
Resampling results across tuning parameters:

  mtry  Accuracy   Kappa    
   2    0.8602922  0.7206238
  10    0.8620455  0.7241029
  19    0.8620130  0.7240248

Accuracy was used to select the optimal model using the largest value.
The final value used for the model was mtry = 10.
```

Los resultados de la validación cruzada se pueden ver en el siguiente boxplot. Se observa como la precisión oscila entre el 80% y el 95%. Además, se puede ver en el resultado del modelo que el hiperparámetro $mtry$ se ha ajustado a 10 variables. 



<div class="figure" style="text-align: center">
<img src="img/rfboxplot.png" alt="Resultados del modelo random forest durante el proceso de validación cruzada." width="60%" />
<p class="caption">(\#fig:RFRESULTS)Resultados del modelo random forest durante el proceso de validación cruzada.</p>
</div>

Finalmente, aunque el *random forest* generado está compuesto por 500 árboles, se puede acceder a cualquiera de ellos para estudiarlos en profundidad. Para ello, es necesario instalar el paquete `reprtree` desde el repositorio <https://github.com/araastat/reprtree>.


```r
library("devtools")
if(!('reprtree' %in% installed.packages())){
  devtools::install_github('araastat/reprtree')
}
```

Se pueden observar las decisiones que se toman en el árbol de forma tabulada, indicando qué variable se utiliza para la partición, cuál es el valor que decide la división, indicando si es un nodo terminal (`-1`) o no (`1`) y la predicción del nodo, el cual es `NA` si no es un nodo terminal.


```r
set.seed(101)
rf <- randomForest(CLS_PRO_pro13~., data = dp_entr_NUM, ntree=500,
                   mtry=unlist(model$bestTune))

# se observa el árbol número 205
tree205 <- getTree(rf, 205, labelVar=TRUE)

head(tree205[,-c(1,2)])
      split var split point status prediction
1 importe_pro15         100      1       <NA>
2 importe_pro12          60      1       <NA>
3 importe_pro16          90      1       <NA>
4  ingresos_ano      156500      1       <NA>
5 importe_pro17         150      1       <NA>
6      anos_exp          33      1       <NA>
  
tail(tree205[,-c(1,2)])
               split var split point status prediction
120                 <NA>         0.0     -1          N
121                 <NA>         0.0     -1          S
122 des_nivel_edu.BASICO         0.5      1       <NA>
123                 <NA>         0.0     -1          S
124                 <NA>         0.0     -1          S
125                 <NA>         0.0     -1          N
```

Este árbol se muestra en la Fig. \@ref(fig:tree-plot).


```r
library("reprtree")
plot.getTree(rf, k=205)
```

<div class="figure" style="text-align: center">
<img src="img/rf_tree205.png" alt="Árbol número 205 del random forest entrenado." width="70%" />
<p class="caption">(\#fig:tree-plot)Árbol número 205 del random forest entrenado.</p>
</div>

Sin embargo, el método por el que se representa gráficamente no es muy claro y puede llevar a confusión o dificultar la interpretación del árbol. Si se desea estudiar hasta cierto nivel del árbol, se puede incluir el argumento `depth` como en el ejemplo abajo mostrado, y que representa el mismo árbol con una profundidad de 5 ramas en la Fig. \@ref(fig:tree-plot2).


```r
plot.getTree(rf, k=205, depth = 5)
```

<div class="figure" style="text-align: center">
<img src="img/rf_tree205_depth5.png" alt="Árbol número 205 del random forest entrenado hasta la capa 5." width="70%" />
<p class="caption">(\#fig:tree-plot2)Árbol número 205 del random forest entrenado hasta la capa 5.</p>
</div>

#### Aplicación del modelo *random forest* con ajuste automático

En este segundo ejemplo, se pretende mejorar la precisión del modelo anterior. Para ello, se ajusta de forma automática\index{ajuste automático} los hiperparámetros de dicho algoritmo. De los mencionados anteriormente, solo se va a ajustar automáticamente `mtry`, que es el único incluido el método `rf`.


```r
modelLookup("rf")
  model parameter                         label forReg forClass probModel
1    rf      mtry #Randomly Selected Predictors   TRUE     TRUE      TRUE
```

Para ajustar el número de árboles y el resto de hiperparámetros, se puede iterar el modelo y probar distintos valores. En una red de opciones se incluyen los valores a probar para el hiperparámetro `mtry`. 


```r
# Se especifica un rango de valores posibles de mtry
tuneGrid <- expand.grid(mtry = 1:18)
```

A continuación, se entrena el modelo para que se ajuste al valor de `mtry` que maximice el rendimiento predictivo del modelo.


```r
# se fija la semilla aleatoria
set.seed(101)

# se entrena el modelo
model <- train(CLS_PRO_pro13 ~ ., data=dp_entr_NUM, 
               method = "rf", metric = "Accuracy",
               tuneGrid = tuneGrid,
               trControl = trainControl(classProbs = TRUE))
```




```r
model

Random Forest 

558 samples
 19 predictor
  2 classes: 'S', 'N' 

No pre-processing
Resampling: Bootstrapped (25 reps) 
Summary of sample sizes: 558, 558, 558, 558, 558, 558, ... 
Resampling results across tuning parameters:

  mtry  Accuracy   Kappa    
   1    0.8641354  0.7283098
   2    0.8650087  0.7298376
   3    0.8629614  0.7256812
   4    0.8635609  0.7268514
   5    0.8639559  0.7276250
   6    0.8612659  0.7222420
   7    0.8604934  0.7206476
   8    0.8610116  0.7216937
   9    0.8590645  0.7177882
  10    0.8589073  0.7174718
  11    0.8607248  0.7211179
  12    0.8583609  0.7163903
  13    0.8587296  0.7170933
  14    0.8587384  0.7171642
  15    0.8583195  0.7163106
  16    0.8585407  0.7167355
  17    0.8573597  0.7144030
  18    0.8581404  0.7159558

Accuracy was used to select the optimal model using the largest value.
The final value used for the model was mtry = 2.
```

Mientras que en el ejemplo anterior el algoritmo sólo probó tres valores de `mtry`, esta vez se realiza una prueba exhaustiva de valores. En el primer ejemplo, el valor del hiperparámetro era `mtry=10`, pero ahora se ha reajustado a `mtry=2`. Esto es equivalente a decir que 2 variables seleccionadas en cada partición son suficientes, y que no son necesarias 10 como en el ejemplo anterior. Finalmente, se puede observar en la Fig. \@ref(fig:rfresults2) los resultados obtenidos durante la validación cruzada. Se observa cómo no sólo la precisión es mayor que en el ejemplo anterior, sino que además los resultados tienen menos dispersión.


```r
ggplot(melt(model$resample[,-4]), aes(x = variable, y = value, fill=variable)) + 
  geom_boxplot(show.legend=FALSE) + 
  xlab(NULL) + ylab(NULL)
```

<div class="figure" style="text-align: center">
<img src="img/rftunedboxplot.png" alt="Resultados obtenidos por el random forest con ajuste automático durante el proceso de validación cruzada." width="60%" />
<p class="caption">(\#fig:rfresults2)Resultados obtenidos por el random forest con ajuste automático durante el proceso de validación cruzada.</p>
</div>

::: {.infobox_resume data-latex=""}
### Resumen {-}
En este capítulo se introduce al lector en el *bagging* y el algoritmo de aprendizaje supervisado conocido como *random forest*, en concreto:

- Se presenta el concepto de aprendizaje ensamblado, y se profundiza en uno de sus paradigmas: el *bagging*.
- Se implementa el *bagging* en `R` a través de un caso de clasificación binaria.
- Se expone cómo medir la importancia de las variables incluidas en un modelo *bagging* para facilitar su interpretación.
- Se explica el modelo *random forest*, fundamentado en los árboles decisión y en el *bagging*. Así como los hiperparámetros más importantes para ajustar el modelo de mayor precisión.
- Se presenta un ejemplo de clasificación binaria utilizando el modelo *random forest* en `R`.
:::