# 集成 vault
{docsify-updated}

> https://cloud.spring.io/spring-cloud-vault/reference/html/

## 自定义 Environment
SpringApplication 有 `ApplicationListener` 和 `ApplicationContextInitializer` 实现，用于将自定义应用到上下文或环境中。Spring Boot 从 `META-INF/spring.factories` 中加载了大量此类自定义，供内部使用。注册其他自定义的方法不止一种：
1. 在运行 SpringApplication 之前，通过调用 SpringApplication 上的 `addListeners` 和 `addInitializers` 方法，以编程方式为每个应用程序注册。
2. 声明式方法：为所有应用程序添加 `META-INF/spring.factories` ，并打包一个 jar 文件，供所有应用程序作为库使用。

SpringApplication 会向监听器发送一些特殊的 `ApplicationEvents` （有些甚至在上下文创建之前），然后也会为 `ApplicationContext` 发布的事件注册监听器。

还可以使用 `EnvironmentPostProcessor` 在应用上下文刷新之前自定义环境。每个实现都应在 `META-INF/spring.factories` 中注册，如下例所示：
```
org.springframework.boot.env.EnvironmentPostProcessor=com.example.YourEnvironmentPostProcessor
```

`YourEnvironmentPostProcessor` 实现可以加载任意文件并将其添加到 `Environment` 中。例如，下面的示例从 classpath 加载了一个 YAML 配置文件：
```
import java.io.IOException;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.env.EnvironmentPostProcessor;
import org.springframework.boot.env.YamlPropertySourceLoader;
import org.springframework.core.env.ConfigurableEnvironment;
import org.springframework.core.env.PropertySource;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;
import org.springframework.util.Assert;

public class MyEnvironmentPostProcessor implements EnvironmentPostProcessor {

	private final YamlPropertySourceLoader loader = new YamlPropertySourceLoader();

	@Override
	public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
		Resource path = new ClassPathResource("com/example/myapp/config.yml");
		PropertySource<?> propertySource = loadYaml(path);
		environment.getPropertySources().addLast(propertySource);
	}

	private PropertySource<?> loadYaml(Resource path) {
		Assert.isTrue(path.exists(), () -> "Resource " + path + " does not exist");
		try {
			return this.loader.load("custom-resource", path).get(0);
		}
		catch (IOException ex) {
			throw new IllegalStateException("Failed to load yaml configuration from " + path, ex);
		}
	}

}
```

## vault
安装:
```
brew tap hashicorp/tap
brew install hashicorp/tap/vault
```

以开发者非TLS模式启动：
```
brew services start vault
less /opt/homebrew/var/log/vault.log

export VAULT_ADDR="http://localhost:8200"
vault login hvs.dhauihdsiudhaidhaid
```

以开发者TLS模式启动：
```
vault server -dev -dev-root-token-id root -dev-tls
less /opt/homebrew/var/log/vault.log


export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_CACERT='/var/folders/qr/zgztx0sj6n1dxy86sl36ntnw0000gn/T/vault-tls3037226588/vault-ca.pem'
vault login root
```


### 命令行工具：
```
vault version
vault help
vault login [token] | [userID]
vault token --help
vault status [-format=json]
```

通用语法： `vault <command> [options] [path] [args]`

```
vault kv list secret

vault write secret/data/root \
    access_key=dasdijaosdaosdoa \
    secret_key=daoidjaoidapdpapogit \
    region=us-east-1

vault read secret/data/foo

```

### springboot 集成
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
    <version>3.0.0-SNAPSHOT</version>
</dependency>


bootstrap.yml
spring.cloud.vault:
    authentication: TOKEN
    token: 00000000-0000-0000-0000-000000000000


public class CustomizationBean implements VaultConfigurer {

    @Override
    public void addSecretBackends(SecretBackendConfigurer configurer) {

        configurer.add("secret/my-application");

        configurer.registerDefaultKeyValueSecretBackends(false);
        configurer.registerDefaultDiscoveredSecretBackends(true);
    }
}
```