# Análise de Agrupamento — Canais do YouTube
### Ciência de Dados II — Etapa 3

---

## Visão Geral

Este projeto aplica algoritmos de agrupamento (*clustering*) sobre um dataset de canais do YouTube, com o objetivo de identificar perfis distintos de canais a partir de suas métricas de desempenho. O notebook parte do dataset padronizado e com features selecionadas gerado na Etapa 2 (EDA), reconstruindo o pipeline de PCA antes de aplicar os algoritmos.

---

## Dataset

- **Fonte:** Dataset público de canais do YouTube
- **Amostras:** 995 canais
- **Features utilizadas (13):** `subscribers`, `video views`, `uploads`, `video_views_for_the_last_30_days`, `highest_monthly_earnings`, `earnings_spread`, `subscribers_for_last_30_days`, `Population`, `channel_age`, `views_per_subscriber`, `views_per_upload`, `urban_ratio`, `subscriber_growth_rate`

---

## Estrutura do Repositório

```
.
├── CDII_Parte3.ipynb     # Notebook principal — Etapa 3
├── df_std.csv            # Dataset padronizado (gerado na Etapa 2)
└── README.md
```

---

## Metodologia

### Espaços de dados testados

| Espaço | Dimensões | Variância explicada |
|---|---|---|
| Original padronizado | 13D | 100% |
| PCA reduzido | 6D | 81.5% |

As matrizes de distância (Euclidiana, Manhattan e Cosseno) são pré-computadas uma vez e reutilizadas em todos os algoritmos.

### Métricas de distância
- **Euclidiana** — sensível à magnitude absoluta
- **Manhattan (Cityblock)** — sensível à magnitude, mais robusta a outliers extremos que a Euclidiana
- **Cosseno** — ignora magnitude, compara perfis relativos entre features

### Algoritmos aplicados

| Família | Algoritmo | Métricas aceitas |
|---|---|---|
| Particionais | K-Means | Euclidiana |
| Particionais | Bisecting K-Means | Euclidiana |
| Particionais | K-Medoids (FasterPAM) | Euclidiana, Manhattan, Cosseno |
| Hierárquicos | Ward, Single, Average, Complete | Euclidiana, Manhattan, Cosseno |
| Densidade | DBSCAN | Euclidiana, Manhattan, Cosseno |
| Grade | CLIQUE | — (grade sobre PCA 2D) |

### Métricas de avaliação

- **Silhouette Score** (referência: ≥ 0.25)
- **Davies-Bouldin** (menor = melhor)
- **Calinski-Harabasz** (maior = melhor)
- **ARI / NMI** entre execuções (estabilidade, referência: ARI > 0.8)
- **Coeficiente Cofenético** (qualidade do dendrograma hierárquico)

---

## Principais Resultados

### Seleção de K

- **K=2** é o corte natural e hierarquicamente estável (~83%/17%), mas representa uma divisão grosseira entre canais típicos e canais acima da curva.
- **K=6** é a segmentação mais informativa: isola o grupo de ~10 mega-canais extremos e revela sub-segmentos interpretáveis. Recomendado com **n_init=50** no K-Means para evitar mínimos locais no espaço PCA.

### Melhores resultados por algoritmo (k=6, espaço PCA 6D)

| Algoritmo | Métrica | Silhouette | Davies-Bouldin | Observação |
|---|---|---|---|---|
| K-Means | Euclidiana | 0.377 | 0.903 | Estável com n_init=50 |
| K-Medoids | **Cosseno** | **0.442** | 1.329 | Melhor sil k=6; clusters equilibrados (142–177) |
| Hierárquico Complete | Manhattan | 0.366 | **0.842** | Melhor DB hierárquico sem chaining |
| Hierárquico Average | Cosseno | 0.340 | 1.490 | Clusters equilibrados (88–246) |
| Hierárquico Ward | Euclidiana | 0.339 | 0.925 | Determinístico; sem artefatos |
| DBSCAN | Euclidiana | 0.531 | **0.542** | **Melhor DB geral**; 3 clusters + 13.5% ruído |
| CLIQUE | — (grade 2D) | 0.804 ⚠️ | — | Inflado: avaliado em PCA 2D (45.3% var.) |

### Achados transversais

**1. Outliers confirmados por múltiplos algoritmos:**
Algoritmos com métricas sensíveis à magnitude (Euclidiana e Manhattan) isolaram consistentemente um grupo de **9–12 canais extremos** — K-Means, Bisecting K-Means, Hierárquico Ward e DBSCAN convergiram para o mesmo grupinho de forma independente. Algoritmos com distância cosseno *não* isolaram esse grupo, produzindo clusters equilibrados. Isso revela que esses canais são outliers de **magnitude absoluta**, não de **perfil relativo**.

**2. Escolha da métrica define a natureza da segmentação:**
Cosseno ignora magnitude e enxerga padrões de perfil relativo entre as métricas, produzindo clusters de tamanho equilibrado. Euclidiana/Manhattan são dominadas pela escala absoluta dos dados. A escolha da métrica muda o *que* é considerado "diferente", não apenas a qualidade do agrupamento.

**3. Hierárquico Average: bom dendrograma, mau clustering (com métricas erradas):**
Average obteve o melhor coeficiente cofenético (0.910), mas sofre encadeamento (*chaining*) severo com Euclidiana e Manhattan — produzindo clusters 985/9/1. Com distância **Cosseno**, o Average elimina o chaining e gera clusters equilibrados, evidenciando que o problema é da métrica, não do método.

**4. Ward: melhor clustering, pior dendrograma:**
Coeficiente cofenético de 0.701 (o mais baixo), mas único método hierárquico sem chaining em nenhum threshold. Tensão entre *preservar distâncias* (objetivo do coeficiente cofenético) e *formar clusters úteis* (objetivo de silhouette/DB).

**5. Estabilidade K-Means:**
- k=2: ARI=1.000 em todos os seeds e espaços — perfeitamente estável
- k=6 Original (13D): ARI mín=0.977 — estável
- k=6 PCA (6D) com n_init=10: ARI mín=0.496 — **instável** (mínimo local)
- k=6 PCA (6D) com n_init=50: ARI=1.000 — resolvido

**6. DBSCAN: melhor separação inter-cluster:**
Menor Davies-Bouldin (0.542) de toda a análise. A estrutura baseada em densidade separa naturalmente o blob principal dos outliers sem precisar fixar k, ao custo de classificar 13.5% dos pontos como ruído.

**7. CLIQUE: limitação dimensional:**
Aplicado sobre PCA 2D (45.3% de variância). O silhouette de 0.804 é real nesse espaço mas não comparável aos demais algoritmos avaliados em 6D. Confirmou a presença dos mega-outliers visualmente.

---

## Instalação e Execução

### Requisitos

```bash
pip install numpy pandas matplotlib seaborn scikit-learn scipy kmedoids pyclustering
```

### Executando no Google Colab

1. Faça upload do arquivo `df_std.csv` quando solicitado
2. Execute as células sequencialmente — o notebook reconstrói o pipeline de PCA internamente antes de aplicar os algoritmos

---

## Tecnologias Utilizadas

| Biblioteca | Uso |
|---|---|
| `scikit-learn` | K-Means, Bisecting K-Means, DBSCAN, Agglomerative, PCA, métricas |
| `scipy` | Linkage hierárquico, dendrogramas, coeficiente cofenético, fcluster |
| `kmedoids` | K-Medoids via FasterPAM (matrizes de distância pré-computadas) |
| `pyclustering` | CLIQUE (algoritmo baseado em grade) |
| `pandas` / `numpy` | Manipulação de dados |
| `matplotlib` / `seaborn` | Visualizações |

---

## Autores

Trabalho desenvolvido para a disciplina **Ciência de Dados II**.

## Integrantes

| Nome |
|------|
| Eduardo de Oliveira Araujo |
| Eduardo Oliveira Marson |
| Hugo Alves Viana |
| José Vitor Oliveira Marson |
| Maria Clara Sailva Borges|



---

## Licença

Este projeto é de uso acadêmico.
