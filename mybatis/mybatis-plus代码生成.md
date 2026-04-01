
```
@SpringBootApplication
public class MsgCenterApp {
    public static void main(String[] args) {
         FastAutoGenerator
                .create(new DataSourceConfig.Builder("jdbc:mysql://127.0.0.1:3306/msg0", "root", "")
                        .typeConvertHandler((GlobalConfig globalConfig, TypeRegistry typeRegistry, TableField.MetaInfo metaInfo) -> {
                            int typeCode = metaInfo.getJdbcType().TYPE_CODE;
                            switch (typeCode) {
                                // TODO 需要增加类型处理，尚未补充完整
                                case Types.TINYINT:
                                    return INTEGER;
                            }
                            return typeRegistry.getColumnType(metaInfo);
                        })
                        .keyWordsHandler(new MySqlKeyWordsHandler()))
                .globalConfig(builder -> {
                    builder.author("demo") // 设置作者
//                            .enableSwagger() // 开启 swagger 模式
                            .outputDir("/Users/coder_wang/Workspace/msg-center/msg-center/msg-center-service/src/main/java"); // 指定输出目录
                })
                .packageConfig(builder -> {
                    builder.parent("com.demo.gjyw.repo.dao") // 设置父包名
                            .pathInfo(Collections.singletonMap(OutputFile.xml, "/Users/coder_wang/Workspace/msg-center/msg-center/msg-center-service/src/main/resources/mappers")); // 设置mapperXml生成路径
                })
                .strategyConfig(builder -> {
                    builder.addInclude("message", "user_msg", "group_msg")// 设置需要生成的表名
                            .entityBuilder()
                            .enableFileOverride()
                            .formatFileName("%sEntity");
//                            .addTablePrefix("t_", "c_"); // 设置过滤表前缀
                })
//                .templateEngine(new FreemarkerTemplateEngine()) // 使用Freemarker引擎模板，默认的是Velocity引擎模板
                .execute();
    }
}
```