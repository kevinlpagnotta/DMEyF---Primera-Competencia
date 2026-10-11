# FV8 — Predicción de bajas (DMEyF, 1° competencia)

Notebook en R (`FV8.ipynb`, experimento `HT4960-FV8`) que entrena con 202103–202106 y predice los clientes de **202108** con mayor probabilidad de ser **BAJA+2**.


---

## Contenido del repositorio

| Archivo | Para qué |
|---|---|
| `FV8.ipynb` | Notebook completa. |
| `BO_log.txt` | Optimización bayesiana en base a 5 semillas y 50 iteraciones. **Imprescindible**: de acá salen las 5 recetas del ensamble y la receta para ejecutar canaritos. |
| `campos_finales.csv`, `filtro_canaritos_etapa1.csv`, `filtro_canaritos_etapa2.csv` | *(Opcionales)* Resultado de los canaritos. Si se copian a `exp/HT4960-FV8/`, la notebook los reutiliza y saltea la etapa más cara de cómputo (~8 h). |

---

## Cómo correrla

### 1. Requisitos

- **VM con 256 GB de RAM y 12 vCPU** (se usó `n2-custom-12-262144-ext`, aprovisionamiento *Standard*). Con 96 GB se queda sin memoria en los canaritos: el pico fue de ~117 GB.
- Disco de arranque de 256 GB.
- R con kernel de Jupyter `ir`. Los paquetes (`data.table`, `R.utils`, `lightgbm`, `Rcpp`, `mlr3`, `mlr3mbo`, `DiceKriging`, etc.) se instalan solos si faltan. `Rcpp` necesita compilador C++.

### 2. Archivos en su lugar

```
~/buckets/b1/datasets/competencia_01_crudo.csv     # dataset de la cátedra
~/buckets/b1/datasets/FV8.ipynb
~/buckets/b1/exp/HT4960-FV4/BO_log.txt             # obligatorio
~/buckets/b1/exp/HT4960-FV8/campos_finales.csv     # opcional (saltea canaritos)
~/buckets/b1/exp/HT4960-FV8/filtro_canaritos_etapa1.csv   # opcional
~/buckets/b1/exp/HT4960-FV8/filtro_canaritos_etapa2.csv   # opcional
```

### 3. Lanzar

```bash
cd ~/buckets/b1/datasets
nohup bash -c '
  jupyter nbconvert --to notebook --execute FV8.ipynb \
    --output FV8_ejecutada.ipynb \
    --ExecutePreprocessor.timeout=-1 \
    --ExecutePreprocessor.kernel_name=ir ;
  sudo shutdown -h now
' >> ~/buckets/b1/corrida_FV8.log 2>&1 &
```

- El avance se sigue en `~/buckets/b1/exp/HT4960-FV8/log_FV8.txt` (cada paso con hora y RAM usada):
  `tail -f ~/buckets/b1/exp/HT4960-FV8/log_FV8.txt`
- **Checkpoints:** si la VM se corta, se relanza el mismo comando y retoma desde la última etapa guardada.
- Al terminar, la VM se apaga sola.


## Qué hace (pipeline)

1. **Clase ternaria** a partir del dataset crudo (BAJA+1, BAJA+2, CONTINUA). `clase01` = BAJA+1 ∪ BAJA+2.
2. **Correcciones:** se eliminan `cprestamos_personales` y `mprestamos_personales`; `ccajas_depositos` de 202105 pasa a NA (dato roto).
3. **Percentil por mes** para todas las variables originales (`_pct`): 0 → 0; positivos repartidos en (0, 1]; negativos en [−1, 0); NA → NA. Quedan en crudo 12 variables categóricas/estables (`cliente_vip`, `cliente_edad`, `cliente_antiguedad`, `cproductos`, `ctarjeta_visa`, `ctarjeta_master`, `Master/Visa_delinquency`, `_status`, `_Finiciomora`).
4. **Historia:** lags 1–3, diferencias 1–3 y acumulado (t − primer mes) sobre los percentiles y las 12 excepciones. 
5. **Tendencia y promedio:** tendencia (`_tend3`, pendiente por cuadrados mínimos) y promedio (`_avg3`) de 3 meses.
6. **Ronda 1 de interacciones:** todas contra todas entre las ~150 originales (A×B, A÷B, B÷A): ~33.500 columnas.
7. **Canarito 1** (selección de variables):
   - 20 % de columnas de ruido uniforme ("canaritos");
   - juez: LightGBM con la mejor receta de la bayesiana, entrenado con **202104 completo**;
   - importancia por **Gain**; corte en *media + 2 desvíos* del puesto de los canaritos;
8. **Ronda 2 de interacciones:** las 20 mejores sobrevivientes (ejes) × las sobrevivientes que no participaron de la ronda 1: 21.870 columnas nuevas, sin repetir pares.
9. **Canarito 2** sobre sobrevivientes + ronda 2 + variables del docente → **5.199 variables finales** (AUC de control 0,9185).
10. **Limpieza cruzada A/B** de 202103–202106: dos mitades estratificadas; entrenado con una mitad predice la otra. Se descartan los CONTINUA con probabilidad > 0,5. Las bajas nunca se tocan.
11. **Ensamble final:** las 5 mejores recetas de `BO_log.txt`, entrenadas con 202103–202106 limpios, sin undersampling; se promedian las probabilidades y se predice 202108.
12. **Entregas:** 15 archivos (8.000 a 15.000 envíos cada 500).


