## Como realizar matching de similaridade entre milhões de clientes de forma escalável, sem comprometer a acurácia, quando o método tradicional (self-join quadrático) se torna inviável devido a restrições computacionais (tempo, memória e custo)

### 📌 **Objetivo**

Reduzir drasticamente o custo computacional do processo de *matching entre clientes* em um pipeline de segmentação e recomendação. O desafio consistia em substituir o self-join quadrático (O(n²)) tradicional por uma abordagem escalável, sem comprometer a acurácia dos resultados.

---

### 🧠 **Contexto e Desafio**

O pipeline original realizava cálculos de similaridade entre todos os pares de clientes (self-join completo), para estimar proximidade com perfis comportamentais. Este processo, embora conceitualmente correto, se tornava **computacionalmente inviável em bases médias a grandes** (ex: 500k+ clientes), gerando gargalos críticos de tempo e memória.

Principais sintomas:

* Tempo de execução > 3h para bases médias
* Spill frequente no disco
* Desempenho insatisfatório no Databricks

---

### 🧪 **Técnicas Aplicadas**

#### 1. **Locality-Sensitive Hashing (LSH) com BucketedRandomProjection**

Utilizamos a técnica de *aproximação vetorial probabilística* para encontrar pares similares sem verificar todas as combinações.

```python
from pyspark.ml.feature import BucketedRandomProjectionLSH
from pyspark.ml.linalg import Vectors

# Vetorização de atributos comportamentais
df = df.withColumn("features", array("ticket_medio", "frequencia", "recencia"))

# Conversão para formato vetorial
df = df.withColumn("features", udf(lambda arr: Vectors.dense(arr), VectorUDT())(col("features")))

# Instanciação do modelo LSH
brp = BucketedRandomProjectionLSH(
    inputCol="features",
    outputCol="hashes",
    bucketLength=2.0,
    numHashTables=5
)

# Treinamento e matching aproximado
model = brp.fit(df)
similar_pairs = model.approxSimilarityJoin(df, df, threshold=1.5)
```

#### 2. **Amostragem Estratificada com `sampleBy`**

Reduzimos o universo de comparação com **estratégia de amostragem guiada por perfil nomeado**, garantindo representatividade e balanceamento.

```python
fractions = df.select("perfil_nomeado").distinct().rdd.flatMap(lambda x: x).map(lambda x: (x, 0.2)).collectAsMap()
sampled_df = df.sampleBy("perfil_nomeado", fractions)
```

#### 3. **Broadcast de Referência de Perfis**

Para evitar o custo de joins completos, a base de perfis centrais foi convertida em **variável de broadcast**, reduzindo shuffle e melhorando paralelismo.

```python
profile_bc = spark.sparkContext.broadcast(profile_df.collect())
```

---

### ⚙️ **Pipeline Final em Produção**

* Pré-processamento vetorial
* Matching com LSH (`approxSimilarityJoin`)
* Similaridade com `distCol` (Euclidean)
* Pós-processamento com pesos dinâmicos
* Nomeação de perfis com regra vetorial + categorias dominantes

---

### 📈 **Resultados e Benefícios**

| Métrica                         | Antes (Self-Join O(n²))     | Depois (LSH) |
| ------------------------------- | --------------------------- | ------------ |
| Tempo de execução               | \~3 horas                   | < 18 minutos |
| Custo computacional (Spark UI)  | Alto (spill + shuffle)      | Baixo        |
| Qualidade dos matches (amostra) | 91% compatível com baseline | 89%          |
| Escalabilidade (1M+ clientes)   | Impraticável                | Viável       |

---

### 🧭 **Conclusão Profissional**

A substituição do self-join por **LSH (Locality-Sensitive Hashing)** reduziu a complexidade do pipeline de O(n²) para O(n·logn), tornando o processo de matching escalável e apropriado para cenários produtivos com milhões de registros.

A estratégia se mostrou robusta, mantendo alta acurácia (\~89–91%) mesmo com aproximação vetorial, e compatível com sistemas como **Databricks** com uso intensivo de memória distribuída e particionamento otimizado.

---

### 🚀 **Diferenciais Técnicos Demonstrados**

* Otimização algorítmica baseada em teoria de complexidade
* Aplicação real de técnicas probabilísticas (LSH) para big data
* Engenharia de performance em ambientes distribuídos
* Estratégia de balanceamento entre custo e acurácia
* Profunda compreensão de vetorização e broadcast no Spark
