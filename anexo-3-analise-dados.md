# Análise de dados com Spark
import com.databricks.spark.avro._
import com.databricks.spark.csv._

val movimentacao_av="/user/hadoop/dwjuris-avro/TB_FATO_MOVIMENTACAO"
val df_mov_av = spark.read.avro(movimentacao_av)
val movimentacao_av_sw="swift://dwjuris.sahara/dwjuris-avro/TB_FATO_MOVIMENTACAO/"
val df_mov_av_sw = spark.read.avro(movimentacao_av_sw)

df_mov_av_sw.limit(1000000).write.avro("swift://dwjuris.sahara/dwjuris-avro/TB_MOV_1M")
df_mov_av.groupBy($"codigomovimento").count.show
df_mov_av_sw.groupBy($"codigomovimento").count.show

val processo_av="/user/hadoop/dwjuris-avro/TB_DIM_PROCESSO"
val processo_av_sw="swift://dwjuris.sahara/dwjuris-avro/TB_DIM_PROCESSO/"
val df_processo_av_sw = spark.read.avro(processo_av_sw)
val df_mov_av = spark.read.avro(movimentacao_av)

df_mov_av.groupBy($"classeprocessual").count.show

