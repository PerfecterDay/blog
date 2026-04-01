# mybatis flex
{docsify-updated}

> https://github.com/mybatis-flex/mybatis-flex


```
select * from isprint_bind_info where acct = ? and (isprint_device_id = ? OR isprint_device_uid = ?)

QueryWrapper queryWrapper = QueryWrapper.create()
                .where(AccountDeviceInfoEntity::getAcct).eq(deviceInfoVao.getTradeAccount())
                .and((Consumer<QueryWrapper>) q -> q.where(AccountDeviceInfoEntity::getIsprintDeviceId).eq(deviceInfoVao.getAppDeviceId())
                        .or(AccountDeviceInfoEntity::getIsprintDeviceUid).eq(deviceInfoVao.getIsprintDeviceUid()))
                .and(AccountDeviceInfoEntity::getDeleted).eq(false);
```