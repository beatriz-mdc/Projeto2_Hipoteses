# Etapas do projeto

1. Processamento e preparação da base de dados
Os dados foram disponibilizados pela Laboratoria em pasta zipada com 3 planilhas CSV nomeadas “track_in_competition”, “track_in_spotify” e “track_technical_info”. 
Esses arquivos contém informações de desempenho de músicas dentro do spotify, bem como em plataformas concorrentes e também informações técnicas sobre cada faixa. 
Foi criado um projeto dentro da ferramenta BigQuery nomeada "projeto 2" e, em seguida, um dataset nomeado "musica_streaming" onde foram feitos os uploads das tabelas extraídas da pasta zipada.

**Identificar e tratar valores nulos**
Para localizar valores nulos foi utilizada a seguinte query em cada uma das tabelas:

```sql
SELECT 
* 
FROM projeto.dataset.tabela 
WHERE coluna IS NULL;
```

A partir dessa query foram localizados valores nulos nas tabelas “track_in_competition” e “track_technical_info”. Na tabela “track_in_competition” os dados nulos estavam na coluna "in_shazam_charts". Essa coluna sinaliza a posição que a música está dentro do chart do shazam. Dessa forma, nossa tratativa foi atribuir o número 0 a esses valores, uma vez que não constam em nenhuma posição do chart. Na tabela “track_technical_info” os dados nulos estavam na coluna "key". Essa coluna informa qual a nota musical de cada música. Para tratar esses valores nulos, verificamos qual a nota musical que mais aparecia nas demais músicas que continham essa informação na tabela. O resultado foi a nota "C#" como maior ocorrência na tabela. Com isso, utilizamos "C#" para substituir os valores nulos dessa coluna. A query utilizada foi a seguinte:

```sql
CREATE OR REPLACE TABLE `projeto-2-456519.musica_streaming.tracks_concorrentes` AS
SELECT
track_id,
bpm,
COALESCE(key, 'C#') AS key,
mode,
`danceability_%`,
`valence_%`,
`energy_%`,
`acousticness_%`,
`instrumentalness_%`,
`liveness_%`,
`speechiness_%`
FROM `projeto-2-456519.musica_streaming.tracks_concorrentes`;
```

*Identificar e tratar valores duplicados*
Esses valores duplicados foram encontrados na tabela “track_in_spotify” com a seguinte query:

```sql
SELECT 
track_name,
artists_name,
COUNT (*) AS Duplicatas
 FROM `projeto-2-456519.musica_streaming.tracks_spotify`
 GROUP BY track_name, artists_name
 HAVING COUNT (*) >1
```

Foram encontradas 4 músicas com duplicatas. A partir de uma análise das outras variáveis, foi verificado um padrão em 3 músicas, onde essas não apresentavam a informação de chart. Com isso, foi decidido excluir essas canções com chart zero.
Dentre as músicas encontradas, apenas 1 apresentava chart zerado para ambas as duplicatas. Dessa forma, foi decidido excluir a que tinha menor número de streams.
A query utilizada para exclusão:

```sql
DELETE FROM `projeto-2-456519.musica_streaming.tracks_spotify` 
WHERE track_id IN ('8173823', '3814670', '7173596', '1119309');
```

*Identificar e tratar dados fora do escopo de análise*
Todas as variáveis das tabelas foram mantidas nessa etapa a fim de evitar possíveis retrabalhos que pudessem afetar a análise futura.

*Identificar e tratar dados discrepantes em variáveis ​​categóricas*
Nesta etapa foram encontrados dados discrepantes em caracteres especiais no nome de artistas e de músicas.
Para correção desses dados foi utilizada a seguinte query:

```sql
SELECT  
  * EXCEPT (track_name, artists_name),
  REGEXP_REPLACE(track_name, r'[^a-zA-Z0-9À-ÿ (),\-\'"&]', '') AS track_corrigida,
  REGEXP_REPLACE(artists_name, r'[^a-zA-Z0-9À-ÿ (),\-\'"&]', '') AS artista_corrigida
FROM `projeto-2-456519.musica_streaming.tracks_spotify`;
```

Após a correção foram identicados 2 músicas com o campo "track_corrigida" vazio. Após análise, foi decidido manter as músicas na base, pois não teria impacto negativo na análise, uma vez que elas possuem os demais dados completos.

*Identificar e tratar dados discrepantes em variáveis ​​numéricas*
Foram calculados o valor máximo, mínimo e média para cada variável numérica.
Abaixo exemplo da query utilizada na tabela "tracks_sportify"

```sql
SELECT  
MAX(coluna) as maximo,
MIN(coluna) as minimo,
AVG (coluna) as media
FROM `projeto-2-456519.musica_streaming.tracks_spotify`;
```

A query acima foi aplicada a todas variáveis númericas de cada tabela.
Com ela identificamos a coluna "streams" classficada como STRING e uma variável fora do padrão dentro da mesma coluna.

*Verificar e alterar o tipo de dados*
A coluna "streams" da tabela "tracks_spotify" foi corrigida de STRING para INTEGER, uma vez que se trata de uma variável numérica.
Abaixo a query utilizada:

```sql
SELECT 
*, 
SAFE_CAST(streams AS INT64) AS streams_corrigido
FROM `projeto-2-456519.musica_streaming.tracks_spotify`;
```

Para a variável fora do padrão da coluna "streams" decidimos manter a música e tornar o campo nulo.
Para substituir o campo nulo, analisamos as músicas lançadas no mesmo ano e identificamos apenas 1. Com essa música, calculamos a média de número de streams por playlists.
Pegamos esse valor e multiplicamos pelo total de playlists que a música com o campo nulo tinha e assim substituímos o campo nulo com esse valor.
Abaixo a query utilizada:

```sql
SELECT  
* EXCEPT (streams_corrigido),
CASE WHEN streams_corrigido IS NULL THEN 394968158
ELSE streams_corrigido
END AS streams_limpo
FROM `projeto-2-456519.musica_streaming.tracks_spotify`;
```

*Criar novas variáveis*
Nesta etapa foram criadas as variáveis "data de lançamento", "total de playlists" e "charts concorrentes".
Abaixo as respectivas queries:

```sql
SELECT 
*,
DATE(CONCAT(released_year, "-", released_month, "-", released_day)) AS data_lancamento
FROM `projeto-2-456519.musica_streaming.tracks_spotify`;
```

```sql
SELECT 
*,
in_apple_playlists + in_deezer_playlists + in_spotify_playlists as total_playlists
FROM `projeto-2-456519.musica_streaming.tabelas_unificadas`;
```

```sql
SELECT
*,
ROUND((in_apple_charts + in_deezer_charts + in_shazam_charts)/3) as charts_concorrentes
FROM `projeto-2-456519.musica_streaming.tabelas_unificadas`
```

*Unir tabelas*
Após a limpeza de dados e criação de novas variáveis, unimos as 3 tabelas na query abaixo:

```sql
CREATE OR REPLACE TABLE `projeto-2-456519.musica_streaming.tabelas_unificadas` AS
SELECT
  a.*,
  b.in_apple_charts,
  b.in_apple_playlists,
  b.in_deezer_charts,
  b.in_deezer_playlists,
  b.in_shazam_charts,
  c.`acousticness_%`,
  c.bpm,
  c.`danceability_%`,
  c.`energy_%`,
  c.`instrumentalness_%`,
  c.key,
  c.`liveness_%`,
  c.mode,
  c.`speechiness_%`,
  c.`valence_%`
FROM `projeto-2-456519.musica_streaming.tracks_spotify` a
LEFT JOIN `projeto-2-456519.musica_streaming.tracks_concorrentes` b ON a.track_id = b.track_id
LEFT JOIN `projeto-2-456519.musica_streaming.tracks_info` c ON a.track_id = c.track_id;
```

*Construir tabelas auxiliares*
Foi criada uma tabela auxiliar com WITH para unir a quantidade de músicas e total de streams para cada artista.

```sql
CREATE OR REPLACE TABLE `projeto-2-456519.musica_streaming.artistas_streams_musicas` AS
WITH streams_artistas AS (
  SELECT 
    artista_corrigida,
    SUM(streams_limpo) AS total_streams
  FROM `projeto-2-456519.musica_streaming.tabelas_unificadas`
  GROUP BY artista_corrigida
),

musicas_unicas AS (
  SELECT 
    artista_corrigida,
    COUNT(*) AS quantidade_musica
  FROM `projeto-2-456519.musica_streaming.tabelas_unificadas`
  GROUP BY artista_corrigida
)

SELECT 
  a.artista_corrigida,
  s.total_streams,
  m.quantidade_musica
FROM `projeto-2-456519.musica_streaming.tabelas_unificadas` AS a
JOIN streams_artistas AS s ON a.artista_corrigida = s.artista_corrigida
JOIN musicas_unicas AS m ON a.artista_corrigida = m.artista_corrigida
GROUP BY a.artista_corrigida, s.total_streams, m.quantidade_musica;
```

*2. Análise Exploratória*
A partir desta etapa começamos a visualizar e explorar os dados em profundidade usando Power BI.

*Agrupamento e visualizar variáveis ​​categóricas*
Relacionamos número de músicas por artista e número de músicas por ano de lançamento através de tabela matricial e gráfico de barras.

[inserir print tabela e gráfico]

*Aplicar medidas de tendência central*
Aplicadas de medidas como média, mediana e desvio padrão para as variáveis de streams e total de playlists.

[inserir print tabela e gráfico]

A média é maior que a mediana, o que indica que alguns valores muito altos (outliers) estão puxando a média para cima.

*Ver distribuição*
Os valores de média e mediana encontrados no tópico anterior indicam uma distribuição dos dados assimétrica à direita.
É possível visualizar essa distribuição através dos histogramas abaixo criados a partir de python:

[inserir print histogramas]

import matplotlib.pyplot as plt
import pandas as pd
# Obtenha os dados do Power BI - você só preciso alterar essas informações de todo o code
data = dataset[['YOUR VARIABLE']]
# Crie o histogram
plt.hist(data, bins=10, color='blue', alpha=0.7)
plt.xlabel('Value')
plt.ylabel('Frequency')
plt .title('Histogram')
# Mostre o histogram
plt.show()

*Aplicar medidas de dispersão*
Os valores de desvio padrão que encontramos são maiores que a média, o que indica que os dados das variáveis analisadas estão bem espalhados, há muita variação nos dados.

*Visualizar o comportamento dos dados ao longo do tempo*
Aqui visualizamos o comportamento do número de streams e do número de músicas ao longo dos anos.

[inserir print do gráfico]

*Calcular quartis, decis ou percentis*
Calculamos os quartis para identificar onde está a maior parte dos dados, se estão mais próximos do início, do meio ou do fim da distribuição.
Na query abaixo calculamos os quartis de cada variável e criamos suas respectivas classificações:

```sql
WITH quartis AS (
  SELECT
    track_id,
    NTILE(4) OVER (ORDER BY total_playlists) AS playlist_quartil,
    NTILE(4) OVER (ORDER BY bpm) AS bpm_quartil,
    NTILE(4) OVER (ORDER BY `danceability_%`) AS danceability_quartil,
    NTILE(4) OVER (ORDER BY `valence_%`) AS valence_quartil,
    NTILE(4) OVER (ORDER BY `energy_%`) AS energy_quartil,
    NTILE(4) OVER (ORDER BY `acousticness_%`) AS acousticness_quartil,
    NTILE(4) OVER (ORDER BY `instrumentalness_%`) AS instrumentalness_quartil,
    NTILE(4) OVER (ORDER BY `liveness_%`) AS liveness_quartil,
    NTILE(4) OVER (ORDER BY `speechiness_%`) AS speechiness_quartil
  FROM
    `projeto-2-456519.musica_streaming.tabelas_unificadas`
)
SELECT
  a.*,
  q.playlist_quartil,
  IF(q.playlist_quartil = 4, "Alto", IF(q.playlist_quartil = 3, "Médio", IF(q.playlist_quartil = 2, "Médio", "Baixo"))) AS playlist_categorizada,
  q.bpm_quartil,
  IF(q.bpm_quartil = 4, "Alto", IF(q.bpm_quartil = 3, "Médio", IF(q.bpm_quartil = 2, "Médio", "Baixo"))) AS bpm_categorizada,
  q.danceability_quartil,
  IF(q.danceability_quartil = 4, "Alto", IF(q.danceability_quartil = 3, "Médio", IF(q.danceability_quartil = 2, "Médio", "Baixo"))) AS danceability_categorizada,
  q.valence_quartil,
  IF(q.valence_quartil = 4, "Alto", IF(q.valence_quartil = 3, "Médio", IF(q.valence_quartil = 2, "Médio", "Baixo"))) AS valence_categorizada,
  q.energy_quartil,
  IF(q.energy_quartil = 4, "Alto", IF(q.energy_quartil = 3, "Médio", IF(q.energy_quartil = 2, "Médio", "Baixo"))) AS energy_categorizada,
  q.acousticness_quartil,
  IF(q.acousticness_quartil = 4, "Alto", IF(q.acousticness_quartil = 3, "Médio", IF(q.acousticness_quartil = 2, "Médio", "Baixo"))) AS acousticness_categorizada,
  q.instrumentalness_quartil,
  IF(q.instrumentalness_quartil = 4, "Alto", IF(q.instrumentalness_quartil = 3, "Médio", IF(q.instrumentalness_quartil = 2, "Médio", "Baixo"))) AS instrumentalness_categorizada,
  q.liveness_quartil,
  IF(q.liveness_quartil = 4, "Alto", IF(q.liveness_quartil = 3, "Médio", IF(q.liveness_quartil = 2, "Médio", "Baixo"))) AS liveness_categorizada,
  q.speechiness_quartil,
  IF(q.speechiness_quartil = 4, "Alto", IF(q.speechiness_quartil = 3, "Médio", IF(q.speechiness_quartil = 2, "Médio", "Baixo"))) AS speechiness_categorizada
FROM
  `projeto-2-456519.musica_streaming.tabelas_unificadas` a
JOIN
  quartis q ON a.track_id = q.track_id
```

*Calcular correlação entre variáveis*
As queries abaixo correspondem às correlações feitas para testar as hipóteses do projeto.
Correlação próxima a 1 indica que à medida que uma variável aumenta, a outra variável também aumenta.
Correlação próxima a -1 indica que à medida que uma variável aumenta, a outra variável diminui.
Correlação próxima a 0 indica uma ausência de correlação linear. Não há relação linear entre as duas variáveis.

```sql
SELECT  
CORR (bpm, streams_limpo) as correlacao_bpm,
CORR (in_spotify_charts, charts_concorrentes) as correlacao_charts,
CORR (total_playlists, streams_limpo) as correlacao_playlists
FROM `projeto-2-456519.musica_streaming.tabelas_unificadas`
```

[inserir print do resultado]

```sql
SELECT  
CORR (Valor, streams_limpo) as correlacao
FROM `projeto-2-456519.musica_streaming.tabela_auxiliar`;
```

[inserir print do resultado]

```sql
SELECT    
CORR (quantidade_musica, total_streams) as correlacao_musicas
FROM `projeto-2-456519.musica_streaming.artistas_streams_musicas`;
```

[inserir print do resultado]

*3. Análise*
Após a exploração de dados é possível realizar os testes das hipóteses levantadas no projeto.

*Aplicar segmentação*
Foi criada uma tabela agrupando os dados entre as categorias criadas para cada característica da música em relação ao número médio de streams por categoria.

[inserir print das tabelas]

*Validar hipótese*
Aqui validaremos ou não as hipóteses levantadas com gráficos de dispersão a partir das correlações calculadas.

> Músicas com BPM (Batidas Por Minuto) mais altos fazem mais sucesso em termos de streams no Spotify.

[inserir print do gráfico]

A hipótese não existe.
O gráfico aponta para uma ausência de correlação entre as músicas com BPM mais alto e o sucesso em termos de streams no Spotify.
Essa falta de relação pode indicar que outros fatores, que não o BPM alto, influenciam na popularidade de streams de uma música.

> As músicas mais populares no ranking do Spotify também possuem um comportamento semelhante em outras plataformas como Deezer.

[inserir print do gráfico]

A hipótese é verdadeira.
O gráfico aponta para uma correlação positiva entre as músicas melhores rankeadas no Spotify em comparação às demais plataformas.
Isso indica que uma música bem rankeada dentro dos charts do Spotify tem chances de apresentar o mesmo comportamento nos charts dos demais concorrentes.

> A presença de uma música em um maior número de playlists é relacionada a um maior número de streams.

[inserir print do gráfico]

A hipótese é válida.
O gráfico indica uma forte correlação positiva entre as variáveis avaliadas.
O resultado mostra que a maior participação em playlists leva a um maior número de streams para a música, sendo assim importante um bom trabalho relacionado a curadoria de playlists.

> Artistas com maior número de músicas no Spotify têm mais streams.

[inserir print do gráfico]

A hipótese é correta.
Nos dados é possível observar que os artistas que possuem o maior número de músicas também possuem o maior somatório no número de streams.
O dado indica que um artista com maior portfólio pode acumular sim um maior número de streams.

> As características da música influenciam no sucesso em termos de streams no Spotify.

[inserir print do gráfico]

Não necessariamente.
No gráfico vemos uma correlação negativa entre as variáveis, indicando que o número de streams diminui conforme as variáveis aumentam.
Vimos até aqui que diversos fatores podem influenciar no sucesso de streams, as características da música não parecem ser uma delas.
