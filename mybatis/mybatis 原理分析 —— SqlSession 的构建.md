
# mybatis 原理分析(2) —— SqlSession 的构建
{docsify-updated}




## SqlSessionFactoryBuilder 构造 SqlSessionFactory 过程
在mybatis入门里介绍过，每个基于 MyBatis 的应用都是以一个 SqlSessionFactory 的实例为中心的。 SqlSessionFactory 的实例可以通过 SqlSessionFactoryBuilder 获得。

    public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
        try {
          XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
          return build(parser.parse());
        } catch (Exception e) {
          throw ExceptionFactory.wrapException("Error building SqlSession.", e);
        } finally {
          ErrorContext.instance().reset();
          try {
            reader.close();
          } catch (IOException e) {
            // Intentionally ignore. Prefer previous error.
          }
        }
      }

      public SqlSessionFactory build(Configuration config) {
          return new DefaultSqlSessionFactory(config);
        }

看上述 SqlSessionFactoryBuilder 源码， build 方法首先 new 一个 XMLConfigBuilder 对象 parser ，然后使用它的 parser.parse 方法构造一个 Configuration 对象，并调用 build 方法构造 SqlSessionFactory 实例。


## SqlSessionFactory
SqlSessionFactory 类本身只有一个 Configuration 类的域， SqlSessionFactory  使用 Configuration 中的信息构建 SqlSession。

    public SqlSession openSession(ExecutorType execType, boolean autoCommit) {
        return openSessionFromDataSource(execType, null, autoCommit);
      }

    private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
        Transaction tx = null;
        try {
          final Environment environment = configuration.getEnvironment();
          final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
          tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
          final Executor  executor = configuration.newExecutor(tx, execType);
          return new DefaultSqlSession(configuration, executor, autoCommit);
        } catch (Exception e) {
          closeTransaction(tx); // may have fetched a connection so lets call close()
          throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
        } finally {
          ErrorContext.instance().reset();
        }
      }
从上述源码中可知， SqlSessionFactory 在构建 SqlSession 的过程中， 借助 Configuration 对象的信息构建了 TransactionFactory 、 Executor ，并使用这些信息构建了一个 DefaultSqlSession 对象实例。