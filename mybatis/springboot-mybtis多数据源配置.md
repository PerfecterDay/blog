#  Springboot mybatis 多数据源配置


```
@Configuration
@MapperScan(basePackages = "com.demo.demo.repo.dao.mapper.user",sqlSessionFactoryRef = "userSqlSessionFactory",sqlSessionTemplateRef = "userSqlSessionTemplate")
@MapperScan(basePackages = "com.demo.demo.repo.dao.mapper.edp",sqlSessionFactoryRef = "edpSqlSessionFactory",sqlSessionTemplateRef = "edpSqlSessionTemplate")
public class MybatisPlusConfig {

    @Bean("user")
    @ConfigurationProperties(prefix = "spring.datasource.user")
    public DataSource userDatasource(){
        return DataSourceBuilder.create().build();
    }

    @Bean(name = "userDataSourceTransactionManager")
    @Primary
    public DataSourceTransactionManager userDataSourceTransactionManager(@Qualifier("user")DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "userSqlSessionFactory")
    @Primary
    public SqlSessionFactory userSessionFactory(@Qualifier("user")DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource);
        return sessionFactoryBean.getObject();
    }

    @Bean("userSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate userSqlSessionTemplate(@Qualifier("userSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

    @Bean("edp")
    @ConfigurationProperties(prefix = "spring.datasource.edp")
    public DataSource edpDatasource(){
        return DataSourceBuilder.create().build();
    }


    @Bean(name = "edpDataSourceTransactionManager")
    @Primary
    public DataSourceTransactionManager edpDataSourceTransactionManager(@Qualifier("user")DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean(name = "edpSqlSessionFactory")
    @Primary
    public SqlSessionFactory edpSessionFactory(@Qualifier("edp")DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource);
        return sessionFactoryBean.getObject();
    }

    @Bean("edpSqlSessionTemplate")
    @Primary
    public SqlSessionTemplate edpSqlSessionTemplate(@Qualifier("edpSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }

}
```