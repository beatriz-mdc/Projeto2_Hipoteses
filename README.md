# projeto 2 - teste de hipoteses

# Objetivo:
O projeto tem como objetivo testar hipóteses levantadas por uma plataforma de streaming de música a partir da utilização de ferramentas como BigQuery e Power BI.

# Ferramentas e Tecnologias:
As ferramentas e tecnologias utilizadas foram:
> Google BigQuery;
> Linguagem Python;
> Power BI.

# Processamento e análises:
Os dados foram disponibilizados pela empresa em pasta zipada com 3 planilhas CSV nomeadas “track_in_competition”, “track_in_spotify” e “track_technical_info”. Esses arquivos traziam informações de desempenho de músicas dentro do spotify, bem como em plataformas concorrentes e também informações técnicas sobre cada faixa. 
Foi criado um projeto dentro da ferramenta BigQuery e, em seguida, um dataset onde foram feitos os uploads das tabelas extraídas da pasta zipada.
Após uma breve avaliação inicial, foi decidido realizar algumas verificações sobre as tabelas a fim de localizar dados fora do padrão que pudessem impactar no processo de análise. Primeiro, foram procurados valores nulos. Para localizar esses valores foi utilizada a seguinte query em cada uma das tabelas:

> SELECT 
*
FROM `projeto.dataset.tabela`
WHERE coluna IS NULL;

A partir dessa query foram localizados valores nulos nas tabelas “track_in_competition” e “track_technical_info”.
Na tabela “track_in_competition” os dados nulos estavam na coluna "in_shazam_charts". Essa coluna sinaliza a posição que a música está dentro do chart do shazam. Dessa forma, nossa tratativa foi atribuir o número 0 a esses valores, uma vez que não constam em nenhuma posição do chart.
Na tabela “track_technical_info” os dados nulos estavam na coluna "key". Essa coluna informa qual a nota musical de cada música. Para tratar esses valores nulos, verificamos qual a nota musical que mais aparecia nas demais músicas que continham essa informação na tabela. O resultado foi a nota "C#" como maior ocorrência na tabela. Com isso, utilizamos "C#" para substituir os valores nulos dessa coluna.
A query utilizada foi a seguinte:

CREATE OR REPLACE TABLE `projeto.dataset.tabela` AS
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
FROM `projeto.dataset.tabela`;







