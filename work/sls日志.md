+ `/cap | SELECT * FROM log WHERE cast(duration AS bigint) > 5000`
+ `* | SELECT count(1) FROM log WHERE cast(status AS bigint) = 748`
+ `* | SELECT "x-request-id",url,duration,time FROM log WHERE cast(status AS bigint) = 748 order by time desc`
+ `/cap | SELECT "x-request-id",url,duration,time FROM log WHERE cast(duration AS bigint) > 5000 order by time desc`

e6241959-929d-4815-bcc1-201acac26ab0