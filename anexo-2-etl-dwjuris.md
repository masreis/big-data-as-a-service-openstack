# Importação de Dados
sqoop \
 list-tables \
 --connect $url \
 --username $user \
 --password $pwd \
 > tables.txt
echo $(tr '\n' ',' < tables.txt ) > tables.txt
time sqoop import-all-tables \
    --connect $url \
    --username $user$ \
    --password $pwd \
    --verbose \
    --direct \
    --num-mappers 1 \
    --compression-codec snappy \
    --exclude-tables $(cat tables.txt) \
    --as-avrodatafile \
    --warehouse-dir $dir
# Data Lake
openstack container list
openstack container create dwjuris
diretorio=dwjuris-avro
hdfs dfs \
    -Dfs.swift.service.sahara.username=admin \
    -Dfs.swift.service.sahara.password=$pwd \
    -du -h swift://dwjuris.sahara/
hdfs dfs \
    -Dfs.swift.service.sahara.username=admin \
    -Dfs.swift.service.sahara.password=$pwd \
    -mkdir -p swift://dwjuris.sahara/$diretorio
time hadoop distcp \
    -Dfs.swift.service.sahara.username=admin \
    -Dfs.swift.service.sahara.password=$pwd \
    -overwrite \
    /user/hdfs/$diretorio \
    swift://dwjuris.sahara/$diretorio/
time hdfs dfs \
    -Dfs.swift.service.sahara.username=admin \
    -Dfs.swift.service.sahara.password=$pwd \
    -cp -f \
    /user/hdfs/$diretorio \
    swift://dwjuris.sahara/$diretorio
time hdfs dfs \
    -Dfs.swift.service.sahara.username=admin \
    -Dfs.swift.service.sahara.password=$pwd \
    -cp -f \
    swift://dwjuris.sahara/$diretorio \
    /user/hadoop/$diretorio
swift list dwjuris -l -p dwjuris-text/
# Copia arquivos Avro
for arquivo in $(hdfs dfs -du -h \
    $diretorio \
    |awk '{print $5}'|awk -F "/" '{print $5}')
    do
    hdfs dfs \
        -Dfs.swift.service.sahara.username=admin \
        -Dfs.swift.service.sahara.password=$pwd \
        -mkdir -p swift://dwjuris.sahara/dwjuris-avro/$arquivo

    time hdfs dfs \
        -Dfs.swift.service.sahara.username=admin \
        -Dfs.swift.service.sahara.password=$pwd \
        -cp -f \
        $dir$arquivo \
        swift://dwjuris.sahara/dwjuris-avro/
done
