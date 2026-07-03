# mybatis flex
{docsify-updated}

> https://github.com/mybatis-flex/mybatis-flex

## 条件查询
```
select * from isprint_bind_info where acct = ? and (isprint_device_id = ? OR isprint_device_uid = ?)

QueryWrapper queryWrapper = QueryWrapper.create()
                .where(AccountDeviceInfoEntity::getAcct).eq(deviceInfoVao.getTradeAccount())
                .and((Consumer<QueryWrapper>) q -> q.where(AccountDeviceInfoEntity::getIsprintDeviceId).eq(deviceInfoVao.getAppDeviceId())
                        .or(AccountDeviceInfoEntity::getIsprintDeviceUid).eq(deviceInfoVao.getIsprintDeviceUid()))
                .and(AccountDeviceInfoEntity::getDeleted).eq(false);
```

## JOIN
```
@Table("sys_user")
public class User {
    @Id
    private Integer userId;
    private String userName;
}


@Table("sys_role")
public class Role {
    @Id
    private Integer roleId;
    private String roleKey;
    private String roleName;
}


QueryWrapper queryWrapper = QueryWrapper.create()
        .select(USER.USER_ID, USER.USER_NAME, ROLE.ALL_COLUMNS)
        .from(USER.as("u"))
        .leftJoin(USER_ROLE).as("ur").on(USER_ROLE.USER_ID.eq(USER.USER_ID))
        .leftJoin(ROLE).as("r").on(USER_ROLE.ROLE_ID.eq(ROLE.ROLE_ID));
List<UserVO> userVOS = userMapper.selectListByQueryAs(queryWrapper, UserVO.class);
userVOS.forEach(System.err::println);
```