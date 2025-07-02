# 🌪️ Projeto de Big-Data: Análise de Constrained-Off Eólico

## 📋 Descrição do Projeto

Este projeto implementa uma solução completa de big-data para análise de constrained-off (restrições de geração) em usinas eólicas brasileiras, utilizando dados consolidados do ONS (Operador Nacional do Sistema Elétrico).

### 🎯 Objetivos

1. **Processamento** dos dados consolidados via lotes
2. **Agregações temporais e espaciais** com conversão para formato Parquet
3. **Detecção de padrões e anomalias** nos eventos de constrained-off
4. **Visualização interativa** das análises
5. **Avaliação da arquitetura** proposta

## 📊 Estrutura dos Dados

### Dados de Entrada (Consolidados)
- **Arquivo Principal**: `restricoes_coff_eolicas_consolidado.csv`
  - Geração, disponibilidade, restrições por usina
  - Dados consolidados de todos os meses
  - Granularidade: 30 minutos

- **Arquivo de Detalhamento**: `restricoes_coff_eolicas_detalhamento_consolidado.csv`
  - Dados de vento, geração estimada vs verificada
  - Informações por unidade geradora

### Período de Análise
- **Início**: Outubro/2021
- **Fim**: Abril/2025
- **Total**: 42 meses de dados
- **Volume**: Dados consolidados de constrained-off eólico

## 🏗️ Arquitetura da Solução

```
Consolidated CSV Files → Parquet Processing → Temporal/Spatial Aggregations → Anomaly Detection → Visualization
```

### Componentes Principais

1. **Pipeline de Processamento** (`big_data_pipeline.py`)
   - Carregamento de dados consolidados
   - Limpeza e normalização
   - Conversão para Parquet

2. **Detecção de Anomalias** (`anomaly_detection_analysis.py`)
   - Análise de constrained-off extremo
   - Variações bruscas na geração
   - Padrões temporais anômalos
   - Clusters espaciais

3. **Dashboard Interativo** (`visualization_dashboard.py`)
   - Visualizações em tempo real
   - Filtros dinâmicos
   - Relatórios automáticos

## 🔍 Tipos de Anomalias Detectadas

### 1. Constrained-Off Extremo
- **Critério**: >70% de restrição média
- **Detecção**: Usinas com perda significativa de geração
- **Impacto**: Perda de energia renovável

### 2. Variações Bruscas na Geração
- **Critério**: Z-score >3 na geração
- **Detecção**: Instabilidade na produção
- **Impacto**: Instabilidade na rede

### 3. Padrões Temporais Anômalos
- **Critério**: Horários com >1.5x a média de constrained-off
- **Detecção**: Padrões recorrentes de restrição
- **Impacto**: Ineficiência operacional

### 4. Clusters Espaciais
- **Critério**: Estados com >3 usinas e >30% constrained-off
- **Detecção**: Problemas regionais de transmissão
- **Impacto**: Limitações de infraestrutura

## 🚀 Como Executar

### 1. Instalação das Dependências

```bash
# Ativar ambiente virtual
source venv/bin/activate  # Linux/Mac
# ou
venv\Scripts\activate     # Windows

# Instalar dependências
pip install -r requirements.txt
```

### 2. Verificar Dados Consolidados

Certifique-se de que os arquivos consolidados existem:
- `restricoes_coff_eolicas_consolidado.csv`
- `restricoes_coff_eolicas_detalhamento_consolidado.csv`

Se não existirem, execute primeiro:
```bash
python transform.py
```

### 3. Execução do Pipeline Completo

```bash
# Executar pipeline completo
python run_pipeline.py

# Ou executar passos individuais
python run_pipeline.py --steps 2 3  # Apenas pipeline e anomalias
```

### 4. Execução Individual dos Componentes

```bash
# Apenas pipeline de big-data
python big_data_pipeline.py

# Apenas análise de anomalias
python anomaly_detection_analysis.py

# Apenas dashboard
streamlit run visualization_dashboard.py
```

## 📁 Estrutura de Arquivos

```
data-ons/
├── restricoes_coff_eolicas_consolidado.csv           # Dados principais consolidados
├── restricoes_coff_eolicas_detalhamento_consolidado.csv # Dados de detalhamento
├── processed_data/                                   # Dados processados
│   ├── main_batch_*.parquet                         # Dados em lotes
│   ├── temporal_summary.parquet                     # Agregações temporais
│   ├── state_summary.parquet                        # Agregações por estado
│   ├── top_usinas_constrained.parquet               # Top usinas
│   └── *_analysis.parquet                           # Análises de anomalias
├── big_data_pipeline.py                             # Pipeline principal
├── anomaly_detection_analysis.py                    # Análise de anomalias
├── visualization_dashboard.py                       # Dashboard Streamlit
├── run_pipeline.py                                  # Script principal
├── transform.py                                     # Consolidação básica
├── requirements.txt                                 # Dependências
└── README.md                                        # Este arquivo
```

## 📈 Queries SQL para Detecção de Anomalias

### 1. Constrained-Off Extremo
```sql
SELECT 
    nom_usina, nom_estado, ano, mes,
    AVG(percentual_constrained) as percentual_medio,
    MAX(percentual_constrained) as percentual_max,
    COUNT(*) as registros
FROM wind_data
WHERE percentual_constrained > 50
GROUP BY nom_usina, nom_estado, ano, mes
HAVING AVG(percentual_constrained) > 70
ORDER BY percentual_medio DESC
```

### 2. Variações Bruscas na Geração
```sql
WITH geracao_stats AS (
    SELECT 
        nom_usina, nom_estado, ano, mes,
        AVG(val_geracao) as geracao_media,
        STDDEV(val_geracao) as geracao_std
    FROM wind_data
    GROUP BY nom_usina, nom_estado, ano, mes
)
SELECT 
    w.nom_usina, w.nom_estado, w.din_instante,
    w.val_geracao, gs.geracao_media, gs.geracao_std,
    ABS(w.val_geracao - gs.geracao_media) / gs.geracao_std as z_score
FROM wind_data w
JOIN geracao_stats gs ON w.nom_usina = gs.nom_usina 
    AND w.ano = gs.ano AND w.mes = gs.mes
WHERE ABS(w.val_geracao - gs.geracao_media) / gs.geracao_std > 3
ORDER BY z_score DESC
```

### 3. Padrões Temporais
```sql
SELECT 
    nom_usina, nom_estado, hora,
    AVG(percentual_constrained) as percentual_medio_hora,
    COUNT(*) as registros_hora
FROM wind_data
GROUP BY nom_usina, nom_estado, hora
HAVING AVG(percentual_constrained) > (
    SELECT AVG(percentual_constrained) * 1.5 
    FROM wind_data 
    WHERE nom_usina = wind_data.nom_usina
)
ORDER BY percentual_medio_hora DESC
```

### 4. Clusters Espaciais
```sql
WITH state_anomalies AS (
    SELECT 
        nom_estado, ano, mes,
        AVG(percentual_constrained) as percentual_estado,
        COUNT(DISTINCT nom_usina) as num_usinas_afetadas
    FROM wind_data
    WHERE percentual_constrained > 30
    GROUP BY nom_estado, ano, mes
)
SELECT 
    nom_estado, ano, mes, percentual_estado, num_usinas_afetadas,
    CASE 
        WHEN percentual_estado > 50 AND num_usinas_afetadas > 5 THEN 'CLUSTER_CRITICO'
        WHEN percentual_estado > 30 AND num_usinas_afetadas > 3 THEN 'CLUSTER_MODERADO'
        ELSE 'ISOLADO'
    END as tipo_cluster
FROM state_anomalies
WHERE percentual_estado > 30
ORDER BY percentual_estado DESC, num_usinas_afetadas DESC
```

## 🎯 Oportunidades de Análise

### 1. Predição de Constrained-Off
- **Dados meteorológicos**: Integração com previsões de vento
- **Machine Learning**: Modelos preditivos baseados em histórico
- **Otimização**: Redução de restrições via operação

### 2. Otimização Operacional
- **Despacho inteligente**: Redistribuição de geração
- **Alertas automáticos**: Detecção em tempo real
- **Planejamento**: Expansão da rede de transmissão

### 3. Análise de Impacto
- **Econômico**: Custos das restrições
- **Ambiental**: Perda de energia renovável
- **Técnico**: Estabilidade da rede

## 📊 Métricas de Sucesso

- **Redução de 20%** no constrained-off total
- **Detecção de anomalias** em <5 minutos
- **Precisão de predição** >80%
- **ROI positivo** em 12 meses

## 🔧 Tecnologias Utilizadas

- **Python**: Linguagem principal
- **Pandas**: Manipulação de dados
- **NumPy**: Computação numérica
- **Plotly**: Visualizações interativas
- **Streamlit**: Dashboard web
- **Parquet**: Formato de armazenamento otimizado
- **SQL**: Queries para detecção de anomalias

## 🚀 Próximos Passos

### Curto Prazo (1-3 meses)
1. Implementar monitoramento em tempo real
2. Criar alertas automáticos para anomalias
3. Otimizar despacho das usinas mais afetadas

### Médio Prazo (3-12 meses)
1. Desenvolver modelos preditivos de constrained-off
2. Implementar otimização automática do despacho
3. Investir em infraestrutura de transmissão crítica

### Longo Prazo (1-3 anos)
1. Integrar dados meteorológicos para predição
2. Implementar machine learning para otimização
3. Desenvolver estratégias de armazenamento de energia

## 📞 Suporte

Para dúvidas ou sugestões sobre o projeto:

1. Verifique a documentação dos arquivos
2. Execute os testes de exemplo
3. Consulte os logs de execução
4. Entre em contato com a equipe de desenvolvimento

---

**Desenvolvido para análise de big-data em energia eólica** 🌪️⚡ 